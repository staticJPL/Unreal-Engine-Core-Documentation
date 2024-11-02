## Game Mode & Game States

The article on `UWorld` and `ULevel` delves into the foundational layers that govern how `AActors` operate within Unreal Engine during runtime. In the context of Unreal Engine's architecture, the concept of the game mode is pivotal as it establishes the rules and flow for players within a level.

Originally, Unreal Engine was predominantly oriented towards the FPS genre, where games typically comprised a single world with multiple levels. In such games, a significant portion—often around 90%—of the gameplay logic was shared across these levels, tailored to the specifics of the genre.

As the engine evolved to support larger worlds and diverse game genres with varying gameplay systems, the separation of gameplay logic from representation became essential. From a technical perspective, this division of game resources is intricately tied to managing `AActors` throughout the runtime game loop.

Ultimately, the world in Unreal Engine serves as the primary framework for gameplay—it defines the operational mode of the game world, hence the term "GameMode." This overarching structure ensures consistency and coherence in how gameplay mechanics are implemented and executed across different levels and game scenarios.
### AGameModeBase

![[Unreal Engine Gameplay Architecture/Diagrams/UGameMode.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/737af604e3738c054be49096ce418a3f5719faa3/Unreal%20Engine%20Gameplay%20Architecture/Diagrams/UGameMode.svg)

In the early days of Unreal Engine, the engine was primarily designed around the FPS genre. Initially, `AGameMode` served as the default base class for managing game rules within a world. As Unreal Engine evolved, the functionality of managing game modes was refactored into `AGameModeBase`, which now serves as the foundational class encompassing essential type information for defining game modes.

`AGameModeBase` is responsible for several core functionalities within Unreal Engine:

1. **Spawning Game Entities**: It handles the instantiation of essential gameplay entities such as default pawns, player controllers, game states, and player states using UClass types created via reflection.

2. **Player Processes**: Manages player-related processes including player login, determining player start locations, and managing player controllers.

3. **Level Management**: Facilitates level management tasks such as seamless travel between levels, which involves transitioning actors from one level to another.

4. **Game Flow Control**: Controls the overall flow of the game, including starting, pausing, restarting, and handling game mode event delegates.

5. **Multiplayer Support**: For multiplayer games, `AGameModeBase` handles connecting players and manages the match state, ensuring consistency across all connected clients.

The relationship between `GameMode` and `World` in Unreal Engine is tightly coupled. This tight coupling means that swapping out a game mode at runtime is not straightforward. When loading into a new level, Unreal Engine creates a new world instance specific to that level. During transitions such as level streaming or loading sublevels, fixing up game entity references is not directly supported due to this coupling.

Unreal Engine manages two primary transition scenarios to optimize memory and performance:

- **Non-seamless Travel**: In scenarios like server travel or client travel where seamless travel is not used, Unreal Engine releases the current world and creates a new instance based on the specified `GameModeClass`.

- **Seamless Travel**: When using seamless travel, the game mode calls `GetSeamlessTravelActorList`, which helps in identifying and transferring specific actors between levels to maintain continuity and game state integrity across transitions.

This architecture ensures that Unreal Engine can efficiently manage large-scale worlds and diverse gameplay scenarios while maintaining performance and consistency in multiplayer environments.

```cpp
void AGameModeBase::GetSeamlessTravelActorList(bool bToTransition, TArray<AActor*>& ActorList)  
{  
    // Get allocations for the elements we're going to add handled in one go  
    const int32 ActorsToAddCount = GameState->PlayerArray.Num() + (bToTransition ? 3 : 0);  
    ActorList.Reserve(ActorsToAddCount);  
  
    // Always keep PlayerStates, so that after we restart we can keep players on the same team, etc  
    ActorList.Append(GameState->PlayerArray);  
  
    if (bToTransition)  
    {       // Keep ourselves until we transition to the final destination  
       ActorList.Add(this);  
       // Keep general game state until we transition to the final destination  
       ActorList.Add(GameState);  
       // Keep the game session state until we transition to the final destination  
       ActorList.Add(GameSession);  
  
       // If adding in this section best to increase the literal above for the ActorsToAddCount  
    }  
}
```

The purpose of this code snippet is to manage seamless travel between different levels in Unreal Engine using the `FSeamlessTravelHandler` class. Here’s an explanation based on the provided description:

1. **FSeamlessTravelHandler Class**: This class is responsible for handling the seamless travel process. It manages the asynchronous loading of a new world (level) and ensures that the current world state and its actors are transferred seamlessly to the new level.
    
2. **Seamless Travel Process**: When seamless travel is initiated:
    
    - The current world (`CurrentWorld`) is prepared for transition.
    - A new world is loaded asynchronously from a package. This ensures that the new level is loaded without blocking the main game thread, which is crucial for maintaining smooth gameplay and responsiveness.
    - Once the new world’s `UObject` is instantiated and ready, the `SeamlessTravelLoadCallback` function is invoked. This callback likely handles the finalization steps after the new world is fully loaded and initialized.
3. **SetHandlerLoadedData**: After the new world’s `UObject` is created and prepared, it is passed into `SetHandlerLoadedData`. This function likely sets the `LoadedWorld` variable within `FSeamlessTravelHandler` to hold a reference to the newly loaded world. This reference is essential for managing and accessing the state of the new level during seamless travel.
    
4. **Garbage Collection Management**: To ensure proper memory management and avoid memory leaks, the newly loaded world (represented by its `UObject`) is added to the root set for garbage collection (`GC`). This ensures that Unreal Engine’s garbage collector can properly manage the lifecycle of objects and release memory when it’s no longer needed.
    

Overall, the `FSeamlessTravelHandler` class encapsulates the entire process of transitioning between levels seamlessly in Unreal Engine, ensuring that game state and actor data are preserved and transferred correctly during level changes.

```cpp

/** callback sent to async loading code to inform us when the level package is complete */
void FSeamlessTravelHandler::SeamlessTravelLoadCallback(const FName& PackageName, UPackage* LevelPackage, EAsyncLoadingResult::Type Result)
{
    // make sure we remove the name, even if travel was canceled.
    const FName URLMapFName = FName(*PendingTravelURL.Map);
    UWorld::WorldTypePreLoadMap.Remove(URLMapFName);

    // defer until tick when it's safe to perform the transition
    if (IsInTransition())
    {
	    // Loaded World Here for transition
        UWorld* World = UWorld::FindWorldInPackage(LevelPackage);

        // If the world could not be found, follow a redirector if there is one.
        if (!World)
        {
            World = UWorld::FollowWorldRedirectorInPackage(LevelPackage);
            if (World)
            {
                LevelPackage = World->GetOutermost();
            }
        }
        SetHandlerLoadedData(LevelPackage, World);
    }

	[...]
}

```

After the transition world is ready and loaded, the `GameMode` along with all other game entities (GameState, GameSession) will be migrated to the `LoadedWorld`. One thing to note is that if no `TransitionMap` is specified, UE will create a dummy world to load into. During the final migration step labeled "Still in transition," `GetSeamlessTravelActorList` is called a second time. This skips `ActorList.Add` for `GameState` and `GameMode`, so on the next tick, `InitWorld` is called with nothing assigned. Unreal then looks for the game entity `UClass` types and reinstates them. Therefore, be cautious when traveling because the state saved in `GameMode` may be lost, but the `Pawn` and `Controller` remain consistent by default.

### Game Mode Game Logic

As a gameplay programmer, the question often arises about which gameplay logic should be implemented and how various systems integrate with the game mode class. The primary role of the game mode is to implement gameplay logic, such as defining winning conditions and setting initial gameplay states.

From the game mode's perspective, gameplay actors like the controller and pawn share the same logic that is defined within the game mode itself.

Regarding networking, the game mode exists solely on the server to prevent cheating issues that would arise if it were accessible to clients.

In comparison, the player controller focuses exclusively on player behavior, making it clear where logic needs to be written for player-related actions versus game-related actions.

Lastly, the game instance manages and oversees coordination between game modes at a higher level. Therefore, any interactions between game modes can be managed at the game instance level, while the game mode class remains responsible for gameplay-specific logic.

### Game State

The `GameMode` and `GameState` are coupled together in Unreal Engine. In networking scenarios, clients are restricted from accessing the `GameMode` object, which exclusively resides on the server. Players require information about the current game state, and in a single-player session, where only one player state exists, the game state might not be crucial. However, in multiplayer setups, the `GameMode` is replicated to every connected client. This replication is facilitated by the `GameState`, which maintains references to all player states replicated from the world. For example, if player1 needs to retrieve information about the current state of player2, player1 would obtain this information from the `GameState`.

![[Unreal Engine Gameplay Architecture/Diagrams/UGameState.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/737af604e3738c054be49096ce418a3f5719faa3/Unreal%20Engine%20Gameplay%20Architecture/Diagrams/UGameState.svg)

The UML diagram of `AGameStateBase` reflects the FPS origins of the engine. Developers are encouraged to customize `GameState` according to their specific needs, which explains why certain legacy code remains in place. For multiplayer programmers, determining which data should be accessible to clients involves a decision-making process: if access is restricted, the logic is implemented within `GameMode`; otherwise, `GameState` is the object where you would implement the logic you intend clients to access.
### Game Mode Flow

![[GameEngineFlow.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/737af604e3738c054be49096ce418a3f5719faa3/Unreal%20Engine%20Gameplay%20Architecture/Diagrams/GameEngineFlow.png)

### Summary

At this level of abstraction, the relationship between the gameplay systems in Unreal Engine should now be clearer. Unreal employs `AInfo` actors to embed functionality into a level, as demonstrated by `AGameMode`, `WorldSettings`, and `GameState`. This approach aims to decouple gameplay logic from lower-level world mechanics while ensuring synchronization of network states across clients. These actors establish rules and relationships for the pawns and controllers that populate the world. The next significant topic to explore is how to construct these gameplay systems starting from a base `AActor`.
