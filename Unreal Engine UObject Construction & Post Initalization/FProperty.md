### FProperty

`FProperty` is a reflection type used in Unreal Engine for properties defined in a class or struct marked with a `UPROPERTY` macro. The core implementation of `FProperty` is crucial for blueprints, serialization, garbage collection, and construction.

The reflection document describes in more detail how properties are generated, collected, and registered by the engine's reflection system, which won't be covered here.

Since `UClass` holds the `FProperty` fields of a specific object, the next step is to investigate several questions:

1. What is the difference between `FProperty` and `UProperty`?
2. What is the architecture of `FProperty`?
3. What are the different `FProperty` types, and how are they constructed?

##### UProperty vs FProperty

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/FPropertyDependency.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/7d09b1d9fe2d20f9a77c62d712aef3cfabbd7416/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/FPropertyDependency.svg)

In the early days of Unreal Engine, `UProperty` was tightly coupled with `UObject`. However, `UProperty` has since become completely legacy and is no longer used. Despite this, if encountered, the engine converts `UProperty` into `FProperty` during serialization.

Epic engineers likely realized before version 4.25 that the original tight coupling between `UObject` and `UProperty` would lead to increased overhead at scale. Therefore, it was redesigned into `FProperty` to eliminate the performance bottlenecks of the original design. `FProperty` functions similarly to `UProperty`, with largely the same code, but with the 'F' prefix denoting it's not tied to a `UObject`. Since the base class `FField` is decoupled from `UObject`, what are the performance gains of this design?

1. Blueprints load much faster due to this design.
2. Garbage collection speed increases because there is a reduction in the number of `UObjects`.
3. Memory footprint is reduced because `UProperty` no longer carries additional logic and data specific to `UObjects`.
4. Iterating through `UObjects` becomes faster, and iterating through `FProperty` speeds up to approximately 2x faster than with `UProperty`.
5. Casting using `FProperty` is approximately 3x faster compared to the original `UObject` design.
#### FProperty Architecture

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/FFieldRelationship.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/7d09b1d9fe2d20f9a77c62d712aef3cfabbd7416/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/FFieldRelationship.svg)

`FField` is the base class for `FProperty`. `FProperties` are defined inside a `UClass` by Unreals Reflection System. Each `FField` is referenced in order by a `Linked List`. 

```cpp
//  
// An UnrealScript variable.  
//  
class FProperty : public FField  
{  
    DECLARE_FIELD_API(FProperty, FField, CASTCLASS_FProperty, COREUOBJECT_API)  
  
    // Persistent variables.  
    int32        ArrayDim;  
    int32        ElementSize;  
    EPropertyFlags PropertyFlags;  
    uint16       RepIndex;  
  
private:  
    TEnumAsByte<ELifetimeCondition> BlueprintReplicationCondition;  
  
    // In memory variables (generated during Link()).  
    int32     Offset_Internal;  
  
public:  
    /** In memory only: Linked list of properties from most-derived to base **/  
    FProperty* PropertyLinkNext;  
    /** In memory only: Linked list of object reference properties from most-derived to base **/  
    FProperty*  NextRef;  
    /** In memory only: Linked list of properties requiring destruction. Note this does not include things that will be destroyed by the native destructor **/  
    FProperty* DestructorLinkNext;  
    /** In memory only: Linked list of properties requiring post constructor initialization.**/  
    FProperty* PostConstructLinkNext;  
  
    FName     RepNotifyFunc;
```

The main attributes defined here are as follows.

1. **ArrayDim** - Defines the object array size, in general it's not touched and is set to 1 by default.
2. **ElementSize** - Defines the actual size of the memory occupied for this property.
3. **EPropertyFlags** - Flags to defines the behavior of the property, some examples CPF_Net, CPF_BlueprintReadOnly,CPF_Transient etc.
4. **RepIndex** - Used for network synchronization.
5. **BlueprintReplicationCondition** - Secondary conditional for a FProperty, examples are OwnerOwnly, InitalOnly,SimulatedOnly.
6. **Offset_Internal** - Holds the offset in memory with respect to other properties in a struct or class. This is set by the serialization system using a dummy `FArchive` .
7. **RepNotifyFunc** - The name lookup for the network synchronization call back function specified for the property.

How is `FProperty` constructed and declared? The answer is in the macro `DECLARE_FIELD`. 

```cpp
#define DECLARE_FIELD(TClass, TSuperClass, TStaticFlags) \  
    DECLARE_FIELD_API(TClass, TSuperClass, TStaticFlags, NO_API)  
  
#define DECLARE_FIELD_API(TClass, TSuperClass, TStaticFlags, TRequiredAPI) \  
private: \  
    TClass& operator=(TClass&&);   \  
    TClass& operator=(const TClass&);   \  
public: \  
    typedef TSuperClass Super;\  
    typedef TClass ThisClass;\  
    TClass(EInternal InInernal, FFieldClass* InClass) \  
       : Super(EC_InternalUseOnlyConstructor, InClass) \  
    { \  
    } \  
    static TRequiredAPI FFieldClass* StaticClass(); \  
    static FField* Construct(const FFieldVariant& InOwner, const FName& InName, EObjectFlags InObjectFlags); \  
    inline static constexpr uint64 StaticClassCastFlagsPrivate() \  
    { \  
       return uint64(TStaticFlags); \  
    } \  
    inline static constexpr uint64 StaticClassCastFlags() \  
    { \  
       return uint64(TStaticFlags) | Super::StaticClassCastFlags(); \  
    } \  
    inline void* operator new(const size_t InSize, void* InMem) \  
    { \  
       return InMem; \  
    } \  
    inline void* operator new(const size_t InSize) \  
    { \  
       DECLARE_FIELD_NEW_IMPLEMENTATION(TClass) \  
    } \  
    inline void operator delete(void* InMem) noexcept \  
    { \  
       FMemory::Free(InMem); \  
    } \  
    friend FArchive &operator<<( FArchive& Ar, ThisClass*& Res ) \  
    { \  
       return Ar << (FField*&)Res; \  
    } \  
    friend void operator<<(FStructuredArchive::FSlot InSlot, ThisClass*& Res) \  
    { \  
       InSlot << (FField*&)Res; \  
    }  
  
#if !CHECK_PUREVIRTUALS  
    #define IMPLEMENT_FIELD_CONSTRUCT_IMPLEMENTATION(TClass) \  
       FField* Instance = new TClass(InOwner, InName, InFlags); \  
       return Instance; #else  
    #define IMPLEMENT_FIELD_CONSTRUCT_IMPLEMENTATION(TClass) \  
       return nullptr;  
#endif
```

The declaration code is similar to `DECLARE_CLASS`, where the macro contains boilerplate code for operators like `new`, defining memory allocation. As seen earlier, operators like `<<` are reserved for serialization, and static helper functions like `StaticClass` return the class in which they reside.

Since `FProperty` is decoupled from the `UObject` lifecycle, it is NOT automatically handled by the garbage collector. `FProperties` are typically created within a `UStruct`, and if a `UStruct` is destroyed, its own property instances must be traversed and deallocated using the `delete` operation.
##### Cast/Cast Field & Outer/Owner

`UObject` utilizes the concept of an `Outer` within the reflection system. In contrast, `FProperty` employs the same concept of `Owner`. The `Owner` of an `FProperty` is obtained similarly through a getter function called `GetOwner`, analogous to how `GetOuter` is used for `UObject`.

Casting in `UObject` differs from `FProperty`. `UObject` types use the `Cast` function explicitly for performance reasons. Similarly, `FProperty` provides its own casting function called `CastField`. Internally, both use templated C++ `static_cast`.

```cpp
// Support for casting between different FFIeld types  
template<typename FieldType>  
FORCEINLINE FieldType* CastField(FField* Src)  
{  
    return Src && Src->IsA<FieldType>() ? static_cast<FieldType*>(Src) : nullptr;  
}
```

`CastClass` flags are used with a `FFieldClass` object to describe the type of `FProperty`. 

```cpp
/**  
 * Flags used for quickly casting classes of certain types; all class cast flags are inherited * * This MUST be kept in sync with EClassCastFlags defined in * Engine\Source\Programs\Shared\EpicGames.Core\UnrealEngineTypes.cs */
enum EClassCastFlags : uint64  
{  
    CASTCLASS_None = 0x0000000000000000,  
  
    CASTCLASS_UField                  = 0x0000000000000001,  
    CASTCLASS_FInt8Property                = 0x0000000000000002,  
    CASTCLASS_UEnum                      = 0x0000000000000004,  
    CASTCLASS_UStruct                 = 0x0000000000000008,  
    CASTCLASS_UScriptStruct                = 0x0000000000000010,  
    CASTCLASS_UClass                  = 0x0000000000000020,  
    CASTCLASS_FByteProperty                = 0x0000000000000040,  
    CASTCLASS_FIntProperty             = 0x0000000000000080,  
    CASTCLASS_FFloatProperty            = 0x0000000000000100,  
    CASTCLASS_FUInt64Property           = 0x0000000000000200,  
    CASTCLASS_FClassProperty            = 0x0000000000000400,  
    CASTCLASS_FUInt32Property           = 0x0000000000000800,  
    CASTCLASS_FInterfaceProperty         = 0x0000000000001000,  
    CASTCLASS_FNameProperty                = 0x0000000000002000,  
    CASTCLASS_FStrProperty             = 0x0000000000004000,  
    CASTCLASS_FProperty                   = 0x0000000000008000,  
    CASTCLASS_FObjectProperty           = 0x0000000000010000,  
    CASTCLASS_FBoolProperty                = 0x0000000000020000,  
    CASTCLASS_FUInt16Property           = 0x0000000000040000,  
    CASTCLASS_UFunction                   = 0x0000000000080000,  
    CASTCLASS_FStructProperty           = 0x0000000000100000,  
    CASTCLASS_FArrayProperty            = 0x0000000000200000,  
    CASTCLASS_FInt64Property            = 0x0000000000400000,  
    CASTCLASS_FDelegateProperty             = 0x0000000000800000,  
    CASTCLASS_FNumericProperty          = 0x0000000001000000,  
    CASTCLASS_FMulticastDelegateProperty   = 0x0000000002000000,  
    CASTCLASS_FObjectPropertyBase        = 0x0000000004000000,  
    CASTCLASS_FWeakObjectProperty        = 0x0000000008000000,  
    CASTCLASS_FLazyObjectProperty        = 0x0000000010000000,  
    CASTCLASS_FSoftObjectProperty        = 0x0000000020000000,  
    CASTCLASS_FTextProperty                = 0x0000000040000000,  
    CASTCLASS_FInt16Property            = 0x0000000080000000,  
    CASTCLASS_FDoubleProperty           = 0x0000000100000000,  
    CASTCLASS_FSoftClassProperty         = 0x0000000200000000,  
    CASTCLASS_UPackage                = 0x0000000400000000,  
    CASTCLASS_ULevel                  = 0x0000000800000000,  
    CASTCLASS_AActor                  = 0x0000001000000000,  
    CASTCLASS_APlayerController             = 0x0000002000000000,  
    CASTCLASS_APawn                      = 0x0000004000000000,  
    CASTCLASS_USceneComponent           = 0x0000008000000000,  
    CASTCLASS_UPrimitiveComponent        = 0x0000010000000000,  
    CASTCLASS_USkinnedMeshComponent          = 0x0000020000000000,  
    CASTCLASS_USkeletalMeshComponent      = 0x0000040000000000,  
    CASTCLASS_UBlueprint               = 0x0000080000000000,  
    CASTCLASS_UDelegateFunction             = 0x0000100000000000,  
    CASTCLASS_UStaticMeshComponent       = 0x0000200000000000,  
    CASTCLASS_FMapProperty             = 0x0000400000000000,  
    CASTCLASS_FSetProperty             = 0x0000800000000000,  
    CASTCLASS_FEnumProperty                = 0x0001000000000000,  
    CASTCLASS_USparseDelegateFunction        = 0x0002000000000000,  
    CASTCLASS_FMulticastInlineDelegateProperty = 0x0004000000000000,  
    CASTCLASS_FMulticastSparseDelegateProperty = 0x0008000000000000,  
    CASTCLASS_FFieldPathProperty         = 0x0010000000000000,  
    CASTCLASS_FObjectPtrProperty         = 0x0020000000000000,  
    CASTCLASS_FClassPtrProperty             = 0x0040000000000000,  
    CASTCLASS_FLargeWorldCoordinatesRealProperty = 0x0080000000000000,  
    CASTCLASS_FOptionalProperty             = 0x0100000000000000,  
};
```

Unreal Engine employs a bitwise ID system for numerous `FProperty` types utilizing `EClassCastFlags`. This system enables bitwise mask operations using 64 bits to differentiate between `UObject` types and `FProperty` types. What's particularly intriguing is the inclusion of casting types for gameplay elements such as `AActor`, `APawn`, `SceneComponent`, and `APlayerController`.

Inside `DECLARE_FIELD`

```cpp
inline static constexpr uint64 StaticClassCastFlagsPrivate() \  
{ \  
    return uint64(TStaticFlags); \  
} \  
inline static constexpr uint64 StaticClassCastFlags() \  
{ \  
    return uint64(TStaticFlags) | Super::StaticClassCastFlags(); \  
} \
```

The functions, assumed to be compile-time evaluated when passed to the `DECLARE_FIELD` macro, include one returning a `CastClass` and another providing value class inheritance information. These evaluated values serve as parameters within the `FFieldClass` constructor.

```cpp
FFieldClass* FField::StaticClass()  
{  
    static FFieldClass StaticFieldClass(TEXT("FField"), FField::StaticClassCastFlagsPrivate(), FField::StaticClassCastFlags(), nullptr, &FField::Construct);  
    return &StaticFieldClass;  
}
```

For instance, `IsA()` operates significantly faster compared to the traditional `UProperty` design. The additional overhead in casting arises because the `UObject` system necessitates traversing the `UClass` `SuperStruct` inheritance chain.
##### FField & FFieldClass

`FField` and `FFieldClass` serve as the foundational classes that facilitate property functionality within the Reflection System. They are particularly instrumental when `UStruct` aggregates properties marked with the `UProperty` macro. These classes provide core functionalities such as runtime traversal of properties/types, serialization of properties (both on disk and in memory), and support for custom initialization behaviors. 

```cpp
/**
 * Base class of reflection data objects.
 */
class FField
{
	UE_NONCOPYABLE(FField);

	/** Pointer to the class object representing the type of this FField */
	FFieldClass* ClassPrivate;

public:
	typedef FField Super;
	typedef FField ThisClass;
	typedef FField BaseFieldClass;	
	typedef FFieldClass FieldTypeClass;

	static COREUOBJECT_API FFieldClass* StaticClass();

	inline static constexpr uint64 StaticClassCastFlagsPrivate()
	{
		return uint64(CASTCLASS_UField);
	}
	inline static constexpr uint64 StaticClassCastFlags()
	{
		return uint64(CASTCLASS_UField);
	}

	/** Owner of this field */
	FFieldVariant Owner;

	/** Next Field in the linked list */
	FField* Next;

	/** Name of this field */
	FName NamePrivate;

	/** Object flags */
	EObjectFlags FlagsPrivate;
```

The objective is to transition away from `UField`, yet `FField` retains a layout very similar to its predecessor. It maintains a pointer to a linked list of property fields as per the original design. However, in the absence of a `UObject` predecessor, `FFieldClass` assumes the role of `UClass` for type reflection purposes.

```cpp
/**  
  * Object representing a type of an FField struct.  * Mimics a subset of UObject reflection functions.  
  */
class FFieldClass  
{  
    UE_NONCOPYABLE(FFieldClass);  
  
    /** Name of this field class */  
    FName Name;  
    /** Unique Id of this field class (for casting) */  
    uint64 Id;  
    /** Cast flags used for casting to other classes */  
    uint64 CastFlags;  
    /** Class flags */  
    EClassFlags ClassFlags;  
    /** Super of this class */  
    FFieldClass* SuperClass;     
/** Default instance of this class */  
    FField* DefaultObject;  
    /** Pointer to a function that can construct an instance of this class */  
    FField* (*ConstructFn)(const FFieldVariant&, const FName&, EObjectFlags);  
    /** Counter for generating runtime unique names */  
    FThreadSafeCounter UnqiueNameIndexCounter;  
  
    /** Creates a default object instance of this class */  
    COREUOBJECT_API FField* ConstructDefaultObject();
```

According to engine comments, `FFieldClass` represents the type of the `FField` struct. The primary attributes of `FFieldClass` are also outlined in these comments.

As previously mentioned, `UStruct` serves the purpose of aggregating property types and exposing fields for post-initialization tasks and serialization operations. Let's now briefly examine the base class responsible for property containment.

```cpp
/** 
 * Base class for all UObject types that contain fields. */
class UStruct : public UField  
#if USTRUCT_FAST_ISCHILDOF_IMPL == USTRUCT_ISCHILDOF_STRUCTARRAY  
    , private FStructBaseChain  
#endif  
{  
    DECLARE_CASTED_CLASS_INTRINSIC_WITH_API(UStruct, UField, CLASS_MatchedSerializers, TEXT("/Script/CoreUObject"), CASTCLASS_UStruct, COREUOBJECT_API)  
  
    // Variables.  
protected:  
    friend struct Z_Construct_UClass_UStruct_Statics;  
private:  
    /** Struct this inherits from, may be null */  
    ObjectPtr_Private::TNonAccessTrackedObjectPtr<UStruct> SuperStruct;  
public:  
    /** Pointer to start of linked list of child fields */  
    TObjectPtr<UField> Children;  
    /** Pointer to start of linked list of child fields */  
    FField* ChildProperties;  
  
    /** Total size of all UProperties, the allocated structure may be larger due to alignment */  
    int32 PropertiesSize;  
    /** Alignment of structure in memory, structure will be at least this large */  
    int32 MinAlignment;  
    /** Script bytecode associated with this object */  
    TArray<uint8> Script;  
    
    /** In memory only: Linked list of properties from most-derived to base */  
    FProperty* PropertyLink;  
    /** In memory only: Linked list of object reference properties from most-derived to base */  
    FProperty* RefLink;  
    /** In memory only: Linked list of properties requiring destruction. Note this does not include things that will be destroyed by the native destructor */  
    FProperty* DestructorLink;  
    /** In memory only: Linked list of properties requiring post constructor initialization */  
    FProperty* PostConstructLink;  
  
    /** Array of object references embedded in script code and referenced by FProperties. Mirrored for easy access by realtime garbage collection code */  
    TArray<TObjectPtr<UObject>> ScriptAndPropertyObjectReferences;
```

The main pointers for managing traversal are.

- FField*         ChildProperties
- FProperty* PropertyLink
- FProperty* RefLink
- FProperty* DestuctorLink
- FProperty* PostConstructLink

The pointer `ChildProperties` remains functional for traversing children properties within the structure. Essentially, `ChildProperties` is updated to the value of the `Next` pointer of `FField`. Despite efforts to move away from `UProperty`, remnants of its influence still persist in the Engine. As a result, this introduces `FFieldVariant`.

```cpp
/**  
 * Special container that can hold either UObject or FField. * Exposes common interface of FFields and UObjects for easier transition from UProperties to FProperties. * DO NOT ABUSE. IDEALLY THIS SHOULD ONLY BE FFIELD INTERNAL STRUCTURE FOR HOLDING A POINTER TO THE OWNER OF AN FFIELD. */
class FFieldVariant  
{  
    union FFieldObjectUnion  
    {  
       FField* Field;  
       UObject* Object;  
    } Container;
...
}
```

As mentioned, the role of `FFieldVariant` is to act as a container object that can either hold an `FField` or a `UObject`. It serves as an interface specifically designed to manage the transition of `UProperty` types to `FProperty`. Additionally, `FFieldVariant` includes a pointer to the owner of an `FField`.

One notable use case is in the construction of `FProperty`.

```cpp
FProperty::FProperty(FFieldVariant InOwner, const FName& InName, EObjectFlags InObjectFlags, int32 InOffset, EPropertyFlags InFlags)  
    : FField(InOwner, InName, InObjectFlags)  
    , ArrayDim(1)  
    , ElementSize(0)  
    , PropertyFlags(InFlags)  
    , RepIndex(0)  
    , BlueprintReplicationCondition(COND_None)  
    , Offset_Internal(InOffset)  
    , PropertyLinkNext(nullptr)  
    , NextRef(nullptr)  
    , DestructorLinkNext(nullptr)  
    , PostConstructLinkNext(nullptr)  
{  
    Init();  
}
```

Inside `Init()` GetOwnerChecked returns the associated `UStruct` pointed by `FFieldVariant` and calls it's member function`AddCppProperty`. 

```cpp
void FProperty::Init()  
{  
#if !WITH_EDITORONLY_DATA  
    //@todo.COOKER/PACKAGER: Until we have a cooker/packager step, this can fire when WITH_EDITORONLY_DATA is not defined!  
    // checkSlow(!HasAnyPropertyFlags(CPF_EditorOnly));#endif // WITH_EDITORONLY_DATA  
    checkSlow(GetOwnerUField()->HasAllFlags(RF_Transient));  
    checkSlow(HasAllFlags(RF_Transient));  
  
    if (GetOwner<UObject>())  
    {       UField* OwnerField = GetOwnerChecked<UField>();  
       OwnerField->AddCppProperty(this);  
    }    
    else  
    {  
       FField* OwnerField = GetOwnerChecked<FField>();  
       OwnerField->AddCppProperty(this);  
    }}
...
}
```

```cpp
void UStruct::AddCppProperty(FProperty* Property)  
{  
    Property->Next = ChildProperties;  
    ChildProperties = Property;  
}
```

At the tail end of the reflection registration phase, the serialization system performs "linking" to establish property flags, fix memory addresses, and organize property fields within the `UStruct`. The reflection document provides extensive details on this process, from which I summarized how the linked list pointers for `FProperty` are configured.

```cpp
void UStruct::Link(FArchive& Ar, bool bRelinkExistingProperties)
{
	// Go through All FProperties and call LinkInternal on them
	for (FField* Field = ChildProperties; (Field != NULL) && (Field->GetOwner<UObject>() == this); Field = Field->Next)
	{
		if (FProperty* Property = CastField<FProperty>(Field))
		{
			Property->LinkWithoutChangingOffset(Ar);
		}
	}

	// Link the references, structs, and arrays for optimized cleanup.
	// Note: Could optimize further by adding FProperty::NeedsDynamicRefCleanup, excluding things like arrays of ints.
	FProperty** PropertyLinkPtr = &PropertyLink;
	FProperty** DestructorLinkPtr = &DestructorLink;
	FProperty** RefLinkPtr = (FProperty**)&RefLink;
	FProperty** PostConstructLinkPtr = &PostConstructLink;

	TArray<const FStructProperty*> EncounteredStructProps;
	for (TFieldIterator<FProperty> It(this); It; ++It)
	{
		FProperty* Property = *It;

		// Ref link contains any properties which contain object references including types with user-defined serializers which don't explicitly specify whether they
		// contain object references
		if (Property->ContainsObjectReference(EncounteredStructProps, EPropertyObjectReferenceType::Any))
		{
			*RefLinkPtr = Property;
			RefLinkPtr = &(*RefLinkPtr)->NextRef;
		}
		const UClass* OwnerClass = Property->GetOwnerClass();
		bool bOwnedByNativeClass = OwnerClass && OwnerClass->HasAnyClassFlags(CLASS_Native | CLASS_Intrinsic);

		if (!Property->HasAnyPropertyFlags(CPF_IsPlainOldData | CPF_NoDestructor) &&
			!bOwnedByNativeClass) // these would be covered by the native destructor
		{
			// things in a struct that need a destructor will still be in here, even though in many cases they will also be destroyed by a native destructor on the whole struct
			*DestructorLinkPtr = Property;
			DestructorLinkPtr = &(*DestructorLinkPtr)->DestructorLinkNext;
		}
		// Link references to properties that require their values to be initialized and/or copied from CDO post-construction. Note that this includes all non-native-class-owned properties.
		if (OwnerClass && (!bOwnedByNativeClass || (Property->HasAnyPropertyFlags(CPF_Config) && !OwnerClass->HasAnyClassFlags(CLASS_PerObjectConfig))))
		{
			*PostConstructLinkPtr = Property;
			PostConstructLinkPtr = &(*PostConstructLinkPtr)->PostConstructLinkNext;
		}
		*PropertyLinkPtr = Property;
		PropertyLinkPtr = &(*PropertyLinkPtr)->PropertyLinkNext;
	}

	// Now collect all references from FProperties to UObjects and store them in GC-exposed array for fast access
	CollectPropertyReferencedObjects(MutableView(ScriptAndPropertyObjectReferences));
}
```

- `PropertyLink`: Points to the most derived base class. Updated inside the for loop when traversing `UStruct` properties.
- `RefLink`: Points to properties with object references (Components, SubObjects, etc.), managed by the garbage collector (GC).
- `PostConstructorLink`: Used to retrieve original default values from the CDO. Attribute values can be set from this CDO or a file.
- `DestructorLink`: Identifies properties requiring additional destruction.

These pointers are designed to optimize performance in various scenarios by reducing traversal of specific attributes. Understanding the serialization system reveals that post-initialization of native property types is simpler compared to initializing `UObject` instances with references. Hence, `PostConstructorLink`, `RefLink`, and `DestructorLink` prove invaluable during serialization operations from disk or when overriding nested default subobjects with `FObjectInitializer`.
#### FProperty Types

In `ttod_qzstudios` article the type system diagram was obtained. The relationships between the types are basically the same as `UProperty`.
![[Unreal Engine UObject Construction & Post Initalization/Diagrams/FPropertyLayout.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/7d09b1d9fe2d20f9a77c62d712aef3cfabbd7416/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/FPropertyLayout.svg)

 The reflection system inserts property types to a PropPointers[] array located instead the generated file. In code example below I have a `float` property collected.

```cpp
const UECodeGen_Private::FPropertyParamsBase* const Z_Construct_UScriptStruct_FMyStruct_Statics::PropPointers[] = {  
    (const UECodeGen_Private::FPropertyParamsBase*)&Z_Construct_UScriptStruct_FMyStruct_Statics::NewProp_Score,  
};
```

The PropPointer members are of type `FPropertyParamsBase`

```cpp
// This is not a base class but is just a common initial sequence of all of the F*PropertyParams types below.  
// We don't want to use actual inheritance because we want to construct aggregated compile-time tables of these things.  
struct FPropertyParamsBase  
{  
    const char*    NameUTF8;  
    const char*       RepNotifyFuncUTF8;  
    EPropertyFlags    PropertyFlags;  
    EPropertyGenFlags Flags;  
    EObjectFlags   ObjectFlags;  
    SetterFuncPtr  SetterFunc;  
    GetterFuncPtr  GetterFunc;  
    uint16         ArrayDim;  
};
```

`FFloatPropertyParams` is really a FGenericPropertyParams and the GenericPropertyParam has a layout like this.

```cpp
struct FGenericPropertyParams // : FPropertyParamsBaseWithOffset  
{  
   const char*      NameUTF8;  
   const char*       RepNotifyFuncUTF8;  
   EPropertyFlags    PropertyFlags;  
   EPropertyGenFlags Flags;  
   EObjectFlags     ObjectFlags;  
   SetterFuncPtr  SetterFunc;  
   GetterFuncPtr  GetterFunc;  
   uint16           ArrayDim;  
   uint16           Offset;  
#if WITH_METADATA  
   uint16                              NumMetaData;  
   const FMetaDataPairParam*           MetaDataArray;  
#endif  
};
```

the property type can be inferred from the additional struct member. Since a struct's memory is designed to be contiguous, the `Offset` difference between `FPropertyParamsBase` and `FGenericPropertyParams` enables type casting. Understanding this, during the registration phase, `ConstructFProperty` recursively processes all `FProperty` types from the `Props` array. The `EPropertyGenFlag` is identified and inserted as the template parameter to construct the correct `FProperty`. This design follows a form of Template Meta Programming, which I am not entirely familiar with.
#### Summary

To summarize, `FProperty` represents a more efficient alternative for the reflection system compared to `UProperty`. Since the decoupling from `UObject`, the concepts of `GetOuter` and `Cast` have been redefined. Notably, `FProperty` is not garbage collected; instead, its allocation and deallocation are managed by `UStruct`. The base classes `FField` and `FFieldClass` collaborate to establish a linked list traversal system for reflected property types.

In scenarios involving legacy code, `UProperty` is often converted to `FProperty` using a `FFieldVariant` container object. `FFieldVariant` also references the owner of an `FField`. During the final stages of registration, `Linking` of a `UStruct` occurs to resolve symbol addresses and assign additional `FProperty` pointers. These `FProperty` pointers facilitate not only property traversal but also identify Class Default Objects (CDOs) during construction and manage attributes necessary for destruction.

With this understanding, the `PostInitialization` process of various `FProperty` types can be comprehended more thoroughly in the concluding sections of `FObjectInitializer`.
