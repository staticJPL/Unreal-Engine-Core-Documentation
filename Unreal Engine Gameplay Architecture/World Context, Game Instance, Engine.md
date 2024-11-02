## World Context

The world context manages World objects, as by comments from the Engine below:

- A context for handling `UWorld` at the engine level. As the engine brings up and destroys worlds, we need a way to track which world belongs to what. WorldContexts can be likened to tracks. By default, we have one track on which we load and unload levels. Adding a second context is like adding another track—a separate progression path for worlds to exist on.
    
- For the `GameEngine`, there will be one `WorldContext` until we decide to support multiple simultaneous worlds. For the `EditorEngine`, there may be one `WorldContext` for the `EditorWorld` and another for the PIE World. `FWorldContext` manages both the current PIE `UWorld` and the state associated with connecting to or traveling between worlds.
    
- `FWorldContext` should remain internal to the UEngine classes. External code should not hold pointers to or attempt to manage `FWorldContexts` directly. External code can still interact with `UWorld*` and pass `UWorlds` into engine-level functions. The engine code can determine the relevant context for a given `UWorld`.
    
- For convenience, `FWorldContext` can maintain external pointers to `UWorlds`. For instance, PIE can link `UWorld* UEditorEngine::PlayWorld` to the PIE world context. If the PIE `UWorld` changes, the `UEditorEngine::PlayWorld` pointer will be automatically updated using `AddRef()` and `SetCurrentWorld()`.
    

In essence, the editor itself represents one world, the game scene represents another, and these worlds are managed by a world context. An example scenario is dragging an actor into the editor, and upon hitting "play in editor" (PIE), the engine generates a new world. The `CurrentWorld` pointer then points to the newly transitioned world, and the World Context retains any necessary saved information for the transition, such as parameters when opening a Level.

![[Unreal Engine Gameplay Architecture/Diagrams/WorldContext.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/c1f8fb4206579ca2a0c0034712b0e01d0a2d5fec/Unreal%20Engine%20Gameplay%20Architecture/Diagrams/WorldContext.svg)
These are the world types defined by the World Context

```cpp
namespace EWorldType
{
	enum Type
	{
		None,		// An untyped world, in most cases this will be the vestigial worlds of streamed in sub-levels
		Game,		// The game world
		Editor,		// A world being edited in the editor
		PIE,		// A Play In Editor world
		Preview,	// A preview world for an editor tool
		Inactive	// An editor world that was loaded but not currently being edited in the level editor
	};
}
```

```cpp
struct FWorldContext
{
   
	TEnumAsByte<EWorldType::Type>	WorldType;

	FSeamlessTravelHandler SeamlessTravelHandler;

	FName ContextHandle;

	/** URL to travel to for pending client connect */
	FString TravelURL;

	/** TravelType for pending client connects */
	uint8 TravelType;

	/** URL the last time we traveled */
	UPROPERTY()
	struct FURL LastURL;

	/** last server we connected to (for "reconnect" command) */
	UPROPERTY()
	struct FURL LastRemoteURL;

}
```

The `TravelURL` and `TravelTypes` handle setup for the target world and any conversion to the next level.

```cpp
// Traveling from server to server.  
UENUM()  
enum ETravelType : int  
{  
    /** Absolute URL. */  
    TRAVEL_Absolute,  
    /** Partial (carry name, reset server). */  
    TRAVEL_Partial,  
    /** Relative URL. */  
    TRAVEL_Relative,  
    TRAVEL_MAX,  
};
```

```cpp
void UEngine::SetClientTravel( UWorld *InWorld, const TCHAR* NextURL, ETravelType InTravelType )
{
	FWorldContext &Context = GetWorldContextFromWorldChecked(InWorld);
	// set TravelURL.  Will be processed safely on the next tick in UGameEngine::Tick().
	Context.TravelURL    = NextURL;
	Context.TravelType   = InTravelType;
    ...
}
```

The engine follows a series of steps when opening a level, beginning with setting the `TravelURL` for the world context. Both the `EditorEngine` and `GameEngine` classes implement the `Tick` function inherited from the base class `UEngine`. Within this function, `TickWorldTravel` is responsible for setting the current world and passing the `WorldContext` for world travel. `TickWorldTravel` manages different types of transitions, such as networked level transitions where the authoritative game mode is established for all clients.
### Persistent Level

The logic handling world transitions isn't defined within the world class because each world typically has only one `PersistentLevel`. The `PersistentLevel` serves as the root level from which sublevels are loaded. If no level is explicitly defined, the `PersistentLevel` defaults to an empty state and acts as a container for other levels that are loaded and unloaded into it.

When the `OpenLevel` function initiates the opening of a `PersistentLevel`, the engine first releases the current world and then creates a new one. This process ensures that any modifications made to the current world before loading a new level are preserved. This behavior is managed through level streaming.

In essence, the `PersistentLevel` remains loaded throughout the session, serving as a stable base while other levels are dynamically loaded and unloaded as needed.

### Level Streaming

```cpp
void UGameplayStatics::LoadStreamLevel(
    const UObject* WorldContextObject, 
    FName LevelName, 
    bool bMakeVisibleAfterLoad, 
    bool bShouldBlockOnLoad, 
    FLatentActionInfo LatentInfo
) {  
    if (UWorld* World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull)) {       
        FLatentActionManager& LatentManager = World->GetLatentActionManager();  
        
        if (LatentManager.FindExistingAction<FStreamLevelAction>(LatentInfo.CallbackTarget, LatentInfo.UUID) == nullptr) {          
            FStreamLevelAction* NewAction = new FStreamLevelAction(
                true, 
                LevelName, 
                bMakeVisibleAfterLoad, 
                bShouldBlockOnLoad, 
                LatentInfo, 
                World
            );  
            LatentManager.AddNewAction(LatentInfo.CallbackTarget, LatentInfo.UUID, NewAction);  
        }    
    }
}
```

The `LoadStreamLevel` function serves as a static method used to load the context object for the current world during level streaming. Handling single world "latent actions" is the responsibility of the `LatentActionManager`. These actions are high-level operations executed once per frame. You can create different latent actions and assign them IDs specific to each world. In the provided code snippet, a latent action is added for level streaming.

Level streaming operations typically incur a high cost due to the processing required for actors, meshes, materials, lights, and other components. To mitigate this, the Level Stream Latent action checks asynchronously once per frame, allowing the loading process to occur in the background and activate seamlessly when ready. This approach ensures smoother gameplay and efficient resource management during level transitions.

There are two asynchronous approaches, each involving the main thread when `OpenLevel` requires access to it:

1. **Recording Information First**:
    - This method begins by recording all necessary information for loading a level before initiating the loading process.
    - It is considered simpler and clearer because it separates the task of recording what needs to be loaded from the actual loading process itself.
2. **Command Mode**:
    - In this approach, commands are pushed into a queue to be executed immediately.
    - It is described as relatively unified because it likely integrates more seamlessly with the engine's overall command processing system, facilitating easier management of complex sequences of operations.

Despite the asynchronous nature of level streaming, certain tasks still require execution on the main thread. For instance, the `OpenLevel` function must create corresponding Actor objects on the main thread. This synchronization is essential because Unreal Engine's architecture mandates that critical actions, such as Actor creation, remain thread-safe and synchronized with the game's main loop.

Handling the passing of data between levels falls under the responsibility of the GameInstance.

With the introduction of UE5 and beyond, traditional level streaming has been replaced by the `WorldPartition` system, which supports spatial compatibility through data layers. However, the core gameplay static functions remain applicable when there is a need to swap levels seamlessly.

## Game Instance

The `GameInstance` represents a single persistent instance of the game application, often likened to a singleton depending on the context.

**Main features include:**

1. **Data Persistence:** It stores game data that needs to persist throughout the application's lifespan, even when transitioning between levels.
    
2. **Communication between Levels:** It facilitates the sharing of information between different levels within the game.
    
3. **Resource Management:** The `GameInstance` manages global resources such as game configurations, audio assets, and other essential data.
    
4. **Initialization:** It handles initialization operations that are crucial to set up the game state properly, ensuring they occur only once at game startup.
    

As a gameplay programmer, decisions about what elements should persist throughout the entire game state, irrespective of game mode or level, should be stored and managed within the `GameInstance`.

Additionally, the `GameInstance` is where the `WorldContext` is stored, providing a central point for managing and accessing information related to the current game world within the application's runtime.
![[Unreal Engine Gameplay Architecture/Diagrams/GameInstance.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/c1f8fb4206579ca2a0c0034712b0e01d0a2d5fec/Unreal%20Engine%20Gameplay%20Architecture/Diagrams/GameInstance.svg)
## UEngine

At the top level `UEngine` is the base for the two main engine types `UGameEngine` & `UEditorEngine`.
![[Unreal Engine Gameplay Architecture/Diagrams/UEngine.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/c1f8fb4206579ca2a0c0034712b0e01d0a2d5fec/Unreal%20Engine%20Gameplay%20Architecture/Diagrams/UEngine.svg)
In the Unreal Engine ecosystem, there's a significant distinction between editor and standalone game environments, influenced by the compilation context. The `UEngine` manages all worlds within a `WorldList`.

- **Standalone Game:** When running as a standalone game, the `UGameEngine` is responsible for creating a single unique `GameWorld`. Due to this singular nature, the `GameInstance` pointer is directly stored for ease of access and management.
    
- **Editor Environment:** In contrast, the `EditorWorld` is primarily used for preview purposes and does not possess an `OwningGameInstance`. Instead, the `OwningGameInstance` indirectly references the `GameInstance` through the `PlayWorld`.

Running multiple worlds simultaneously is possible through unconventional methods but lacks official support, especially concerning network replication. Notably, in the editor environment, each player operates within their own world instance for networking purposes. Similarly, in dedicated server mode, the dedicated server operates within its own distinct world.

`GEngine` serves as the singleton root and global pointer to the `UEngine` instance, providing centralized access and control over engine-level functionalities across different runtime scenarios.

## Gameplay Statics

The `GameplayStatics` class contains a bunch of static functions which grant access to world operations in C++ in the Engine. It's exposed to the blueprint system and is a useful library for managing Engine Level gameplay systems.

```cpp
```text
UCLASS ()
class UGameplayStatics : public UBlueprintFunctionLibrary 
```

## Sub Systems

Unreal Engine features subsystems, each with its own distinct lifecycle, which gameplay programmers may want to integrate custom functionality into. These subsystems remain alive and initialized throughout their lifecycle:

1. **GEngine (Engine Lifetime)** - Singleton
    
    - **Description:** Exists uniquely for the editor or runtime environment, created during engine startup and destroyed upon exit.
    - **Notes:** Provides global access and management of engine-level functionalities.
2. **Editor (Editor Lifetime)** - Singleton
    
    - **Description:** Specific to the editor environment, globally stored and initialized upon editor launch. It is terminated upon exiting the editor.
    - **Notes:** Facilitates editor-specific functionalities and tools.
3. **GameInstance (Game Instance Lifetime)** - Singleton
    
    - **Description:** Created at game start and destroyed at game exit, applicable to both runtime and Play-In-Editor (PIE) modes.
    - **Notes:** Manages game-specific global data and provides a central point for initializing game state.
4. **UWorld (Multiple Lifecycles)** - 1 to Many Relationship
    
    - **Description:** Shares lifecycle with the GameMode and can encompass multiple levels.
    - **Notes:** Represents the current world state during gameplay, influencing level management and transitions.
5. **LocalPlayer (Local Player Lifetime)** - 1 to Many (if using split-screen)
    
    - **Description:** Integral to multiplayer gameplay systems, tied to the lifecycle of UGameInstance.
    - **Notes:** Each instance corresponds to a local player and is accessed in conjunction with their respective PlayerController.

These subsystems provide structured and persistent contexts for managing various aspects of gameplay, from engine-level functionality to multiplayer interactions, ensuring robust control and customization throughout the application's lifecycle.

## Summary

`GEngine` serves as the foundational root akin to a vast tree, representing the universe that encompasses the entire game world. It is crucial in both Editor and Game environments, housing essential components like `WorldContext` and `UGameInstance`.

- **Editor and Game Types:** Both Editor and Game environments are supported, each containing critical elements such as `WorldContext` and `UGameInstance`.
    
- **WorldContext:** This serves as the primary orchestrator for managing world and level operations, as discussed earlier during the level streaming sections.
    
In essence, `GEngine` acts as the central hub from which all game-related functionalities are orchestrated, providing the necessary infrastructure for managing game states, levels, and interactions across both development and gameplay phases.
