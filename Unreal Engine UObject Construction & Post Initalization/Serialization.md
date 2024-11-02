### Serialization

During my exploration of Unreal Engine's reflection system, I encountered the intricacies of its serialization system, which initially posed challenges to understand. To prioritize clarity, I deferred deep exploration until the need arose.

Understanding Unreal's serialization system is crucial for comprehending various aspects of the construction and post-initialization processes. While navigating the code can initially be daunting, I found valuable resources such as articles and talks from Unreal Fest 2023. These resources provided insights that I've distilled into this section, aiming to consolidate key concepts.

For a deeper analysis, I recommend referring to the original sources cited at the end of this section.

#### What is Serialization?

The technical computer science term refers to a data processing technique used to store an object's state into a data structure. This structure can then be converted into a format accessible in various forms.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/SerializationProcess.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/71e18ed9b3cfd2199c045302bc128c4bf673b4a8/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/SerializationProcess.svg)

`Serialization` involves converting an object into a stream of bytes to store or transfer it into memory, a database, file, or any desired format. By encoding objects into byte streams, their states can be preserved. Deserialization, on the other hand, is the process of reading serialized data and reconstructing the object within a specific computer environment.
#### Unreal Serialization 

According to `Mouhuak` and `Stones` blog posts, the Unreal Engine serialization system utilizes the `Visitor Pattern`. In this pattern, the `FArchive` class acts as the visitor, abstracting the serialized archive interface. Every `UObject`, including `FProperty` (non-UObject), implements the `void Serialize(FArchive& Ar)` virtual function. `FArchive` is particularly powerful because it facilitates serialization and deserialization of assets to disk, as well as handling memory and UObject operations.

```cpp
// APlayer Example

// UProperty tagged variables
UPROPERTY()
int Health;

UPROPERTY()
float Stamina;

// C++ ordinary variables
int32 UserID;
uint AmmoIndex; 
```

According to `Stones` article, `UClass` encapsulates properties tagged with `UPROPERTY`, enabling extraction of stored type information from memory and serialization of all required data into a desired format. As of UE 4.25, there are two serialization methods available:

- TaggedPropertySerializer (TPS)
- UnversionedPropertySerializer (UPS)
##### TaggedPropertySerializer

TPS manages the serialization tasks for saving or loading assets in Editor mode. Once TPS completes its operations, UPS takes over to handle tasks related to saving and loading cooked build assets.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/TPSPT1svg.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/71e18ed9b3cfd2199c045302bc128c4bf673b4a8/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/TPSPT1svg.svg)

`FPropertyTag` is the structure that aids in the serialization of different types. The structure contains several important fields. The code layout looks like this:

```cpp
/**  
 *  A tag describing a class property, to aid in serialization. */struct FPropertyTag  
{  
    // Transient.  
    FProperty* Prop = nullptr;  
  
    // Variables.  
    FName  Type;     // Type of property  
    uint8  BoolVal = 0;// a boolean property's value (never need to serialize data for bool properties except here)  
    FName  Name;     // Name of property.  
    FName  StructName;    // Struct name if FStructProperty.  
    FName  EnumName;  // Enum name if FByteProperty or FEnumProperty  
    FName  InnerType; // Inner type if FArrayProperty, FSetProperty, or FMapProperty  
    FName  ValueType; // Value type if UMapPropery  
    int32  Size = 0;   // Property size.  
    int32  ArrayIndex = INDEX_NONE; // Index if an array; else 0.  
    int64  SizeOffset = INDEX_NONE; // location in stream of tag size member  
    FGuid  StructGuid;  
    uint8  HasPropertyGuid = 0;  
    FGuid  PropertyGuid;
```

The fields are used to track any modifications of the serialized properties (Name, Type, Redirection). The `GUID` is used to compare versions of the property to ensure that data isn't lost to major engine changes, I suspect.

`Deserialization` is simply the process in reverse. The `FPropertyTag` is extracted from the uasset file, and then `UClass` is searched for based on the tag information. The serialized data is then mapped to the data object in the correct memory order.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/TPSDeserialize.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/71e18ed9b3cfd2199c045302bc128c4bf673b4a8/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/TPSDeserialize.svg)

In summary, the PropertyTag system makes data more flexible to modification however the trade off introduces more overhead when loading data into memory.
##### UnversionedPropertySerialziation

As mentioned earlier, the UPS system is an optional method during cooking. This implementation method is a more performant option. However, UPS follows a strict sequence rule when data is collected from `UClass`. If the sequence is not maintained for serialization or deserialization, then the stored data is considered invalid.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UPSDeserialize.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/71e18ed9b3cfd2199c045302bc128c4bf673b4a8/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/UPSDeserialize.svg)

After FProperties are serialized, UPS creates a header based on the data serialized.
![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UPSHeader.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/71e18ed9b3cfd2199c045302bc128c4bf673b4a8/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/UPSHeader.svg)

`Deserialization` is again the reverse operation. The generated header data is read from and loaded into the object's memory in the correct order.

One must be careful using UPS because it's more error prone when the object compiled in the editor and the data structure of the asset file after cooking change!

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UPSError.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/71e18ed9b3cfd2199c045302bc128c4bf673b4a8/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/UPSError.svg)
#### FArchive

Unreal Archive system is very powerful and `FArchive` is the main driver for all things related to data serialization in the engine. The UML example below involves `FLinkerLoad` and `FLinkerSave`.  

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/FArchiveUML.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/71e18ed9b3cfd2199c045302bc128c4bf673b4a8/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/FArchiveUML.png)

`FLinkerLoad` and `FLinkerSave` are among the many derived classes of `FArchive` that specialize in loading and saving `uassets`. Other notable derived classes include:

- **FArchiveProxy**: Handles storage of object references (virtual paths), such as redirects and path manipulation.
- **FBitReader/FBitWriter**: Used for creating bitstream formats, typically for network data transmission.
- **FMemoryArchive**: Serializes arbitrary data into memory.
- **FScopeSeekTo**: Useful for setting the archive's position within its scope.

During read and write operations using `FArchive`, the `operator <<` is overloaded to handle specific serialization tasks. It adapts based on context:

_"What goes in must come out the same way."_

To clarify, when loading from an archive, `<<` acts as a read operator. Conversely, when saving to an archive, `<<` is used as a write operator. Thus, `<<` serves as both a read and write operation, depending on the serialization context.

#### UObject Serialization

`UObject` serialization starts with the macro

```cpp
IMPLEMENT_FARCHIVE_SERIALIZER(UObject)
// Expands into
void UObject::Serialize(FArchive& Ar) { UObject::Serialize(FStructuredArchiveFromArchive(Ar).GetSlot().EnterRecord()); }
```

`FStructuredArchive` is another layer of abstraction that wraps an `FArchive`.
##### FStructuredArchive

In an Unreal Fest 2023 talk, Alex Stevens discusses serialization best practices, emphasizing `FStructuredArchive`. This concept aims to maintain structure and state through scopes during serialization, allowing tracking of entry and exit points within an archive. This capability facilitates easier conversion of archive data into different formats.

`FStructuredArchive` consists of several key components:

- **Records**: Containers for named slots.
- **Slots**: Storage locations for values, which can be literals or containers.
- **Containers**:
    - **Arrays**: Contain a fixed number of unnamed child slots.
    - **Streams**: Similar to arrays but unbound in size.
    - **Maps**: Contain a fixed number of named child slots.

Analogously, `FStructuredArchive` functions similarly to JSON, organizing and structuring data for serialization and deserialization processes.

```json
// Start Record
{
	//Slots (Health to Items)
	//Container (Items)
	"Player": "Staticjpl",{
		"Heath": 100,
		"Id": 2246
		"Items":["DestructoDisk","Bfg","PlasmaGun"]
	}
	
	"Player": "Tork",{
		"Heath": 60,
		"Id": 2245
		"Items":["RocketLauncher","Railgun","Grenade"]
	}
}
// End Record
```

In the provided example code, each record within `FStructuredArchive` is encapsulated in `{}` and contains slots defining literals or containers. `FStructuredArchive` abstractly constructs a mental map to facilitate processing of records.

Serialization of `UObject` depends on the type of `FArchive` used. Examples in the source code demonstrate operations like redo/undo tracking, sparse class management, and more. Deserialization occurs after an object is instantiated and an `FArchive` is passed, following these steps:

1. Retrieve the current `UClass` and its `Outer`. If the object belongs to another object, the `Outer` specifies this relationship.
2. Check if the `UClass` information is already loaded; if not:
    - Preload the `UClass` information.
    - Preload the Class Default Object (CDO) for the `UClass`.
3. Load the object's name.
4. Load its `Outer`.
5. Load the `UClass` information for the object.
6. Load all script member variables after loading the `UClass`, as it determines which script member variables need loading.

Next, `SerializeScriptProperties` handles serialization of object properties defined in the class. During saving, properties that differ from the archetype are serialized.

Continuing with the process:

1. Use `MarkScriptSerializationStart` to indicate the beginning of serializing object property data using script serialization (starting with the Export Map's first index).
2. For the specific `UClass`, use `SerializeTaggedProperties` to serialize object properties and add tags.
3. Use `MarkScriptSerializationEnd` to denote the conclusion of object script serialization.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UObjectSerialization.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/71e18ed9b3cfd2199c045302bc128c4bf673b4a8/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/UObjectSerialization.svg)

Code examples below are stripped to the relevant logic

```cpp
void UObject::Serialize(FStructuredArchive::FRecord Record)  
{  
    FArchive& UnderlyingArchive = Record.GetUnderlyingArchive();  

    // These three items are very special items from a serialization standpoint. They aren't actually serialized.  
    UClass* ObjClass = GetClass();  
    UObject* LoadOuter = GetOuter();  
    FName LoadName = GetFName();  

    // Make sure this object's class's data is loaded.  
    if (ObjClass->HasAnyFlags(RF_NeedLoad))  
    {          
        UnderlyingArchive.Preload(ObjClass);  

        // make sure this object's template data is loaded - the only objects  
        // this should actually affect are those that don't have any defaults          
        // to serialize.  for objects with defaults that actually require loading          
        // the class default object should be serialized in FLinkerLoad::Preload, before          
        // we've hit this code.          
        if (!HasAnyFlags(RF_ClassDefaultObject) && ObjClass->GetDefaultsCount() > 0)  
        {             
            UnderlyingArchive.Preload(ObjClass->GetDefaultObject());  
        }       
    }  

    // Special info.  
    if ((!UnderlyingArchive.IsLoading() && !UnderlyingArchive.IsSaving() && !UnderlyingArchive.IsObjectReferenceCollector()))  
    {          
        Record << SA_VALUE(TEXT("LoadName"), LoadName);  
        if (!UnderlyingArchive.IsIgnoringOuterRef())  
        {             
            Record << SA_VALUE(TEXT("LoadOuter"), LoadOuter);  
        }          
        if (!UnderlyingArchive.IsIgnoringClassRef())  
        {             
            Record << SA_VALUE(TEXT("ObjClass"), ObjClass);  
        }       
    }       
	
    // Serialize object properties which are defined in the class.  
    // Handle derived UClass objects (exact UClass objects are native only and shouldn't be touched)       
    if (ObjClass != UClass::StaticClass())  
    {          
        SerializeScriptProperties(Record.EnterField(TEXT("Properties")));  
    }  
    
    // Serialize a GUID if this object has one mapped to it  
    FLazyObjectPtr::PossiblySerializeObjectGuid(this, Record);  

    // Invalidate asset pointer caches when loading a new object  
    if (UnderlyingArchive.IsLoading())  
    {          
        FSoftObjectPath::InvalidateTag();  
    }  

    // Memory counting (with proper alignment to match C++)  
    SIZE_T Size = GetClass()->GetStructureSize();  
    UnderlyingArchive.CountBytes(Size, Size);  
}

```

```cpp
void UObject::SerializeScriptProperties(FStructuredArchive::FSlot Slot) const
{
	FArchive& UnderlyingArchive = Slot.GetUnderlyingArchive();
	
	UnderlyingArchive.MarkScriptSerializationStart(this);
	if (HasAnyFlags(RF_ClassDefaultObject))
	{
		UnderlyingArchive.StartSerializingDefaults();
	}

	UClass* ObjClass = GetClass();
	if (UnderlyingArchive.IsTextFormat() || ((UnderlyingArchive.IsLoading() || UnderlyingArchive.IsSaving()) && !UnderlyingArchive.WantBinaryPropertySerialization()))
	{
		//@todoio GetArchetype is pathological for blueprint classes and the event driven loader; the EDL already knows what the archetype is; just calling this->GetArchetype() tries to load some other stuff.
		UObject* DiffObject = UnderlyingArchive.GetArchetypeFromLoader(this);
		if (!DiffObject)
		{
			DiffObject = GetArchetype();
		}

		ObjClass->SerializeTaggedProperties(Slot, (uint8*)this, HasAnyFlags(RF_ClassDefaultObject) ? ObjClass->GetSuperClass() : ObjClass, (uint8*)DiffObject, bBreakSerializationRecursion ? this : nullptr);
	}
	else if (UnderlyingArchive.GetPortFlags() != 0 && !UnderlyingArchive.ArUseCustomPropertyList)
	{
		//@todoio GetArchetype is pathological for blueprint classes and the event driven loader; the EDL already knows what the archetype is; just calling this->GetArchetype() tries to load some other stuff.
		UObject* DiffObject = UnderlyingArchive.GetArchetypeFromLoader(this);
		if (!DiffObject)
		{
			DiffObject = GetArchetype();
		}
		ObjClass->SerializeBinEx(Slot, const_cast<UObject*>(this), DiffObject, DiffObject ? DiffObject->GetClass() : NULL);
	}
	else
	{
		ObjClass->SerializeBin(Slot, const_cast<UObject*>(this));
	}

	if (HasAnyFlags(RF_ClassDefaultObject))
	{
		UnderlyingArchive.StopSerializingDefaults();
	}
	UnderlyingArchive.MarkScriptSerializationEnd(this);
}
```

#### Tofu Theory

In the "Tofu theory," byte data stored on disk lacks inherent meaning until interpreted. In languages like C++, deserializing base types such as float, bool, or int allocates specific byte lengths. Conceptually, this byte data resembles slices of tofu aligned lengthwise. Each slice, representing a data type like float or bool, occupies a portion of the total tofu's size.

The critical challenge is identifying what type a slice of tofu represents—float, bool, double, or even a string. This is resolved by establishing agreed-upon rules for slicing and ordering the tofu. By enforcing alignment and ordering rules, various data types can be accurately reconstructed.

For example, suppose Tofu A is composed of 1 float, 2 bools, and 3 doubles. These components are arranged in sequence without gaps, frozen together, and later thawed to reconstruct the original slices based on their length and order.

Below is an example found in an Unreal Engine forum post that visually illustrates this concept.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/SertializedExample.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/71e18ed9b3cfd2199c045302bc128c4bf673b4a8/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/SertializedExample.png)

In Unreal Engine C++, serializing different base types, member fields, or sub-objects with pointer references to different types can be more complex than with native C++ types alone. While the tofu analogy works well for serializing basic native types, custom types require a different approach.

Unreal Engine's solution is deeply rooted in the visitor pattern. Within this pattern, a concept of "self-serialization" emerges. Each custom type defines its own serialization function, allowing the process to unfold recursively, akin to navigating a tree structure. When encountering a "custom slice" in the byte array, it signals the presence of a custom type where the serialization format must be explicitly defined.

Applying these concepts to Unreal Engine's deserialization method reveals a two-step process:

1. When loading the class information of an object, data from different segments of the tofu (byte array) provides insights into the structure of those segments. These segments represent different types and collectively form a blueprint or roadmap for the serialized data.
    
2. Using this blueprint and adhering to the slicing rules, each segment of the tofu is carefully encapsulated during serialization. This ensures that the deserialization process accurately reconstructs the original packing arrangement, preserving data integrity and structure.
#### uasset

Below is a refined version with improved grammar:

The following describes the basic structure of a uasset file, which you can verify using a hex editor. For example, I created a basic actor blueprint and uploaded the uasset file to a hex editor website called `hexed.it`.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/uasset.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/71e18ed9b3cfd2199c045302bc128c4bf673b4a8/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/uasset.svg)

**Header Information:**

- File Summary: Contains summarized data about the uasset.
- Name Table: Stores a list of names for objects within the package.
- Import Table: Defines virtual paths and type information for objects imported from other packages.
- Export Table: Stores virtual paths and type information for objects defined within this package.
- GUID: Unique identifiers used for tracking and identifying objects in the import/export maps.
- Exported Objects: Contains the actual data for the objects defined in the export table.

Next, I'll illustrate how the uasset system works using a visual example extracted from the article.
![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UassetObjectExample.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/71e18ed9b3cfd2199c045302bc128c4bf673b4a8/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/UassetObjectExample.svg)

Suppose there is a `UClass` that contains Objects A,B,C,D,E respectively. In the serialized package there are pointer reference between object A->B, C->B and C->E respectively. The problem arises of maintaining the references early on and then reconstructing into the memory despite the addresses changing.

**uasset Serialization**

1. The Export table lists the objects contained within the package, while the Import table holds information about objects referenced from other packages.
    
2. When serializing a UObject pointer, a common issue in C++ serialization arises: `FArchive` cannot directly record the memory address of Object B that Object A references.
    
    a. Read the Export table and mark Object B.
    
    b. Modify Object A's pointer reference field with the Export information of Object B.
    
    c. Determine Object B's Outer using `NewObject`, which is the true owner responsible for serializing Object B.
    
    d. Save relevant information such as Object A's FName and properties. If another UObject is encountered, repeat steps 2-3.
    
3. If `FArchive` encounters Object C pointing to Object E, which is outside the package, it marks Object E with an index of -1 in the import table. Object E's data isn't serialized as its slot remains open.
    
4. Once the tables are saved, process the data of each object one by one.
    

The question arises: how are these pointer references reconstructed when memory changes each time? In computer science, this process of reconstructing references in memory is known as `pointer swizzling`. Conversely, `unswizzling` refers to the reverse operation used to save pointer references.

Unreal Engine uses a swizzling approach that utilizes the virtual path system to generate unique identifiers for unswizzling. For instance, assets and objects serialized under `/Game/Blah/SomeAssetPackage.SomeAsset` in your main content directory are stored relative to where they are "mounted". This path essentially becomes what is saved when serializing an object pointer.

**uasset de-serialization**

Suppose now the memory is completely empty at this time, and there is no object information. `FArchive` comes into play again.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UassetObjectPointeFixup.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/71e18ed9b3cfd2199c045302bc128c4bf673b4a8/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/UassetObjectPointeFixup.svg)


1. `FArchive` loads the serialized asset information and extracts the objects to determine their types.
    
    a. If the `UClass` type is not loaded, then it loads it and reads the CDO. b. Based on the `UClass` information, `FArchive` initially creates a dummy object (empty).
    
2. Depending on the class information, `FArchive` reads the data and identifies the `FProperties`.
    
    a. If the `FProperty` belongs to the base object, it deserializes it immediately, partially initializing the dummy object. b. When encountering a UObject type, `FArchive` checks whether the package index is positive or negative. c. If the index is positive, it checks the Export Table to determine if the object has already been serialized. If so, it replaces the pointer with the existing object. Otherwise, it creates a dummy object and defers loading until the `Outer` handles it. d. If the index is negative, `FArchive` checks the Import Table to see if the corresponding package is already loaded in memory. If not, it loads it and retrieves the object's address from the other package.
    
3. Finally, the entire process concludes:
    
    a. Dummy objects that have not been constructed with `NewObject` and restored to their original form begin to come into existence. b. The dummy object progressively restores its original state as more information is deserialized and incorporated. c. The dummy object is fully restored, and its pointer address is adjusted to the current runtime memory address.
    

Therefore, in the editor, `swizzling` attempts to locate an object with that path or loads it if it isn't already loaded (soft object pointers behave similarly when explicitly loading).

#### Summary

1. Serialization selectively saves only necessary or differential data, optimizing storage and transmission.
2. Objects are initially constructed as skeletons and then restored with their data during deserialization.
3. Responsibility for serialization and deserialization is tied to the object's owner or outer relationship, particularly through the `Outer` passed into `NewObject`.
4. The validity of an object is determined by its memory layout and behavior matching the original object, regardless of its memory address.

These concepts highlight efficient data handling through serialization, the procedural approach to object restoration, and the critical role of ownership relationships in managing object state across serialization cycles.
