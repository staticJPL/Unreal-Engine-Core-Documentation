## World and Levels

At this layer of gameplay architecture, we focus on managing worlds and their levels. In the previous discussion on `WorldContext`, `Game Instance`, & `Engine`, we explored how `WorldContext` facilitates the transition between different worlds across various engine types. From the perspective of a world, its primary responsibility lies in managing levels and their seamless transitions using features like seamless travel.

With the introduction of UE5, the `WorldPartition` system replaces `WorldComposition` as the new method for handling level streaming. This system enhances the management of level streaming by incorporating features such as `DataLayers` within a `WorldPartition`. Additionally, each world object retains crucial information, including its owning game instance, default game mode authority, and game state.

Levels represent the next layer of abstraction where all `AActors` reside, including those organized into `DataLayers` under `WorldPartition`. The life cycle and construction of `Actor` types are explored further in the `AActor & Actor Components` section, delving into the details of how these entities are instantiated and managed.

For gameplay programmers, understanding and constructing this gameplay framework is essential for designing robust gameplay systems in advance, ensuring efficient management of game worlds, levels, and their interactions. This foundational knowledge supports the development of scalable and organized gameplay systems within Unreal Engine environments.

![[Unreal Engine Gameplay Architecture/Diagrams/UWorldLevelUML.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/bbe7fe6b19a0b83b43784e98994ecbc08058ad80/Unreal%20Engine%20Gameplay%20Architecture/Diagrams/UWorldLevelUML.png)

## ULevel

Levels are represented as a class derived from `UObject` within Unreal Engine, encapsulating a collection of Actor objects. Each Level object contains detailed information such as BSP data, a list of brushes. When a level is streamed in, the `OwningWorld` attribute indicates the world to which it belongs.

In upcoming sections, you'll discover that the `WorldPartition` system has succeeded the `WorldComposition` approach. This new system enhances level streaming management by incorporating features such as cached precomputed lighting and navigation information, further optimizing performance and gameplay experiences.

```cpp
class ULevel : public UObject, public IInterface_AssetUserData, public ITextureStreamingContainer  
{  
    GENERATED_BODY()  
  
public:  
  
    /** URL associated with this level. */  
    FURL               URL;  
  
    /** Array of all actors in this level, used by FActorIteratorBase and derived classes */  
    TArray<TObjectPtr<AActor>> Actors;  
  
    /** Array of actors to be exposed to GC in this level. All other actors will be referenced through ULevelActorContainer */  
    TArray<TObjectPtr<AActor>> ActorsForGC;  
  
#if WITH_EDITORONLY_DATA  
    AActor* PlayFromHereActor;  
  
    /** Use external actors, new actor spawned in this level will be external and existing external actors will be loaded on load. */  
    UPROPERTY(EditInstanceOnly, Category=World)  
    bool bUseExternalActors;  
#endif  
  
    /** Set before calling LoadPackage for a streaming level to ensure that OwningWorld is correct on the Level */  
    ENGINE_API static TMap<FName, TWeakObjectPtr<UWorld> > StreamedLevelsOwningWorld;  
       /**   
     * The World that has this level in its Levels array.   
     * This is not the same as GetOuter(), because GetOuter() for a streaming level is a vestigial world that is not used.   
     * It should not be accessed during BeginDestroy(), just like any other UObject references, since GC may occur in any order.  
     */    UPROPERTY(Transient)  
    TObjectPtr<UWorld> OwningWorld;

[....]

private:

// World & Partition Objects
	UPROPERTY()  
	TObjectPtr<AWorldSettings> WorldSettings;  
	  
	UPROPERTY()  
	TObjectPtr<AWorldDataLayers> WorldDataLayers;  
	  
	UPROPERTY()  
	TSoftObjectPtr<UWorldPartitionRuntimeCell> WorldPartitionRuntimeCell;
...
}

```

Certain actors within a level do not require a transform and are consequently invisible. These actors are known as `InfoActor` types, designed to hold critical information that the engine utilizes within the level. Since it's of `AActor` type its information can also be serialized with the level package itself.

Examples of `InfoActors` include:

- **AWorldSettings:** This actor defines various settings that affect the behavior and appearance of the entire world.
- **WorldDataLayers:** These actors are part of the `WorldPartition` system and manage data layers within the world, influencing how different aspects of the level are organized and streamed.

`InfoActors` play a crucial role in providing essential configuration and organizational data within levels, ensuring efficient management and operation of Unreal Engine environments.

```cpp
void ULevel::SortActorList() {  
    QUICK_SCOPE_CYCLE_COUNTER(STAT_Level_SortActorList);  
    
    if (Actors.Num() == 0) {       
        // No need to sort an empty list  
        return;  
    }
    
    LLM_REALLOC_SCOPE(Actors.GetData());  
    UE_MEMSCOPE_PTR(Actors.GetData());  
  
    TArray<AActor*> NewActors;  
    TArray<AActor*> NewNetActors;  
    NewActors.Reserve(Actors.Num());  
    NewNetActors.Reserve(Actors.Num());  
  
    if (WorldSettings) {       
        // The WorldSettings tries to stay at index 0  
        NewActors.Add(WorldSettings);  
  
        if (OwningWorld != nullptr) {          
            OwningWorld->AddNetworkActor(WorldSettings);  
        }    
    }
    
    // Add non-net actors to the NewActors immediately, cache off the net actors to Append after  
    for (AActor* Actor : Actors) {       
        if (IsValid(Actor) && Actor != WorldSettings) {          
            if (IsNetActor(Actor)) {             
                NewNetActors.Add(Actor);  
                if (OwningWorld != nullptr) {                
                    OwningWorld->AddNetworkActor(Actor);  
                }          
            } else {  
                NewActors.Add(Actor);  
            }       
        }    
    }  
    
    NewActors.Append(MoveTemp(NewNetActors));  
  
    // Replace with sorted list.  
    Actors = ObjectPtrWrap(MoveTemp(NewActors));  
}
```

The `WorldSettings` actor is strategically placed at the front of the actor array within a level, followed by non-replicating actors and then replicated actors. This arrangement optimizes the replication process by segregating actors based on their replication needs.

- **Purpose of Ordering:**
    - **WorldSettings:** Positioned at the start of the array as it provides global settings that remain constant and do not require replication.
    - **Non-Replicating Actors:** Placed next in the array, these actors do not need to synchronize their state across networked clients.
    - **Replicated Actors:** Positioned last, these actors have state information that needs to be synchronized across networked clients.

This ordering scheme leverages caching techniques to enhance replication efficiency. For `InfoActors` that do not require replication, they are typically placed in a fixed world coordinate that remains unchanged, simplifying their management within the replication system.

Moreover, if an `InfoActor` is the last element in the array, it minimally impacts the replication system due to its fixed nature and lack of replication needs.

This approach ensures that Unreal Engine optimizes network bandwidth by prioritizing and segregating actors based on their replication requirements and stability within the game world.
#### Level Streaming

With the introduction of UE5, Epic Games has shifted towards the `World Partition System` for dynamically streaming levels into the game world. This modern approach supersedes `World Composition`, which remains available as an optional feature but may eventually be phased out over time. Despite this transition, the core concepts and functionalities persist across both systems.

In UE5, a key component for managing level streaming is the `ULevelStreaming` class. While primarily used within the `WorldPartitionSubsystem` for legacy reasons, such as converting stream levels, it remains capable of loading and managing levels dynamically.

The `WorldPartitionSubsystem` plays a pivotal role in integrating `ULevelStreaming` within the new framework, ensuring compatibility and support for transitioning existing content from older systems to the new `World Partition` approach.

Epic's commitment to improving level management and streaming capabilities within Unreal Engine is aiming for a future of more efficient and scalable solutions .

#### Reference of Levels

`PersistentLevels` in Unreal Engine adhere to a virtual path naming convention (`World/Game/Maps/Level.Sublevel`). This naming convention is crucial when opening or traveling via Seamless Travel within the engine.

However, managing level references based on full path names can be tricky, especially when it comes to version control or making modifications:

- **Handling Duplicates:** If multiple levels share the same name, Unreal Engine loads the first level it finds, which can lead to unintended results.
    
- **Package Considerations:** When using packages, all assets within the package are searched to locate matching levels. This becomes particularly sensitive when transitioning to a `World Partition` system where each actor resides in its own file. During merges or version control operations, it's essential to `CHECK OUT` every actor reference to avoid issues.
    

When dealing with references, especially for levels loaded by name, they may fail to load if they lack direct references in the `Import/Export Table` serialized with the package. To ensure proper loading of levels:

- **Creating References:** You can create indirect references to ensure your level loads correctly:
    - Writing references in a config INI file.
    - Specifying references during the cooking process.
    - Creating a dedicated Reference table.

By establishing an indirect reference table for a map loaded at game startup, you streamline the cooking process. Simply specifying the main map ensures that all associated maps are cooked and included within the package, reducing the risk of loading errors during gameplay.

## UWorld & UWorld Partition

In Unreal Engine, a world consists of multiple levels, and the introduction of the `WorldPartition` system enhances how sub-levels are managed and streamed into a grid-based world system. The `PersistentLevel` remains the root main level from which all other levels are streamed.

- **PersistentLevel:** This serves as the primary root level where core gameplay elements are typically initialized and managed. It anchors the world structure and is essential for seamless integration of sub-levels.
    
- **CurrentLevel:** During runtime, the `CurrentLevel` typically refers to the currently active level within the game. However, it's important to note that for stability and consistency, during runtime operations, this should primarily point to the `PersistentLevel`.
    
- **Actors and Levels:** Actors within Unreal Engine are contained within specific levels. These actors contribute to the overall content and functionality of the world, interacting with each other based on their respective level contexts.
    

The integration of the `WorldPartition` system introduces a more efficient method for managing and streaming levels within Unreal Engine, optimizing performance and scalability while maintaining the foundational role of the `PersistentLevel`.

```cpp
class ENGINE_API UWorld final : public UObject, public FNetworkNotify  
{
	UPROPERTY(Transient)  
	TObjectPtr<class ULevel> PersistentLevel;  
	  
	/** The NAME_GameNetDriver game connection(s) for client/server communication */  
	UPROPERTY(Transient)  
	TObjectPtr<class UNetDriver> NetDriver;  
	  
	/** Instance of this world's game-specific networking management */  
	UPROPERTY(Transient)  
	TObjectPtr<class AGameNetworkManager> NetworkManager;  
	  
	/** Instance of this world's game-specific physics collision handler */  
	UPROPERTY(Transient)  
	TObjectPtr<class UPhysicsCollisionHandler> PhysicsCollisionHandler;
private:  
    /** Level collection. ULevels are referenced by FName (Package name) to avoid serialized references. Also contains offsets in world units */  
    UPROPERTY(Transient)  
    TArray<TObjectPtr<ULevelStreaming>> StreamingLevels
	
	[...]
	// Tick stuff
	TEnumAsByte<ETickingGroup> TickGroup;  
	  
	/** The type of world this is. Describes the context in which it is being used (Editor, Game, Preview etc.) */  
	TEnumAsByte<EWorldType::Type> WorldType;

	FPhysScene*                   PhysicsScene;
	
	TArray<TWeakObjectPtr<class AController> > ControllerList;  
	
	/** List of all the player controllers in the world. */  
	TArray<TWeakObjectPtr<class APlayerController> > PlayerControllerList;
	
	UPROPERTY(Transient)  
	TObjectPtr<class AGameStateBase> GameState;
	
	UPROPERTY(Transient)  
	TObjectPtr<class AGameModeBase> AuthorityGameMode;

	[...]
```

`UWorld` is derived from `UObject`, inheriting features such as reflection, garbage collection, serialization, and construction/post-initialization functionalities. However, from a gameplay systems perspective, several critical systems exist within this hierarchy:

- **World Partition:** Manages levels, level streaming, and data layers, optimizing how content is loaded and unloaded dynamically within the game world.
    
- **Networking:** Includes components like `NetDriver` and `NetworkManager`, facilitating network communication and synchronization across multiplayer environments.
    
- **Physics:** Encompasses the Chaos Scene, Solver, Collisions, and Physics Volumes, governing physical interactions and simulations within the game world.
    
- **Gameplay:** Central components like `GameMode`, `GameState`, `Controller`, `PlayerControllers`, and Gameplay Timers, responsible for defining game rules, managing player interactions, and coordinating gameplay events.
    
- **Ticking:** Organizes entities into groups and types for efficient update processing, ensuring timely execution of game logic and state updates.
    
- **Rendering:** Handles graphics rendering aspects such as `FeatureLevels`, ViewLocation Cache, Canvas operations, Editor Views, and SceneManager, optimizing visual presentation and performance.
    
- **Subsystems:** A collection of specialized systems, including the `WorldPartition`, which enhance specific aspects of world and gameplay management.
    
- **WorldDelegates:** Provides hooks into key events such as Level Change, World Cleanup, World Startup, OnPostDuplicate, and Pre/Post Actor Tick, allowing for custom logic and functionality to be executed at critical points in the game world's lifecycle.
    

The fundamental relationship between World, Level, and Actor will become more clear in later sections. As you go down the abstraction layers you'll feel how these classes collectively manage and govern the behavior, interactions for the entire gameplay architecture.
### World Partition

The World Partition system in Unreal Engine represents a direct evolution from UE4's World Composition, serving as the modern level-based streaming solution. It introduces several key layers of abstraction:

- **Grid:** Defines the spatial layout where levels are organized, allowing for structured management and streaming of game content.
    
- **LevelInstance:** Represents individual instances of levels within the grid, enabling dynamic instantiation and management based on player location and interaction.
    
- **DataLayer:** Provides a mechanism to categorize and manage subsets of game content within levels, facilitating granular control over what is streamed in or out based on gameplay requirements.
    

Within this system, objects are categorized as actors, each capable of defining its own loading rules. At runtime, actors are organized based on their `StreamSource`, which encompasses information from the `Grid`, `LevelInstance`, and `DataLayer`. This information tags actors that need to be dynamically loaded or unloaded during sub-level cell streaming, optimizing performance and memory usage.

#### One File Per Actor(OFPA)

Using `One File Per Actor` (OFPA) in Unreal Engine allows each object to be saved into a separate file, providing a convenient approach for collaborative level development. This system also integrates seamlessly with Unreal's built-in version management system, enhancing project organization and collaboration workflows.

When OFPA is enabled, all external actors are automatically saved to a dedicated folder named `_ExternalActors_`. This feature ensures that each actor's data is neatly organized and can be managed individually.

However, it's important to note that enabling OFPA may require conversion for existing sublevels to reload them properly. Unreal Engine offers a `CommandLet` for these conversions, which helps streamline the process and ensures compatibility with the new file structure.

For more detailed information on using OFPA and performing conversions, you can refer to Unreal Engine's official documentation on [One File Per Actor](https://dev.epicgames.com/documentation/en-us/unreal-engine/one-file-per-actor-in-unreal-engine).

This approach not only improves workflow efficiency but also supports collaborative efforts by simplifying asset management and version control within Unreal Engine projects.

#### Partitioning of Actors in a WorldPartition

The `WorldPartition` system organizes `AActors` using an elegant runtime hashed grid system. This system effectively partitions levels into cells for efficient spatial level streaming.

![[Unreal Engine Gameplay Architecture/Diagrams/WorldPartitionGridCell.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/bbe7fe6b19a0b83b43784e98994ecbc08058ad80/Unreal%20Engine%20Gameplay%20Architecture/Diagrams/WorldPartitionGridCell.png)

For Editor Only modes actors need to be organized into sub levels for cooking.
```cpp
// WorldPartitionRuntimeSpatialHash.cpp
for (int32 GridIndex = 0; GridIndex < AllGrids.Num(); GridIndex++)
{
	// Partitions Actors into respective cells.
    const FSpatialHashRuntimeGrid& Grid = AllGrids[GridIndex];
    const FSquare2DGridHelper PartionedActors = GetPartitionedActors(WorldBounds, Grid, GridActorSetInstances[GridIndex]);
	// CreateStreamingGrid Creates the Sublevel
    if (!CreateStreamingGrid(Grid, PartionedActors, StreamingPolicy, OutPackagesToGenerate))
    {
        return false;
    }
}
```

This functionality is facilitated through the interaction between the `Grid`, `Level`, and `Cell`, which is demonstrated in the `FSquare2DGridHelper` function.

```cpp
FSquare2DGridHelper::FSquare2DGridHelper(const FBox& InWorldBounds, const FVector& InOrigin, int64 InCellSize)
    : WorldBounds(InWorldBounds)
    , Origin(InOrigin)
    , CellSize(InCellSize)
{
    // Compute Grid's size and level count based on World bounds
    int64 GridSize = 1;
    int32 GridLevelCount = 1;

    if (WorldBounds.IsValid)
    {
        const FVector2D DistMin = FVector2D(WorldBounds.Min - Origin).GetAbs();
        const FVector2D DistMax = FVector2D(WorldBounds.Max - Origin).GetAbs();
        const double WorldBoundsMaxExtent = FMath::Max(DistMin.GetMax(), DistMax.GetMax());

        if (WorldBoundsMaxExtent > 0)
        {
            GridSize = 2 * FMath::CeilToDouble(WorldBoundsMaxExtent / CellSize); 
            if (!FMath::IsPowerOfTwo(GridSize))
            {
                GridSize = FMath::Pow(2, FMath::CeilToDouble(FMath::Log2(static_cast<double>(GridSize))));
            }
            GridLevelCount = FMath::FloorLog2_64(GridSize) + 1;
        }
    }

    check(FMath::IsPowerOfTwo(GridSize));

    Levels.Reserve(GridLevelCount);
    int64 CurrentCellSize = CellSize;
    int64 CurrentGridSize = GridSize;
    for (int32 Level = 0; Level < GridLevelCount; ++Level)
    {
        int64 LevelGridSize = CurrentGridSize;

        if (!GRuntimeSpatialHashUseAlignedGridLevelsEffective)
        {
            // Except for top level, adding 1 to CurrentGridSize (which is always a power of 2) breaks the pattern of perfectly aligned cell edges between grid level cells.
            // This will prevent weird artefact during actor promotion when an actor is placed using its bounds and which overlaps multiple cells.
            // In this situation, the algorithm will try to find a cell that encapsulates completely the actor's bounds by searching in the upper levels, until it finds one.
            // Also note that, the default origin of each level will always be centered at the middle of the bounds of (level's cellsize * level's grid size).
            LevelGridSize = (Level == GridLevelCount - 1) ? CurrentGridSize : CurrentGridSize + 1;
        }

        Levels.Emplace(FVector2D(InOrigin), CurrentCellSize, LevelGridSize, Level);

        CurrentCellSize <<= 1;
        CurrentGridSize >>= 1;
    }

    // Make sure the always loaded cell exists
    GetAlwaysLoadedCell();
}
```

1. Centered around `InOrigin`, divide the world into 2^n cells based on `InCellSize`.
2. Initialize n+1 levels.

So, if we have a square world of [-2000, 2000] and a cell size of 1600:

- Divide the world into a 4x4 grid of cells, each cell being 1600.
- Initialize 3 `FGridLevels`
	- Level 0 CellSize[1600] , GridSize[4]
	- Level 1 CellSize [3200], GridSize[2]
	- Level 3 CellSize[6400], GridSize[1]

Looks something like this.

![[Unreal Engine Gameplay Architecture/Diagrams/WorldPartitionGridExample.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/bbe7fe6b19a0b83b43784e98994ecbc08058ad80/Unreal%20Engine%20Gameplay%20Architecture/Diagrams/WorldPartitionGridExample.png)

For a final level it's always considered an "AlwaysLoaded" situation

```cpp
// Returns the always loaded (top level) cell  
inline FGridLevel::FGridCell& GetAlwaysLoadedCell() { return Levels.Last().GetCell(FGridCellCoord2(0,0)); }
```

Further more Epic describes in a comment about an `AlwaysLoaded` Cell

```cpp
// In PIE, Always loaded cell is not generated. Instead, always loaded actors will be added to AlwaysLoadedActorsForPIE.
// This will trigger loading/registration of these actors in the PersistentLevel (if not already loaded).
// Then, duplication of world for PIE will duplicate only these actors.
// When stopping PIE, WorldPartition will release these FWorldPartitionReferences which 
// will unload actors that were not already loaded in the non PIE world.
bool UWorldPartitionRuntimeHash::ConditionalRegisterAlwaysLoadedActorsForPIE(...)
{
   (...)
}

```

The `AlwaysLoadedCell` corresponds to the `PersistentLevel` within the WorldPartition system.

After actors are distributed into their respective cells, all the data is processed and stored within `UWorldPartitionRuntimeSpatialHash`. This organization is illustrated in the diagram presented at the beginning of this section.

**Cell Coordinates**

The `FGridLevel` uses hash storage.

```cpp
//FGridCell
struct FGridLevel : public FGrid2D
{
    // Variables from FGrid2D
    int64 CellSize;
    int64 GridSize;
    // End of FGrid2D
    
    int32 Level; // Grid level
    TArray<FGridCell> Cells; // Hashed storage of cells. Cells without data (Actor, LOD, etc.) may not be stored.
    TMap<int64, int64> CellsMapping; // Key: Mapping of cell's 2D coordinates in the grid, Value: Index in Cells hash table
};
```

The Cell Index can be obtain given the cell coordinates.

```cpp
/**  
 * Returns the cell index of the provided coords * * @return true if the coords was inside the grid */
inline bool GetCellIndex(const FGridCellCoord2& InCoords, uint64& OutIndex) const  
{  
    if (IsValidCoords(InCoords))  
    {       OutIndex = (InCoords.Y * GridSize) + InCoords.X;  
       return true;  
    }    return false;  
}
```

```cpp
struct FGridCell
{
    FGridCell(const FGridCellCoord& InCoords)
        : Coords(InCoords)
    {}

#if WITH_EDITOR
    // Editor-specific functionality
#endif

    FGridCellCoord GetCoords() const
    {
        return Coords;
    }

private:
    FGridCellCoord Coords;

#if WITH_EDITOR
    TSet<FGridCellDataChunk> DataChunks;
#endif
};
```

`FGridCells` stores the coordinates for a Grid and gets converted into global coordinates.

```cpp
inline bool GetCellGlobalCoords(const FGridCellCoord& InCoords, FGridCellCoord& OutGlobalCoords) const
{
    // Check if the Z index of InCoords is valid within the Levels array
    if (Levels.IsValidIndex(InCoords.Z))
    {
        // Access the GridLevel corresponding to InCoords.Z
        const FGridLevel& GridLevel = Levels[InCoords.Z];
        
        // Check if the coordinates (X, Y) are valid within the GridLevel
        if (GridLevel.IsValidCoords(FGridCellCoord2(InCoords.X, InCoords.Y)))
        {
            // Calculate the coordinate offset
            int64 CoordOffset = Levels[InCoords.Z].GridSize >> 1;
            
            // Adjust OutGlobalCoords based on CoordOffset
            OutGlobalCoords = InCoords;
            OutGlobalCoords.X -= CoordOffset;
            OutGlobalCoords.Y -= CoordOffset;
            
            // Return true indicating successful conversion
            return true;
        }
    }
    
    // Return false if either the Z index is invalid or the coordinates are invalid
    return false;
}
```

When creating a new cell object in C++ using `NewObject` you need to assign it a name.

```cpp
NewObject<UWorldPartitionRuntimeSpatialHashCell>(this, StreamingPolicy->GetRuntimeCellClass(), FName(CellName))
```

The partition logic is handed inside the `FSquare2DGridHelper GetPartitionedActors(..)` a function call inside the for loop below

```cpp
for (const IStreamingGenerationContext::FActorSetInstance* ActorSetInstance : ActorSetInstances)
{
    check(ActorSetInstance->ActorSet->Actors.Num() > 0);

    FSquare2DGridHelper::FGridLevel::FGridCell* GridCell = nullptr;

    if (ActorSetInstance->bIsSpatiallyLoaded)
    {
        const FBox2D ActorSetInstanceBounds(FVector2D(ActorSetInstance->Bounds.Min), FVector2D(ActorSetInstance->Bounds.Max));
        int32 LocationPlacementGridLevel = 0;
        if (ShouldActorUseLocationPlacement(ActorSetInstance, ActorSetInstanceBounds, LocationPlacementGridLevel))
        {
            // Find grid level cell that contains the actor cluster pivot and put actors in it.
            FGridCellCoord2 CellCoords;
            if (PartitionedActors.Levels[LocationPlacementGridLevel].GetCellCoords(ActorSetInstanceBounds.GetCenter(), CellCoords))
            {
                GridCell = &PartitionedActors.Levels[LocationPlacementGridLevel].GetCell(CellCoords);
            }
        }
        else
        {
            // Find grid level cell that encompasses the actor cluster bounding box and put actors in it.
            const FVector2D ClusterSize = ActorSetInstanceBounds.GetSize();
            const double MinRequiredCellExtent = FMath::Max(ClusterSize.X, ClusterSize.Y);
            const int32 FirstPotentialGridLevel = FMath::Max(FMath::CeilToDouble(FMath::Log2(MinRequiredCellExtent / (double)PartitionedActors.CellSize)), 0);

            for (int32 GridLevelIndex = FirstPotentialGridLevel; GridLevelIndex < PartitionedActors.Levels.Num(); GridLevelIndex++)
            {
                FSquare2DGridHelper::FGridLevel& GridLevel = PartitionedActors.Levels[GridLevelIndex];

                if (GridLevel.GetNumIntersectingCells(ActorSetInstance->Bounds) == 1)
                {
                    GridLevel.ForEachIntersectingCells(ActorSetInstance->Bounds, [&GridLevel, &GridCell](const FGridCellCoord2& Coords)
                    {
                        check(!GridCell);
                        GridCell = &GridLevel.GetCell(Coords);
                    });

                    break;
                }
            }
        }
    }
    
    if (!GridCell)
    {
        GridCell = &PartitionedActors.GetAlwaysLoadedCell();
    }

    GridCell->AddActorSetInstance(ActorSetInstance);
}

return PartitionedActors;
```

In a 4x4 cell grid system, the origin point for loaded actors and the concept of common boundaries are critical considerations:

1. **Grid Layout:** With a 4x4 grid, you have a total of 16 cells arranged in a matrix format.
    
2. **Origin Point:** The origin point typically refers to the center of the grid. In a 4x4 grid, the center would be a 2x2 cell located at the origin coordinates.
    
3. **AlwaysLoaded Cell:** This cell corresponds to the `PersistentLevel`, which is always loaded and serves as the root of the world.
    
4. **Actor Placement:** If you place an actor, such as a cube, at the origin (center boundary) of this grid:
    
    - The cube will be positioned in the cell that represents the `PersistentLevel`.
    - This means the cube will always be loaded and present in the game world, as it resides in the `PersistentLevel` cell which remains active throughout gameplay.
5. **Impact of Grid Configuration:** The power of 2 division of cells (in this case, 4x4) ensures systematic organization of game content. Actors placed within common boundaries, such as the origin cell, benefit from being part of the `PersistentLevel`, ensuring their consistent presence and functionality in the game environment.
    

In summary, placing an actor at the origin point of a 4x4 grid ensures it remains loaded and active due to its association with the `PersistentLevel` cell, which forms the central and always loaded part of the `WorldPartition` system.
#### Data Layers

Data Layers in Unreal Engine serve as an asset tagging system for actors within a level. They enable designers to apply customized loading logic for different runtime or editor-based scenarios. For instance, they can control the spawning of specific actor types based on various game mechanics, such as special events (like holidays).

The data layer system consists of three main components:

1. **Data Layers:** These are the primary containers where actors are categorized based on specific criteria or requirements. Each data layer represents a distinct set of conditions or rules that govern how actors are managed or loaded within the level.
    
2. **Layer Policies:** These define the loading behavior and rules associated with each data layer. They specify conditions under which actors within a data layer should be loaded, unloaded, or managed during gameplay.
    
3. **Layer Data:** This includes any additional metadata or configuration settings tied to each data layer. It may encompass details such as actor visibility settings, streaming priorities, or any custom parameters relevant to the layer's function.
    

Together, these components empower level designers to efficiently manage and control the presence and behavior of actors within a level, enhancing flexibility and customization based on dynamic gameplay needs or specific development scenarios.

**Data Layer Sub System**

```cpp
// DataLayerSubsystem.h
class ENGINE_API UDataLayerSubsystem : public UWorldSubsystem
{
public:
    // Get the state of a specific DataLayer
    UFUNCTION(BlueprintCallable, Category = "DataLayers")
    UDataLayerInstance* GetDataLayerInstanceFromAsset(const UDataLayerAsset* InDataLayerAsset) const;

    // Get the runtime state of a DataLayer instance
    UFUNCTION(BlueprintCallable, Category = "DataLayers")
    EDataLayerRuntimeState GetDataLayerInstanceRuntimeState(const UDataLayerAsset* InDataLayerAsset) const;

    // Get the effective runtime state of a specific DataLayer
    UFUNCTION(BlueprintCallable, Category = "DataLayers")
    EDataLayerRuntimeState GetDataLayerInstanceEffectiveRuntimeState(const UDataLayerAsset* InDataLayerAsset) const;
};
```

**Data Layer Asset**

```cpp
// DataLayerAsset.h
class ENGINE_API UDataLayerAsset : public UObject
{
public:
    // Type of data layer, which has two categories:
    // Editor: Effective in the Editor but DataLayerSubsystem only provides runtime interfaces,
    //         so this type seems to have little utility.
    // Runtime: Effective in both Editor and Runtime.
    EDataLayerType DataLayerType;
};
```

**World DataLayer**

```cpp
class ENGINE_API AWorldDataLayers : public AInfo
{
public:
    // Holds the names of DataLayerInstance that need to be loaded or activated.
    // These can be replicated to the client through relevant replication interfaces.
    UPROPERTY(Transient, Replicated, ReplicatedUsing=OnRep_ActiveDataLayerNames)
    TArray<FName> RepActiveDataLayerNames;

    UPROPERTY(Transient, Replicated, ReplicatedUsing=OnRep_LoadedDataLayerNames)
    TArray<FName> RepLoadedDataLayerNames;
};
```

The world data layer is added with the world partition and is saved with serialization. This holds the state of the data layers.

```cpp
DataLayerInstanceType* AWorldDataLayers::CreateDataLayer(CreationsArgs... InCreationArgs)
{
    DataLayerInstanceType* NewDataLayer = NewObject<DataLayerInstanceType>(...);
    DataLayerInstances.Add(NewDataLayer);
    return NewDataLayer;
}
```

For the streaming grid when the actors are divided into cells all the actors will be added to a `DataChunk` of `FGridCell`

```cpp
#if WITH_EDITOR

void AddActorSetInstance(const IStreamingGenerationContext::FActorSetInstance* ActorSetInstance)
{
    const FDataLayersID DataLayersID = FDataLayersID(ActorSetInstance->DataLayers);
    FGridCellDataChunk& ActorDataChunk = DataChunks.FindOrAddByHash(DataLayersID.GetHash(), FGridCellDataChunk(ActorSetInstance->DataLayers, ActorSetInstance->ContentBundleID));
    ActorDataChunk.AddActorSetInstance(ActorSetInstance);
}

const TSet<FGridCellDataChunk>& GetDataChunks() const
{
    return DataChunks;
}

const FGridCellDataChunk* GetNoDataLayersDataChunk() const
{
    for (const FGridCellDataChunk& DataChunk : DataChunks)
    {
        if (!DataChunk.HasDataLayers())
        {
            return &DataChunk;
        }
    }
    return nullptr;
}

#endif // WITH_EDITOR

```

If an Actor on a Cell has n different `DataLayers` assigned to it, then the cells have n chunks.

Creating the `StreamingGrid` is creating our sublevel 

```cpp
bool UWorldPartitionRuntimeSpatialHash::CreateStreamingGrid(...)
{
    // For each level of the grid
    for (const FSquare2DGridHelper::FGridLevel& TempLevel : PartionedActors.Levels)
    {
        // Create a streaming cell based on DataChunks in each cell.
        const FSquare2DGridHelper::FGridLevel::FGridCell& TempCell = TempLevel.Cells[TempCellMapping.Value];
        for (const FSquare2DGridHelper::FGridLevel::FGridCellDataChunk& GridCellDataChunk : TempCell.GetDataChunks())
        {
            FString CellName = GetCellNameString(...);
            UWorldPartitionRuntimeSpatialHashCell* StreamingCell = NewObject<UWorldPartitionRuntimeSpatialHashCell>(...);
            // Further processing or initialization of StreamingCell can be added here
        }
    }

    // Return true or false based on success or failure
    return true; // Placeholder return value, adjust as per actual implementation
}
```

The unit for a `StreamCell` is cooked into assets. 

![[Unreal Engine Gameplay Architecture/Diagrams/StreamCellAssets.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/bbe7fe6b19a0b83b43784e98994ecbc08058ad80/Unreal%20Engine%20Gameplay%20Architecture/Diagrams/StreamCellAssets.png)

The subscript (-1,2) denotes that an actor has two different `DataLayers`.
When dealing with `DataLayers` `YOU MUST` be careful about performance costs that come from streaming in actors, especially when you have a lot of `DataLayers` in the world.

Furthermore you can interface with the `DataLayer Subsystem` to set the state of a `DataLayer`.

```cpp
void AWorldDataLayers::SetDataLayerRuntimeState(...)
{
    // Ensure it can only be called on the server: GetLocalRole() == ROLE_Authority
    // Update the saved DataLayerInstance state
    // Handle recursive DataLayers
}
```

Gets the currently visible cells this frame

```cpp
void FSpatialHashStreamingGrid::GetCells(...) const
{
    // For each cell coordinate, get all cells, i.e.,
    for (const UWorldPartitionRuntimeCell* Cell : LayerCell->GridCells)
    {
        // If this cell is not a DataLayer cell or its DataLayers are visible
        if (!Cell->HasDataLayers() || (DataLayerSubsystem && DataLayerSubsystem->IsAnyDataLayerInEffectiveRuntimeState(Cell->GetDataLayers(), EDataLayerRuntimeState::Activated)))
        {
            // Add to the load or visibility list
        }
    }
}

```

#### Level Instance

Level instancing in Unreal Engine enables designers to edit the content of a template within a level or utilize it for level streaming. It allows for dynamic updates to content that can be streamed in or out based on gameplay needs. However, it's important to note that level streaming can be used independently of World Partition.

According to Epic's Documentation, level instances and packed level blueprints support `DataLayers`. Actors within a level instance inherit the data layer assigned to its corresponding level instance actor by default. Additionally, actors within the level instance can also support additional data layers for further customization.

There are two main types of level streaming modes:

1. **Embedded Mode:** This mode applies to level instances that use the `One File Per Actor (OFPA)` system. In embedded mode, these level instances are discarded, and their actors are integrated into the World Partition at runtime. It's crucial to note that non-OFPA actors, such as `AWorldSettings`, do not persist at runtime in embedded mode. Designers must avoid relying on these actors or switch to level streaming mode as necessary.
    
2. **Level Streaming Mode:** Level instances that do not use OFPA cannot be embedded in the World Partition grid. Instead, they utilize standard level streaming at runtime. In this mode, when the level instance actor is loaded through its owning World Partition runtime cell, the associated level is loaded accordingly.
    

In essence, a level instance can be likened to an actor prefab, providing a streamlined method for designing and managing reusable content within Unreal Engine levels.

### Summary

`ULevel` serves as the container for actors and collaborates with the World Partition runtime grid, which divides the world into cells to enable `LevelStreaming` features. From the perspective of a game programmer, the `GameWorld` manages all players and enforces rules across all levels.

When designing low-level gameplay systems, developers may consider creating a `UWorldSubsystem` to enhance the functionality of the world. Alternatively, an `InfoActor` type can be placed within a level to expose specific level information to engine subsystems. Depending on the demands of the game design, most gameplay mechanics can typically be implemented using the `GameMode` and `GameState`. However, there's value in extending or adding to existing engine systems to tailor functionality to specific game requirements.
