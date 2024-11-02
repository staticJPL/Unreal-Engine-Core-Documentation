
# Unreal Engine UObject Construction & Post Initialization Architecture

### Reflection and UObject

The reflection system sets up the creation of the core `UObject` within `UClass`. Without going through the entire reflection system process, the main focus is on how the reflection system sets up the construction process until `NewObject` is called to instance objects. After the Core module is fully loaded and native types are allocated, the engine CDOs (Create Default Subobject) and all the gameplay systems (Actors, Components) construct their subsequent UObjects using `NewObject` to allocate them into a Global UObject Array.

During the `Generation` phase, the Unreal Header Tool parses for the macros `UFunction`, `UEnum`, `UClass`, and `UStruct`. UHT reads these macros and inserts C++ code into `.generated` files which hold the definitions of the type information that are collected during dynamic linking.

The Unreal Engine is broken up into modules, which also perform `static collection`.

In Editor mode, each module is processed in dependency order, starting with `CoreUObject`. `UPackage` is constructed first and defines an `Outer` object for all the UObjects within that module. This `Outer` reference is used to define the UObject hierarchy of that module and aids in `serialization` and `GC`. The CoreUObject module's name is referenced by the string name `"/Script/CoreUObject"`.

Moreover, if any type information changes within a module, these changes are tied to the binary itself. Returning focus to `UClass`, inside the generated code at the bottom of the cpp file, you'll find the static struct `FClassRegisterCompiledInfo`.

During dynamic linking of each module, the `UClass` type information is added to the global static array `FClassDeferredRegistry`. This is done cleverly through a constructor call of `FClassRegisterCompiledInfo`, which then forwards the `Outer` and `Inner` registrant function pointers.

```cpp
/**  
 * Helper class to perform registration of object information.  It blindly forwards a call to RegisterCompiledInInfo */
struct FRegisterCompiledInInfo  
{  
    template <typename ... Args>  
    FRegisterCompiledInInfo(Args&& ... args)  
    {       
	    RegisterCompiledInInfo(std::forward<Args>(args)...);  
    }};
}
```

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UObjectRegistration.png]]

The image illustrates the linking phase, adding a `UClass` registrant for `UObject`.

- `FRegisterCompiledInfo` constructor is called, forwarding a call to `RegisterCompiledInInfo`.
- `Package` is defined as "Script/CoreUObject".
- `FRegistrant` is added to the appropriate registry, passing the function pointers `InOuterRegister` and `InInnerRegister`.

`InOuterRegister` holds `Z_Construct_UClass_UObject` and `InInnerRegister` holds `UObject::StaticClass()`, both of which are function pointers generated and inserted by UHT.

Furthermore, the engine code below illustrates the different deferred registry arrays that will be processed during engine startup.

```cpp
using FClassDeferredRegistry = TDeferredRegistry<FClassRegistrationInfo>; 
using FEnumDeferredRegistry = TDeferredRegistry<FEnumRegistrationInfo>;  
using FStructDeferredRegistry = TDeferredRegistry<FStructRegistrationInfo>;  
using FPackageDeferredRegistry = TDeferredRegistry<FPackageRegistrationInfo>;
```

In the special case of `UClass`, the inner registrant function pointers are defined by the macro `IMPLEMENT_CLASS`, where `StaticClass()` is called from the `UClassRegisterAllCompiledInClasses` phase.

```cpp
for (const FClassDeferredRegistry::FRegistrant& Registrant : Registry.GetRegistrations())  
{       
	UClass* RegisteredClass = FClassDeferredRegistry::InnerRegister(Registrant);   
}
```

In the case of a `UObject`, the **construction** of its `UClass` is defined inside `GetPrivateStaticClass()`. This pattern follows for modules and their UObjects that need an allocated `UClass` for reflection.

```cpp
ReturnClass = (UClass*)GUObjectAllocator.AllocateUObject(sizeof(UClass), alignof(UClass), true);
ReturnClass = ::new (ReturnClass)  
    UClass  
    (  
    EC_StaticConstructor,  
    Name,  
    InSize,  
    InAlignment,  
    InClassFlags,  
    InClassCastFlags,  
    InConfigName,  
    EObjectFlags(RF_Public | RF_Standalone | RF_Transient | RF_MarkAsNative | RF_MarkAsRootSet),  
    InClassConstructor,  
    InClassVTableHelperCtorCaller,  
    MoveTemp(InCppClassStaticFunctions)  
    );  
check(ReturnClass);  
  
InitializePrivateStaticClass(  
    InSuperClassFn(),  
    ReturnClass,  
    InWithinClassFn(),  
    PackageName,  
    Name  
    );
```

From the code snippet, `GUObjectAllocator` is a class used to call `Malloc` with an appropriate size and memory alignment for `UObjectBase`. Early on, the engine allocates a memory block for `UClass` and then constructs its skeleton at the address of that memory block.

In C++, this process is known as `placement new`. It's best practice for game engines to pre-allocate memory in advance. This is because if you pre-allocate the memory block first, you can reduce overhead from the OS when repeated calls are made with `new`. The OS needs to find a spot in memory to allocate space for every new object constructed. Thus, with the size of Unreal Engine, constructing many objects increases this overhead.

UHT inserts additional operator declarations that will be called later in the registration process, so let's expand the `DECLARE_CLASS` macro below.

```cpp
/*-----------------------------------------------------------------------------  
    Class declaration macros.-----------------------------------------------------------------------------*/  
  
#define DECLARE_CLASS( TClass, TSuperClass, TStaticFlags, TStaticCastFlags, TPackage, TRequiredAPI  ) \  
private: \  
    TClass& operator=(TClass&&);   \  
    TClass& operator=(const TClass&);   \  
    TRequiredAPI static UClass* GetPrivateStaticClass(); \  
public: \  
    /** Bitwise union of #EClassFlags pertaining to this class.*/ \  
    static constexpr EClassFlags StaticClassFlags=EClassFlags(TStaticFlags); \  
    /** Typedef for the base class ({{ typedef-type }}) */ \  
    typedef TSuperClass Super;\  
    /** Typedef for {{ typedef-type }}. */ \  
    typedef TClass ThisClass;\  
    /** Returns a UClass object representing this class at runtime */ \  
    inline static UClass* StaticClass() \  
    { \  
       return GetPrivateStaticClass(); \  
    } \  
    /** Returns the package this class belongs in */ \  
    inline static const TCHAR* StaticPackage() \  
    { \  
       return TPackage; \  
    } \  
    /** Returns the static cast flags for this class */ \  
    inline static EClassCastFlags StaticClassCastFlags() \  
    { \  
       return TStaticCastFlags; \  
    } \  
    /** For internal use only; use StaticConstructObject() to create new objects. */ \  
    inline void* operator new(const size_t InSize, EInternal InInternalOnly, UObject* InOuter = (UObject*)GetTransientPackage(), FName InName = NAME_None, EObjectFlags InSetFlags = RF_NoFlags) \  
    { \  
       return StaticAllocateObject(StaticClass(), InOuter, InName, InSetFlags); \  
    } \  
    /** For internal use only; use StaticConstructObject() to create new objects. */ \  
    inline void* operator new( const size_t InSize, EInternal* InMem ) \  
    { \  
       return (void*)InMem; \  
    } \  
    /* Eliminate V1062 warning from PVS-Studio while keeping MSVC and Clang happy. */ \  
    inline void operator delete(void* InMem) \  
    { \  
       ::operator delete(InMem); \  
    }
```

`#define IMPLEMENT_CLASS(TClass, TClassCrc)` macro that expands out to `GetPrivateStaticClass`

```cpp
// Implement the GetPrivateStaticClass and the registration info but do not auto register the class.  // This is primarily used by UnrealHeaderTool  
#define IMPLEMENT_CLASS_NO_AUTO_REGISTRATION(TClass) \  
    FClassRegistrationInfo Z_Registration_Info_UClass_##TClass; \  
    UClass* TClass::GetPrivateStaticClass() \  
    { \  
       if (!Z_Registration_Info_UClass_##TClass.InnerSingleton) \  
       { \  
          /* this could be handled with templates, but we want it external to avoid code bloat */ \  
          GetPrivateStaticClassBody( \  
             StaticPackage(), \  
             (TCHAR*)TEXT(#TClass) + 1 + ((StaticClassFlags & CLASS_Deprecated) ? 11 : 0), \  
             Z_Registration_Info_UClass_##TClass.InnerSingleton, \  
             StaticRegisterNatives##TClass, \  
             sizeof(TClass), \  
             alignof(TClass), \  
             TClass::StaticClassFlags, \  
             TClass::StaticClassCastFlags(), \  
             TClass::StaticConfigName(), \  
             (UClass::ClassConstructorType)InternalConstructor<TClass>, \  
             (UClass::ClassVTableHelperCtorCallerType)InternalVTableHelperCtorCaller<TClass>, \  
             UOBJECT_CPPCLASS_STATICFUNCTIONS_FORCLASS(TClass), \  
             &TClass::Super::StaticClass, \  
             &TClass::WithinClass::StaticClass \  
          ); \  
       } \  
       return Z_Registration_Info_UClass_##TClass.InnerSingleton; \  
    }
```

`UClass` has three different constructors callable depending on the function signatures and the current state of the engine. Therefore, `StaticClass` is called in multiple scenarios or multiple times.

Depending on the function parameters, these cases are:

- The internal constructor
- A new `UClass` is given its `SuperClass`
- Statically linked (Deferred Registration)

### UObject Constructor Setup

During `Preinit()`, CoreUObject's `StartupModule` function calls `UClassRegisterAllCompiledInClasses()` to process the inner registrant function pointers passed into `FClassDeferredRegistry`. These function pointers are used to construct a `UClass` object for all module objects.

Before `NewObject` calls can be made to construct any `UObject`s, the `UObject` itself must be initialized with its `Outer` `UPackage` using `NewObject`.

UHT inserts additional generated code that defines the construction and allocation of `UObject`s (and their derived classes). These macros are used later in the registration phase or runtime:

- `operator new` returning `StaticAllocateObject`
- `operator new` returning `(void*)InMem`
- `__DefaultConstructor``

##### DefaultConstructor

Default constructors are static type class functions defined by macros with UHT.

In the case of UObject

```cpp
class COREUOBJECT_API UObject : public UObjectBaseUtility { DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(UObject) };
```

As seen below, a derived class this function is overridden by a subclass 

```cpp
// Sample UMyClass.gen.h
/** Standard constructor, called after all reflected properties have been initialised */ 
NO_API UMyClass(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get()); DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(UMyClass)
```

```cpp
#define DEFINE_DEFAULT_CONSTRUCTOR_CALL(TClass)  
static void __DefaultConstructor(const FObjectInitializer& X) 
{ 
	new((EInternal*)X.GetObj())TClass;  
}
    
#define DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(TClass) 
static void __DefaultConstructor(const FObjectInitializer& X) 
{ 
	new((EInternal*)X.GetObj())TClass(X);
}

```

`__DefaultConstructor` is a function pointer of type `void` that eventually gets `typedef`ed to `ClassConstructorType`. `ClassConstructor` is a variable that holds the function pointer.

This approach addresses the limitation in C++ where it's not possible to create a direct function pointer to a constructor.

The macro `DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL` has two cases:

1. If only a default constructor is present in a class, then `DEFINE_DEFAULT_CONSTRUCTOR_CALL` is used.
2. If a constructor with `FObjectInitializer` and an `X` parameter is present in a class, then `DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL` is used.

Recall, `T::StaticClass()->GetStaticPrivateClass()->GetPrivateStaticClassBody()` is called, where `TClass` is passed as the template parameter for `InternalConstructor`.

```cpp
/**
 * Helper template to call the default constructor for a class
 */
template<class T>
void InternalConstructor( const FObjectInitializer& X )
{ 
    T::__DefaultConstructor(X);
}
```

These construction variations exist to encapsulate how you call the constructor of `UObject` when there might be a subclass involved.

1. In C++, the constructor of a regular class cannot be pointed to directly by a function pointer. Therefore, the first layer involves using a normal function that will eventually be pointed to by a function pointer.

```cpp
static void __DefaultConstructor(const FObjectInitializer& X) { new((EInternal*)X.GetObj())TClass(X); }
```

2.  `UMyClass::_DefaultConstructor` is another layer to a template function call.

```cpp
template<class T>
void InternalConstructor( const FObjectInitializer& X )
{ 
    T::__DefaultConstructor(X);
}
```

3. The `InternalConstructor` can be wrapped inside another function, which would suffice. However, an issue arises with templates in C++. Templating introduces additional code bloat if every subclass type is templated.

The solution is to template **only** the `InternalConstructor` function and make a single call to it.

Remember:

`#define DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(TClass)` is the macro inserted in `gen.h` file for each subclass (`TClass`).

`typedef` gives a constructor pointer to the specific subclass, which becomes an alias of `ClassConstructorType`. Reflection is advantageous here because the type information is known at runtime. Thus, you can significantly reduce templated bloat code for subclasses by passing the appropriate parameter to `GetPrivateStaticClassBody`. Additionally, at runtime, you could traverse through `UClass` and assign `ClassConstructorType`.

```cpp
typedef void (*ClassConstructorType) (const FObjectInitializer&)
```

For `UClass:Bind()` on blueprint classes

```cpp
/**
 * Find the local constructor of the class. Mainly for blueprint classes.
 */
void UClass::Bind()
{
    // Omitting some code...
    UClass* SuperClass = GetSuperClass();
    if (SuperClass && (ClassConstructor == nullptr || ClassAddReferencedObjects == nullptr
        || ClassVTableHelperCtorCaller == nullptr
        ))
    {
        // Ensure correctness of the binding of the superclass.
        SuperClass->Bind();
        if (!ClassConstructor)
        {   // Use the superclass's constructor as its own.
            ClassConstructor = SuperClass->ClassConstructor;
        }
    }
    // Omitting some code...
    // If it's still not possible, report an error!
    if( !ClassConstructor )
    {
        UE_LOG(LogClass, Fatal, TEXT("Can't find ClassConstructor for class %s"), *GetPathName() );
    }
}
```

Binding of a `UClass` is necessary only when compiling blueprints or loading a class in a `Package`. Native classes already pass a function pointer to their previous `GetPrivateStaticClassBody`. Classes without any C++ code entities require a bound constructor to the base class to correctly inherit functions and call them.

To summarize the macro constructor generation, refer to the diagram below:

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/Constructor Macros.png]]
### UObject Initialization

The UObject Initialization phase begins in `AppInit()` with the following call stack order:

1. **InitUObject**: Sets up initialization and processes the `ProcessNewLoadedUObjects` delegate.
    
2. **StaticUObjectInit**: Calls `UObjectBaseInit` to initialize the garbage collector with a transient package.
    
3. **UObjectBaseInit**: Allocates a global object array pool in memory and confirms that the UObject system is initialized.
    
4. **UObjectProcessRegistrants**: Traverses a pending registrant link list and passes registrants to `UObjectForceRegistration`.
    
5. **UObjectForceRegistration**: Retrieves a Registrant Object and calls `DeferredRegister` with parameters such as `UClass::StaticClass`, `PackageName`, and Name.
### UObject Allocation & Construction

At this stage, the UObject Initialization has completed. The next step is the process of UObject Allocation and Construction via `NewObject`

```cpp
/**  
 * Convert a boot-strap registered class into a real one, add to uobject array, etc * * @param UClassStaticClass Now that it is known, fill in UClass::StaticClass() as the class */
void UObjectBase::DeferredRegister(UClass *UClassStaticClass,const TCHAR* PackageName,const TCHAR* InName)  
{  
    check(UObjectInitialized());  
    // Set object properties.  
    UPackage* Package = CreatePackage(PackageName);  
    check(Package);  
    Package->SetPackageFlags(PKG_CompiledIn);  
    OuterPrivate = Package;  
  
    check(UClassStaticClass);  
    check(!ClassPrivate);  
    ClassPrivate = UClassStaticClass;  
  
    // Add to the global object table.  
    AddObject(FName(InName), EInternalObjectFlags::None);  
    // At this point all compiled-in objects should have already been fully constructed so it's safe to remove the NotFullyConstructed flag  
    // which was set in FUObjectArray::AllocateUObjectIndex (called from AddObject)    GUObjectArray.IndexToObject(InternalIndex)->ClearFlags(EInternalObjectFlags::PendingConstruction);  
  
    // Make sure that objects disregarded for GC are part of root set.  
    check(!GUObjectArray.IsDisregardForGC(this) || GUObjectArray.IndexToObject(InternalIndex)->IsRootSet());  
  
    UE_LOG(LogUObjectBootstrap, Verbose, TEXT("UObjectBase::DeferredRegister %s %s"), PackageName, InName);  
}
```

A call to `NewObject` is made to set an Outer `Package` for an Object Registrant currently being processed.
### NewObject

NewObject is the core function used to create new UObjects in Unreal Engine, replacing `ConstructObject` since UE3. Its influence spans across various areas for objects constructed within the engine:

- Templated objects CDO (Create Default Object, Archetypes).
- Actors and their Components.
- Default Sub Objects.

When `NewObject` is called, it performs the following steps:

- `StaticConstructObject_Internal`
- `StaticAllocateObject`
    - Determines if the Object Exists and reuses the memory location if possible.
    - If not, calls `AllocateUObject`.

Visually, the call stack looks like this:

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/NewObjectCallStack.png]]

##### StaticConstructObject_Internal

```cpp
template< class T >  
T* NewObject(UObject* Outer, FName Name, EObjectFlags Flags = RF_NoFlags, UObject* Template = nullptr, bool bCopyTransientsFromClassDefaults = false, FObjectInstancingGraph* InInstanceGraph = nullptr)  
{  
    if (Name == NAME_None)  
    {       
	    FObjectInitializer::AssertIfInConstructor(Outer, TEXT("NewObject with empty name can't be used to create default subobjects (inside of UObject derived class constructor) as it produces inconsistent object names. Use ObjectInitializer.CreateDefaultSubobject<> instead."));  
    }  
    FStaticConstructObjectParameters Params(T::StaticClass());  
    Params.Outer = Outer;  
    Params.Name = Name;  
    Params.SetFlags = Flags;  
    Params.Template = Template;  
    Params.bCopyTransientsFromClassDefaults = bCopyTransientsFromClassDefaults;  
    Params.InstanceGraph = InInstanceGraph;  
    return static_cast<T*>(StaticConstructObject_Internal(Params));  
}
```

`StaticConstructObject_Internal` handles different UObject types based on their flags, which determine the type or state of the object.

Digging deeper, `StaticAllocateObject` is called by `ClassConstructor`.

```cpp
// Line 4383 UObjectGlobals.cpp
// Call to StaticAllocateObject to Allocate the memory
Result = StaticAllocateObject(InClass, InOuter, InName, InFlags, Params.InternalSetFlags, bCanRecycleSubobjects, &bRecycledSubobject, Params.ExternalPackage);

//.........................................................................

// Line 4389 UObjectGlobals.cpp
// Call to the Class Constructor 

// Don't call the constructor on recycled subobjects, they haven't been destroyed.  
if (!bRecycledSubobject)  
{        
STAT(FScopeCycleCounterUObject ConstructorScope(InClass->GetFName().IsNone() ? nullptr : InClass, GET_STATID(STAT_ConstructObject)));  
    (*InClass->ClassConstructor)(FObjectInitializer(Result, Params));  
}
```

##### StaticAllocateObject

`StaticAllocateObject` allocates a memory block before setting any initial values during construction. Additionally, the function performs several reflection checks or checks based on UObject flags. Before allocation, `StaticFindObjectFastInternal` is called to determine if the object already exists. If the object does exist, reusing the memory block at its address is more efficient.

The main takeaway from reading the source code is how the UObject system utilizes `EObjectFlags`. These flags track the state or type of a allocated UObject.

The process of allocating a base UObject is illustrated in the code snippets below:

```cpp
// UObjectGlobals.cpp
// Line 3357
if( Obj == nullptr )  
{     
	int32 Alignment = FMath::Max( 4, InClass->GetMinAlignment() );  
    Obj = (UObject *)GUObjectAllocator.AllocateUObject(TotalSize,Alignment,GIsInitialLoad);  
}
```

The first call will be to `GUObjectAllocator` to allocate the memory.

```cpp
if (!bSubObject)  
{  
    FMemory::Memzero((void *)Obj, TotalSize);  
    new ((void *)Obj) UObjectBase(const_cast<UClass*>(InClass), InFlags|RF_NeedInitialization, InternalSetFlags, InOuter, InName, OldIndex, OldSerialNumber);  
}
else  
{  
    // Propagate flags to subobjects created in the native constructor.  
    Obj->SetFlags(InFlags);  
    Obj->SetInternalFlags(InternalSetFlags);  
}
```

Interestingly, if the object isn't a sub-object, then `placement new` is applied to the `UObjectBase` portion of the memory layout. Subsequently, flags are propagated to any sub-objects from the native constructor.

#### Summary

The process involves calling `NewObject` twice, each time invoking a different overloaded version of the constructor for `placement new`. From the diagram, the first call to `placement new` finalizes the addition of a new `UObject` to the global array object pool. The second case involves a regular invocation of a parent class constructor during object construction.

This approach mitigates the issue of undefined behavior from destruction calls in two consecutive `placement new` operations.

Moreover, `FObjectInitializer` serves as a helper initializer object created just before the constructor is called. Essentially, after the native constructor is invoked, it's pushed to a thread-local array stack via a `ThreadContext`. Its destruction becomes an integral step in initializing `UObject` data. During this step, attribute fields or sub-objects are initialized based on reflection information. Attributes can also be loaded from configuration files, ensuring comprehensive initialization of an object's state. More details on this will be explored later on.
### Template Object

Before delving deeper into `FObjectInitializer`, it's important to introduce the prototype design pattern. Understanding this pattern will provide context for Unreal Engine's extensive usage of template objects and clarify when objects are instantiated and initialized from these templates.

**Prototype Pattern**

*"The Prototype Pattern in programming is a design approach where objects are copied or cloned rather than created a new. It involves defining a prototype interface or abstract class with a method for cloning, then implementing concrete classes that adhere to this interface. By cloning existing objects, the pattern promotes efficiency, especially when creating instances is resource-intensive. It also fosters flexibility, allowing for dynamic modifications to class structures at runtime. This pattern is particularly useful in scenarios where creating new objects is costly or where object creation needs to be dynamic and customizable."*

Unreal Engine's prototype-like pattern is closely tied to its reflection system. `UClass`, which holds type information at runtime, associates a template object, typically known as a Class Default Object (CDO), with itself. The CDO serves as the object that instances the default state for any new objects of that class.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/TemplateObjects.svg]]
#### Template CDO and Archetypes Flags

A Template Object can manifest in several forms:

1. **Archetype object** (RF_ArchetypeObject).
2. **CDO object** (RF_ClassDefaultObject).
3. **Object instance** with or without RF_ArchetypeObject or RF_ClassDefaultObject flags.

Expressed differently:

- A Template object can serve as both an Archetype and a CDO.
- A Template object can solely function as a CDO.
- A Template object can be an instantiated object, which may also act as an archetype or a CDO.

Another crucial aspect of `NewObject` involves the use of a provided template object:

- `UClass* Class`: Specifies the class of the object to create. The return value of `Class->GetDefaultObject()` is the CDO.
- `UObject* Template`: If specified, the property values from this object will be copied to the new object, and the new object's ObjectArchetype value will be set to this object. If nullptr, the class default object (CDO) is used instead.
- `EObjectFlags InFlags`: Specifies the ObjectFlags to assign to the new object.

The key takeaway is: If a Template Object is specified, it takes precedence over the CDO for copying default property values to newly instantiated objects.

```cpp
FObjectInitializer::~FObjectInitializer()
{
	//...
	UObject* Defaults = ObjectArchetype ? ObjectArchetype : BaseClass->GetDefaultObject(false);
	InitProperties(Obj, BaseClass, Defaults, bCopyTransientsFromClassDefaults);
	//...
}
```

Archetype objects are initialized from `uassets`, which requires understanding the serialization system, specifically `FArchive`. I plan to add a section on this topic later on. An illustrative example of archetype usage in the engine source code involves loading a UObject from disk using the function `CreateExport`.

```cpp
UObject* FLinkerLoad::CreateExport( int32 Index )
{
	FObjectExport& Export = ExportMap[ Index ];
	...
	// RF_ArchetypeObject and other flags
	EObjectFlags ObjectLoadFlags = Export.ObjectFlags;
	...
	Export.Object = StaticConstructObject_Internal
	(
		LoadClass,
		ThisParent,
		NewName,
		ObjectLoadFlags,
		EInternalObjectFlags::None,
		Template
	);
}
```

### Create Default Object

In this section, we delve into the latter stages of Unreal Engine's registration phase, focusing on the construction process of Class Default Objects (CDOs). While the construction nuances differ for native CDOs, the overall process remains consistent for new class CDOs across subsequently loaded modules.

To recap:

`UClassRegisterAllCompiledInClasses` is initially called twice during the CoreUObject module's initialization. This dual invocation is crucial due to the fundamental relationship between UObject, UClass, and the reflection system.

As a key principle:

_"UClass is a UObject and UObject is a UClass."_

The first call to `UClassRegisterAllCompiledInClasses` ensures the setup of the UClass memory layout before Core UObject types are instantiated. The second call occurs within `ProcessNewlyLoadedUObjects`, where the inner registrant function pointer is invoked again to construct actual Core UObject types (such as Structs, UClasses, and UObjects).

In the previous section, `StaticAllocateObject` was discussed during the registration phase, following `UObjectBaseInit`, which initializes the global UObject array pool in memory. Subsequently, `NewObject` is first used to construct an Outer `UPackage` for the UObject.

Continuing through the constructor process, `UObjectForceRegistration` invokes `DeferredRegister` for each pending registrant in the global UObject array. Finally, `Object->DeferredRegister` adds the UObject to a global hash map, where object references are stored with a `FName` key.
#### ProcessNewlyLoadedObjects

The `ProcessNewlyLoadedObjects` function is a delegate set up early in the loading phase of the Engine Launch loop during `LoadCoreModules`. Its purpose is to execute the same UObject construction process for newly loaded modules after the Core UObject system is initialized.

During this phase of the registration process, the Class Default Objects (CDOs) are constructed using the `UObjectLoadAllCombinedInDefaultsProperties` function.

While this holds true for the majority of class objects, I discovered an interesting detail when I inserted a debug breakpoint early inside `UObjectLoadAllCombinedInDefaultProperties`. It revealed that some CoreUObject classes (`UObject`, `GCObjectReference`, `TextBuffer`, `Field`, `Struct`, `ScriptStruct`) already had their CDOs created!

For certain native types like UObject CDOs, they are actually constructed when `NewObject` is initially called to set their outer package.

Following the construction path after the outer package is created by `NewObject`, `GetDefaultObject()` is invoked to forcefully construct a CDO object using an `FObjectInitializer`. The role of `FObjectInitializer` will be explained in more detail later.

Additionally, a CDO instance is identified with a `Default_` prefix in its `FName` to distinguish it from its native UObject. For instance, if a `TestActor` is created in C++, its constructor would only be called after the `TestActor` UClass is fully initialized and its `OuterRegister` function pointer is invoked. This process follows the superclass chain for any derived classes. It can be inferred that if a UClass is derived from a parent class, its CDO would be aware of the parent's CDO.

The CDO represents the final default state when the `TestActor` constructor is invoked. The CDO is referenced in the UClass as the main template object.

`UObjectLoadAllCompiledInDefaultProperties()` categorizes CDO construction into `NewCoreUObject`, `NewEngineClasses`, and `NewClasses` array types, in that specific order.

As noted earlier in memory, I observed that the first few core UObjects processed in `NewCoreUObject` already had their CDOs (`UObject`, `GCObjectReference`, `TextBuffer`, `Field`, `Struct`, `ScriptStruct`) constructed.

```cpp
/**  
 * Get the default object from the class 
 * @param bCreateIfNeeded if true (default) then the CDO is created if it is null 
 * @return the CDO for this class */
UObject* GetDefaultObject(bool bCreateIfNeeded = true) const  
{  
    if (ClassDefaultObject == nullptr && bCreateIfNeeded)  
    {       
	    InternalCreateDefaultObjectWrapper();  
    }  
    return ClassDefaultObject;  
}
```

Subsequently, the code snippet doesn't invoke `InternalCreateDefaultObjectWrapper` for CDOs that have already been allocated. This omission is likely because these already constructed CDOs are necessary for derived classes that might use their parent CDOs as template objects to inherit default values.

Despite this, `UClass` stands out as the first element in the `NewCoreUObject` array that doesn't initially have its CDO constructed. Therefore, `InternalCreateDefaultObjectWrapper` isn't called for `UClass` until it is encountered first in the `NewCoreUObject` array.

After the core engine CDOs are created, packages like those containing your game module's UObjects are also sorted into the `NewClasses` array to have their CDOs constructed.

Further investigation into the Engine source code reveals that `CreateDefaultObject` is invoked in two primary scenarios:

1. **To construct a CDO instance:**
    
    - During the registration phase of the engine.
    - During serialization, where a CDO may need construction for UObjects being deserialized from a package in a deep hierarchy of templated objects.
    - When creating and registering a new CDO, either to override or replace an existing one.
2. **To obtain the CDO instance in memory:**
    
    - To retrieve a CDO and load its default settings.
    - During delta serialization, where a Blueprint class compares its CDO to a native CDO. If changes are detected, it may resistate a new CDO and copy over the native CDO values.
    - In nested `ClassConstructor` calls, where a CDO instance is passed as a parameter with `FObjectInitializer` to override or construct initial values for various objects.

```cpp
/**
 * Get the default object from the class, creating it if missing, if requested or under a few other circumstances
 * @return the CDO for this class
 */
UObject* UClass::CreateDefaultObject()
{
    if (ClassDefaultObject == NULL)
    {
        ensureMsgf(!bLayoutChanging, TEXT("Class named %s creating its CDO while changing its layout"), *GetName());

        UClass* ParentClass = GetSuperClass();
        UObject* ParentDefaultObject = NULL;
        if (ParentClass != NULL)
        {
            UObjectForceRegistration(ParentClass);
            ParentDefaultObject = ParentClass->GetDefaultObject(); // Force the default object to be constructed if it isn't already
            check(GConfig);
            if (GEventDrivenLoaderEnabled && EVENT_DRIVEN_ASYNC_LOAD_ACTIVE_AT_RUNTIME)
            {
                check(ParentDefaultObject && !ParentDefaultObject->HasAnyFlags(RF_NeedLoad));
            }
        }
        if ((ParentDefaultObject != NULL) || (this == UObject::StaticClass()))
        {
            // If this is a class that can be regenerated, it is potentially not completely loaded.
            // Preload and Link here to ensure we properly zero memory and read in properties for the CDO
            if (HasAnyClassFlags(CLASS_CompiledFromBlueprint) && (PropertyLink == NULL) && !GIsDuplicatingClassForReinstancing)
            {
                auto ClassLinker = GetLinker();
                if (ClassLinker)
                {
                    if (!GEventDrivenLoaderEnabled)
                    {
                        UField* FieldIt = Children;
                        while (FieldIt && (FieldIt->GetOuter() == this))
                        {
                            // If we've had cyclic dependencies between classes here, we might need to preload to ensure that we load the rest of the property chain
                            if (FieldIt->HasAnyFlags(RF_NeedLoad))
                            {
                                ClassLinker->Preload(FieldIt);
                            }
                            FieldIt = FieldIt->Next;
                        }
                    }
                    StaticLink(true);
                }
            }
            // in the case of cyclic dependencies, the above Preload() calls could end up
            // invoking this method themselves... that means that once we're done with
            // all the Preload() calls we have to make sure ClassDefaultObject is still
            // NULL (so we don't invalidate one that has already been setup)
            if (ClassDefaultObject == NULL)
            {
                // RF_ArchetypeObject flag is often redundant to RF_ClassDefaultObject, but we need to tag
                // the CDO as RF_ArchetypeObject in order to propagate that flag to any default sub objects.
                ClassDefaultObject = StaticAllocateObject(this, GetOuter(), NAME_None, EObjectFlags(RF_Public|RF_ClassDefaultObject|RF_ArchetypeObject));
                check(ClassDefaultObject);
                // Register the offsets of any sparse delegates this class introduces with the sparse delegate storage
                for (TFieldIterator<FMulticastSparseDelegateProperty> SparseDelegateIt(this, EFieldIteratorFlags::ExcludeSuper, EFieldIteratorFlags::ExcludeDeprecated); SparseDelegateIt; ++SparseDelegateIt)
                {
                    const FSparseDelegate& SparseDelegate = SparseDelegateIt->GetPropertyValue_InContainer(ClassDefaultObject);
                    USparseDelegateFunction* SparseDelegateFunction = CastChecked<USparseDelegateFunction>(SparseDelegateIt->SignatureFunction);
                    FSparseDelegateStorage::RegisterDelegateOffset(ClassDefaultObject, SparseDelegateFunction->DelegateName, (size_t)&SparseDelegate - (size_t)ClassDefaultObject.Get());
                }
                EObjectInitializerOptions InitOptions = EObjectInitializerOptions::None;
                if (!HasAnyClassFlags(CLASS_Native | CLASS_Intrinsic))
                {
                    // Blueprint CDOs have their properties always initialized.
                    InitOptions |= EObjectInitializerOptions::InitializeProperties;
                }
                (*ClassConstructor)(FObjectInitializer(ClassDefaultObject, ParentDefaultObject, InitOptions));
                if (GetOutermost()->HasAnyPackageFlags(PKG_CompiledIn) && !GetOutermost()->HasAnyPackageFlags(PKG_RuntimeGenerated))
                {
                    TCHAR PackageName[FName::StringBufferSize];
                    TCHAR CDOName[FName::StringBufferSize];
                    GetOutermost()->GetFName().ToString(PackageName);
                    GetDefaultObjectName().ToString(CDOName);
                    NotifyRegistrationEvent(PackageName, CDOName, ENotifyRegistrationType::NRT_ClassCDO, ENotifyRegistrationPhase::NRP_Finished, nullptr, false, ClassDefaultObject);
                }
                ClassDefaultObject->PostCDOContruct();
            }
        }
    }
    return ClassDefaultObject;
}

```

The `CreateDefaultObject` function follows a straightforward process, summarized in several steps:

1. If the Class Default Object (CDO) of this UClass exists (`CDO` is not null), return its reference immediately.
    
2. If a parent superclass exists, ensure its registration if it's being loaded asynchronously. Obtain its CDO and set its state to `RF_NeedLoad`.
    
3. If either the parent object exists or if we are creating the CDO for a UObject for the first time:
    
4. If the object is being compiled from a blueprint, use the `ClassLinker` to preload `UField`s and then invoke `StaticLink` on the class. Note: `UField` is still used with blueprints despite `FField` being phased in. `StaticLink` is crucial as it's called at the end of the registration phase. Its purpose is to utilize a `Dummy Archive` object to access functionalities defined in `FArchive`. This process resolves memory address changes after structural modifications to `UClass`.
    
5. If the CDO is null, proceed to:
    
    - Set up the object's flags.
    - Allocate its memory block.
    - Finally, call its constructor `(*ClassConstructor)(FObjectInitializer(ClassDefaultObject, ParentDefaultObject, InitOptions))`.
#### Summary

The Class Default Object is instantiated when the default constructor is called, representing the final default state of a specific `UClass`. It serves as a prototype object used to instantiate objects at runtime. This approach serves as a workaround for not having a direct way to invoke a native C++ constructor using a function pointer.

Understanding the distinction between regular C++ object-oriented programming and Unreal Engine's approach should now be clearer. For new programmers, grasping this core concept early on can be more beneficial than navigating through basic C++ experience alone. While tutorials can provide foundational knowledge for building games with Unreal Engine and C++, there are occasions where implementing a new gameplay system specific to a game's design may be necessary.

In such cases, if the desired system isn't readily available in the engine, developers may find themselves delving into the intricacies of the core UObject system. Along this journey, developers may encounter several "invisible guardrails." These guardrails are critical aspects of the engine's architecture that, when not clearly understood, can lead to challenges or frustration. Without awareness of these guardrails, developers might unknowingly face obstacles or resort to less optimal solutions that could become problematic with future engine updates.

While hacky implementations may temporarily suffice, they are often susceptible to issues if not aligned with subsequent engine updates. Therefore, maintaining clarity and understanding of the underlying UObject system ensures more robust and future-proofed implementations.

--------------------------------------------------------------------
### FObjectInitalizer

The goal of this section is to explore the purpose and use of `FObjectInitializer`. The `FObjectInitializer` is a helper object designed to address various object creation challenges that arise from deep inheritance and nested structures. The intent is to separate the construction process from the complex object and its representation. This is known in design patterns as the Builder Pattern, which provides flexibility in using the same construction process while supporting different representations.

Unreal Engine follows what appears to be a builder pattern with `FObjectInitializer`. The object is passed around default constructors to modify initialization values of derived classes or sub-objects within the class. An example is mentioned in the Engine source code comments of `Object.h`.

```cpp
/**   
 *  Constructor that takes an ObjectInitializer.   
 *  Typically not needed, but can be useful for class hierarchies that support  
 *  optional subobjects or subobject class overriding */
COREUOBJECT_API 
UObject(const FObjectInitializer& ObjectInitializer);
```

Following the constructor path with `FObjectInitializer`, its optional parameter is created as a temporary object to encapsulate construction initialization objects in the `Super` constructor call chain.

In `NoExportTypes.h`, a file where base declarations of native objects are defined, the instantiation of `FObjectInitializer` for `UObject` is shown below:

```cpp
UObject(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
```

`FObjectInitalizer` calls a getter function to a singleton object called the `ThreadContext`.

```cpp
FObjectInitializer& FObjectInitializer::Get()  
{  
    FUObjectThreadContext& ThreadContext = FUObjectThreadContext::Get();  
    UE_CLOG(!ThreadContext.IsInConstructor, LogUObjectGlobals, Fatal, TEXT("FObjectInitializer::Get() can only be used inside of UObject-derived class constructor."));  
    return ThreadContext.TopInitializerChecked();  
}
```

However, it is fatal to call this constructor using an `FObjectInitializer` for a base UObject. The logical design of `FObjectInitializer` ends here as an initialization helper, hence the fatal remark. Additionally, at the gameplay level, the base class `FObjectInitializer` should stop at `AActor`, as stated in engine source code comments.

The main use of `FObjectInitializer` is for post-initialization work after calling the default constructor of a class or sub-object.

```cpp
(*ClassConstructor)(FObjectInitializer(ClassDefaultObject, ParentDefaultObject, InitOptions));
```

```cpp
FObjectInitializer::FObjectInitializer(UObject* InObj, UObject* InObjectArchetype, EObjectInitializerOptions InOptions, struct FObjectInstancingGraph* InInstanceGraph)  
    : Obj(InObj)  
    , ObjectArchetype(InObjectArchetype)  
      // if the SubobjectRoot NULL, then we want to copy the transients from the template, otherwise we are doing a duplicate and we want to copy the transients from the class defaults  
    , bCopyTransientsFromClassDefaults(!!(InOptions & EObjectInitializerOptions::CopyTransientsFromClassDefaults))  
    , bShouldInitializePropsFromArchetype(!!(InOptions & EObjectInitializerOptions::InitializeProperties))  
    , InstanceGraph(InInstanceGraph)  
    , PropertyInitCallback([](){})  
{  
    Construct_Internal();  
}
```

This `FObjectInitializer` constructor has several different function parameters used to handle various object construction scenarios, which will be explored in more detail later on. However, for now, let's briefly mention the main ones:

- `Obj` - Copies the pointer to the object we're overriding, in this case, it will be the CDO object. This is set to null at the end of its use to prevent any undefined behavior with the actual CDO.
- `ObjectArchetype` - Allows overriding the CDO and providing a custom template object.
- `InstanceGraph` - A helper data structure that constructs an instantiation relationship graph for sub-objects and their default template objects.
- `PropertyInitCallback` - An optional function object where a lambda can be passed to execute additional custom initialization logic just before the `PostInitProperties` delegate is called.

`Construct_Internal` is a helper function that creates a `ThreadContext`, as seen in the previous example with `UObject`.

```cpp
void FObjectInitializer::Construct_Internal()  
{  
    FUObjectThreadContext& ThreadContext = FUObjectThreadContext::Get();  
    // Mark we're in the constructor now.  
    ThreadContext.IsInConstructor++;  
    LastConstructedObject = ThreadContext.ConstructedObject;  
    ThreadContext.ConstructedObject = Obj;  
    ThreadContext.PushInitializer(this);  
  
    if (Obj && GetAllowNativeComponentClassOverrides())  
    {       
	    Obj->GetClass()->SetupObjectInitializer(*this);  
    }
```

#### Thread Context

A thread context is an intermediary singleton object used to push `FObjectInitializers` into a stack array allocated within Thread Local Storage (TLS). Thread local storage is created using the Windows API. Its purpose is to allow programmers to allocate separate local memory spaces for individual threads in a multithreaded environment. This is another design pattern known as the `Thread Local Storage Design Pattern`.
![[Unreal Engine UObject Construction & Post Initalization/Diagrams/TLSExample.svg]]
The concept is fairly simple. Imagine two situations side by side in a diagram: on the left-hand side, threads 1-3 have their own local methods and variables that write to a single global memory space. Since the memory space is shared among those threads, it obviously creates a race condition. To solve this issue, the thread local storage design pattern is employed. Since C++11, `thread_local` has been a keyword storage class specifier. For thread-local types, initialization and destruction occur within the thread's lifetime.

Thread local storage isolates its own global variables specific to each individual thread, operating on its own copy of the global variable. This reduces processing overhead and eliminates the need for a mutex to lock a single global memory pool across multiple threaded operations.

Each thread stores TLS data in a slot of a TLS array. Before usage, each index must be allocated by that thread. Typically, threads store their data directly in the TLS slot. However, for larger datasets, separate storage may be allocated to minimize TLS slot usage.

From the Microsoft documentation the image illustrates this below.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/ThreadContextMicrosoft.png]]

##### FUObjectThreadContext

When dealing with the creation and processing of a large number of new `UObjects`, efficiency becomes important. In a multi-threaded environment, optimizing the construction of CDOs using the reflection system can significantly enhance performance. It's important to note that CDOs typically do not contain complex game logic by design, making them generally thread-safe to construct across multiple threads.

```cpp
class COREUOBJECT_API FUObjectThreadContext : public TThreadSingleton<FUObjectThreadContext>  
{  
    friend TThreadSingleton<FUObjectThreadContext>;  
  
    FUObjectThreadContext();  
    virtual ~FUObjectThreadContext();  
  
    /** Stack of currently used FObjectInitializers for this thread */  
    TArray<FObjectInitializer*> InitializerStack;
    ....
}
```

`FUObjectThreadContext` is a templated `TThreadSingleton` where a `friend` relationship defines the type of the singleton context.

```cpp
/**  
 *  @return an instance of a singleton for the current thread. */
FORCEINLINE static T& Get()  
{  
    return *(T*)FThreadSingletonInitializer::Get( [](){ return (FTlsAutoCleanup*)new T(); }, T::GetTlsSlot() ); //-V572  
}
```

The lambda function `CreateInstance` is passed to a getter to return the instance of the actual singleton that points to the `TLS slot` holding a specific thread context. 

```cpp
/**  
 *  @param CreateInstance Function to call when a new instance must be created. *  @return an instance of a singleton for the current thread. */
FORCEINLINE static T& Get(TFunctionRef<FTlsAutoCleanup*()> CreateInstance)  
{  
    return *(T*)FThreadSingletonInitializer::Get(CreateInstance, T::GetTlsSlot()); //-V572  
}
```

`FTlsAutoCleanup` is an additional wrapper class that handles clean up functionality using a unique pointer singleton to the thread's local memory resource.

```cpp
/*-----------------------------------------------------------------------------  
    FThreadSingletonInitializer-----------------------------------------------------------------------------*/  
FTlsAutoCleanup* FThreadSingletonInitializer::Get(TFunctionRef<FTlsAutoCleanup*()> CreateInstance, uint32& InOutTlsSlot)
{
    uint32 TlsSlot;
    UE_AUTORTFM_OPEN(
    {
        TlsSlot = (uint32)FPlatformAtomics::AtomicRead_Relaxed((int32*)&InOutTlsSlot);
        if (TlsSlot == FPlatformTLS::InvalidTlsSlot)
        {
            const uint32 ThisTlsSlot = FPlatformTLS::AllocTlsSlot();
            check(FPlatformTLS::IsValidTlsSlot(ThisTlsSlot));
            const uint32 PrevTlsSlot = FPlatformAtomics::InterlockedCompareExchange((int32*)&InOutTlsSlot, (int32)ThisTlsSlot, FPlatformTLS::InvalidTlsSlot);
            if (PrevTlsSlot != FPlatformTLS::InvalidTlsSlot)
            {
                FPlatformTLS::FreeTlsSlot(ThisTlsSlot);
                TlsSlot = PrevTlsSlot;
            }
            else
            {
                TlsSlot = ThisTlsSlot;
            }
        }
    });
    FTlsAutoCleanup* ThreadSingleton = (FTlsAutoCleanup*)FPlatformTLS::GetTlsValue(TlsSlot);
    if (!ThreadSingleton)
    {
	    // Here the Lambda function creates the singleton!
        ThreadSingleton = CreateInstance();
        // Register() is line where thread_local is declared 
        // for the array in the diagram we saw in the microsoft doc
        ThreadSingleton->Register();
        FPlatformTLS::SetTlsValue(TlsSlot, ThreadSingleton);
    }
    return ThreadSingleton;
}

```

In the core implementation, `FThreadSingletonInitializer` manages the retrieval and allocation of thread-local storage using Windows API-level functions. The `CreateInstance` method is pivotal here, responsible for allocating a context using the `new` operator. The registration function uses the `thread_local` storage class specifier to define an array of unique pointers to instances of thread context.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/FObjectInitalizer.svg]]
The diagram below illustrates the post-initialization process using `FObjectInitializer`. Consider a class hierarchy from A to C, with a sub-object created inside the Class C constructor. A similar hierarchy exists for the sub-object.

1. The constructor of Class A is invoked first, creating a temporary `FObjectInitializer`.
2. The `FObjectInitializer` constructor calls `Construct_Internal`.
3. Inside `Construct_Internal`, the logic retrieves `FUObjectThreadContext` and pushes itself onto the thread-local storage `InitializerStack` array.
4. Steps 1-3 are repeated recursively until the constructor of the most derived Class (Class C) is called.
5. Upon exiting the scope of the constructor of Class C, the `FObjectInitializer` goes out of scope, triggering its destructor.
6. During destruction, each class in the hierarchy is recursively called upwards, executing post-initialization work. `FUObjectThreadContext` is again accessed to pop itself from the `InitializerStack` array in the thread-local storage.
#### Summary

`FObjectInitializer` is a temporary object scoped within a constructor, primarily used for performing post-initialization operations on properties and sub-objects of a UObject.

The main use cases are:

1. **CDO or Archetype Copying**: It facilitates copying properties from a Class Default Object (CDO) or an Archetype to a new instance.
    
2. **NewObject Creation or Template Object Overriding**: When creating a new instance using NewObject, it allows overriding properties with a specified template object.
    
3. **Reading or Extracting Configuration Values**: It handles reading or extracting configuration values for fields marked (CPF_Config) during the startup phase.
    

As discussed, `FObjectInitializer` operates in two primary construction scenarios. Firstly, as a temporary object instantiated with a default constructor when `NewObject` is called. After the constructor completes, it initializes the attribute fields or sub-objects of a UObject based on the reflection information defined in its UClass.

Secondly, it is used for additional property initialization tasks such as copying properties from a UObject's CDO or loading attributes from configuration Ini files via `FConfigInitializer`.

The post-initialization process is intricately tied to the lifecycle of FObjectInitializer. Upon calling `Construct_Internal` within the default constructor of a superclass hierarchy, the object instance is cached. As the scope exits from the most derived class, the destructor of FObjectInitializer is recursively invoked, executing the initialization logic. This mechanism is efficiently managed using a `ThreadContext` singleton, which utilizes a TLS Slot to store FObjectInitializers in a ThreadLocal InitializerStack.
#### Intermission Review

Before delving into the Post Construction Initialization process, there are several concepts I needed to grasp for a better understanding of how sub-objects and properties are instantiated and initialized. Additionally, topics such as Serialization and the usage of FProperty will be discussed.

Let's quickly review what has been covered so far to recall the concepts and their relevance to the upcoming discussion on post-constructor initialization:

**Reflection**: UHT generates and inserts code defined by macros to gather type information and construct a UClass. During post-initialization, UClass's FProperties are traversed for initialization, overwriting, or reading.

**UObject Architecture**: During the registration phase, we observed the construction process of native types and how UObjects are statically allocated in memory and registered into a global UObject memory pool.

**NewObject**: Once the UObject system is initialized, NewObject becomes available for instantiating new objects. This is where the default constructor is called, and where FObjectInitializer may first be used.

**Template Objects**: Template objects follow a prototype design pattern to instantiate objects (via Archetypes or CDOs). These objects determine default values of properties or sub-objects. The InstanceGraph loads template objects into a graph structure and maps them to their instantiated objects, enabling modification or copying.

**Create Default Object**: Towards the end of the registration phase, ProcessNewlyLoadedObjects creates CDOs for all UObjects within the loaded module. The CDO is associated with its UClass and defines the default state from which objects are instantiated. The CDO often serves as the template object during post-initialization.

**FObjectInitializer**: Following a builder design pattern, FObjectInitializer provides flexibility in the engine's construction process while separating the representation of the object being constructed. FObjectInitializer is pushed to a thread-local storage stack array for every constructor call. Upon leaving scope from the most derived class, the destructor recursively pops the initializer from the stack array and executes post-initialization work. Depending on the FObjectInitializer function signature, different post-initialization behaviors can be observed.
### Serialization

During my exploration of Unreal Engine's reflection system, I encountered the intricacies of its serialization system, which initially posed challenges to understand. To prioritize clarity, I deferred deep exploration until the need arose.

Understanding Unreal's serialization system is crucial for comprehending various aspects of the construction and post-initialization processes. While navigating the code can initially be daunting, I found valuable resources such as articles and talks from Unreal Fest 2023. These resources provided insights that I've distilled into this section, aiming to consolidate key concepts.

For a deeper analysis, I recommend referring to the original sources cited at the end of this section.

#### What is Serialization?

The technical computer science term refers to a data processing technique used to store an object's state into a data structure. This structure can then be converted into a format accessible in various forms.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/SerializationProcess.svg]]
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

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/TPSPT1svg.svg]]
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

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/TPSDeserialize.svg]]
In summary, the PropertyTag system makes data more flexible to modification however the trade off introduces more overhead when loading data into memory.
##### UnversionedPropertySerialziation

As mentioned earlier, the UPS system is an optional method during cooking. This implementation method is a more performant option. However, UPS follows a strict sequence rule when data is collected from `UClass`. If the sequence is not maintained for serialization or deserialization, then the stored data is considered invalid.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UPSDeserialize.svg]]
After FProperties are serialized, UPS creates a header based on the data serialized.
![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UPSHeader.svg]]
`Deserialization` is again the reverse operation. The generated header data is read from and loaded into the object's memory in the correct order.

One must be careful using UPS because it's more error prone when the object compiled in the editor and the data structure of the asset file after cooking change!

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UPSError.svg]]
#### FArchive

Unreal Archive system is very powerful and `FArchive` is the main driver for all things related to data serialization in the engine. The UML example below involves `FLinkerLoad` and `FLinkerSave`.  

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/FArchiveUML.png]]

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

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UObjectSerialization.svg]]
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

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/SertializedExample.png]]

In Unreal Engine C++, serializing different base types, member fields, or sub-objects with pointer references to different types can be more complex than with native C++ types alone. While the tofu analogy works well for serializing basic native types, custom types require a different approach.

Unreal Engine's solution is deeply rooted in the visitor pattern. Within this pattern, a concept of "self-serialization" emerges. Each custom type defines its own serialization function, allowing the process to unfold recursively, akin to navigating a tree structure. When encountering a "custom slice" in the byte array, it signals the presence of a custom type where the serialization format must be explicitly defined.

Applying these concepts to Unreal Engine's deserialization method reveals a two-step process:

1. When loading the class information of an object, data from different segments of the tofu (byte array) provides insights into the structure of those segments. These segments represent different types and collectively form a blueprint or roadmap for the serialized data.
    
2. Using this blueprint and adhering to the slicing rules, each segment of the tofu is carefully encapsulated during serialization. This ensures that the deserialization process accurately reconstructs the original packing arrangement, preserving data integrity and structure.
#### uasset

Below is a refined version with improved grammar:

The following describes the basic structure of a uasset file, which you can verify using a hex editor. For example, I created a basic actor blueprint and uploaded the uasset file to a hex editor website called `hexed.it`.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/uasset.svg]]

**Header Information:**

- File Summary: Contains summarized data about the uasset.
- Name Table: Stores a list of names for objects within the package.
- Import Table: Defines virtual paths and type information for objects imported from other packages.
- Export Table: Stores virtual paths and type information for objects defined within this package.
- GUID: Unique identifiers used for tracking and identifying objects in the import/export maps.
- Exported Objects: Contains the actual data for the objects defined in the export table.

Next, I'll illustrate how the uasset system works using a visual example extracted from the article.
![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UassetObjectExample.svg]]
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

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/UassetObjectPointeFixup.svg]]


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
### FProperty

`FProperty` is a reflection type used in Unreal Engine for properties defined in a class or struct marked with a `UPROPERTY` macro. The core implementation of `FProperty` is crucial for blueprints, serialization, garbage collection, and construction.

The reflection document describes in more detail how properties are generated, collected, and registered by the engine's reflection system, which won't be covered here.

Since `UClass` holds the `FProperty` fields of a specific object, the next step is to investigate several questions:

1. What is the difference between `FProperty` and `UProperty`?
2. What is the architecture of `FProperty`?
3. What are the different `FProperty` types, and how are they constructed?

##### UProperty vs FProperty

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/FPropertyDependency.svg]]

In the early days of Unreal Engine, `UProperty` was tightly coupled with `UObject`. However, `UProperty` has since become completely legacy and is no longer used. Despite this, if encountered, the engine converts `UProperty` into `FProperty` during serialization.

Epic engineers likely realized before version 4.25 that the original tight coupling between `UObject` and `UProperty` would lead to increased overhead at scale. Therefore, it was redesigned into `FProperty` to eliminate the performance bottlenecks of the original design. `FProperty` functions similarly to `UProperty`, with largely the same code, but with the 'F' prefix denoting it's not tied to a `UObject`. Since the base class `FField` is decoupled from `UObject`, what are the performance gains of this design?

1. Blueprints load much faster due to this design.
2. Garbage collection speed increases because there is a reduction in the number of `UObjects`.
3. Memory footprint is reduced because `UProperty` no longer carries additional logic and data specific to `UObjects`.
4. Iterating through `UObjects` becomes faster, and iterating through `FProperty` speeds up to approximately 2x faster than with `UProperty`.
5. Casting using `FProperty` is approximately 3x faster compared to the original `UObject` design.
#### FProperty Architecture

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/FFieldRelationship.svg]]
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
![[Unreal Engine UObject Construction & Post Initalization/Diagrams/FPropertyLayout.svg]]
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

he property type can be inferred from the additional struct member. Since a struct's memory is designed to be contiguous, the `Offset` difference between `FPropertyParamsBase` and `FGenericPropertyParams` enables type casting. Understanding this, during the registration phase, `ConstructFProperty` recursively processes all `FProperty` types from the `Props` array. The `EPropertyGenFlag` is identified and inserted as the template parameter to construct the correct `FProperty`. This design follows a form of Template Meta Programming, which I am not entirely familiar with.
#### Summary

To summarize, `FProperty` represents a more efficient alternative for the reflection system compared to `UProperty`. Since the decoupling from `UObject`, the concepts of `GetOuter` and `Cast` have been redefined. Notably, `FProperty` is not garbage collected; instead, its allocation and deallocation are managed by `UStruct`. The base classes `FField` and `FFieldClass` collaborate to establish a linked list traversal system for reflected property types.

In scenarios involving legacy code, `UProperty` is often converted to `FProperty` using a `FFieldVariant` container object. `FFieldVariant` also references the owner of an `FField`. During the final stages of registration, `Linking` of a `UStruct` occurs to resolve symbol addresses and assign additional `FProperty` pointers. These `FProperty` pointers facilitate not only property traversal but also identify Class Default Objects (CDOs) during construction and manage attributes necessary for destruction.

With this understanding, the `PostInitialization` process of various `FProperty` types can be comprehended more thoroughly in the concluding sections of `FObjectInitializer`.
### Post Construction Initialization

To delve deeper into the post-initialization process within `FObjectInitializer`, it's essential to understand how it operates within the destructor just before being popped off the `InitializerStack`. This phase heavily relies on the reflection system to traverse and locate `FProperties`, utilizing the serialization system to establish or override default properties in various contexts.

Previously, the sections on `FProperty` and `Serialization` were introduced to provide foundational knowledge necessary for exploring the final segments of `FObjectInitializer`. These remaining sections are crucial for a comprehensive understanding:

- **PostConstructInit**: This likely involves initializing properties after construction, possibly involving setting default values or performing additional setup steps.
    
- **InitProperties**: This phase probably handles the initialization of core properties defined within the object's class.
    
- **InitSubobjectProperties**: Deals with initializing properties of subobjects (components, child objects, etc.) that are part of the main object.
    
- **FObjectInstancingGraph**: This term may refer to the overall structure or graph that manages the instantiation and initialization of objects within the engine.
    

Throughout my research, articles from `DarkFlameMasters` have provided valuable insights and references, particularly in understanding the intricate workings of the source code. These articles serve as a useful guide in deciphering the complexities involved in these final sections of `FObjectInitializer`.
#### PostConstructInit

`PostConstructInit` is the starting point for the post-initialization of properties. Specifically, it's a member function of `FObjectInitializer`. Below are the more relevant members participating inside `FObjectInitializer`.

```cpp
private:
/** Little helper struct to manage overrides from derived classes **/
struct FOverrides
{
	/** Add an override, make sure it is legal **/
	COREUOBJECT_API void Add(FName InComponentName, const UClass* InComponentClass, const TArrayView<const FName>* FullPath = nullptr);

	/** Add a potentially nested override, make sure it is legal **/
	COREUOBJECT_API void Add(FStringView InComponentPath, const UClass* InComponentClass);

	/** Add a potentially nested override, make sure it is legal **/
	COREUOBJECT_API void Add(TArrayView<const FName> InComponentPath, const UClass* InComponentClass, const TArrayView<const FName>* FullPath = nullptr);

	struct FOverrideDetails
	{
		const UClass* Class = nullptr;
		FOverrides* SubOverrides = nullptr;
	};

	/** Retrieve an override, or TClassToConstructByDefault::StaticClass or nullptr if this was removed by a derived class **/
	FOverrideDetails Get(FName InComponentName, const UClass* ReturnType, const UClass* ClassToConstructByDefault, bool bOptional) const;

private:
	static bool IsLegalOverride(const UClass* DerivedComponentClass, const UClass* BaseComponentClass);

	/** Search for an override **/
	int32 Find(FName InComponentName) const
	{
		for (int32 Index = 0; Index < Overrides.Num(); Index++)
		{
			if (Overrides[Index].ComponentName == InComponentName)
			{
				return Index;
			}
		}
		return INDEX_NONE;
	}

	/** Element of the override array **/
	struct FOverride
	{
		FName ComponentName;
		const UClass* ComponentClass = nullptr;
		TUniquePtr<FOverrides> SubOverrides;
		bool bDoNotCreate = false;

		FOverride(FName InComponentName)
			: ComponentName(InComponentName)
		{}

		FOverride& operator=(const FOverride& Other)
		{
			ComponentName = Other.ComponentName;
			ComponentClass = Other.ComponentClass;
			SubOverrides = (Other.SubOverrides ? MakeUnique<FOverrides>(*Other.SubOverrides) : nullptr);
			bDoNotCreate = Other.bDoNotCreate;
			return *this;
		}

		FOverride(const FOverride& Other)
		{
			*this = Other;
		}

		FOverride(FOverride&&) = default;
		FOverride& operator=(FOverride&&) = default;
	};

	/** The override array **/
	TArray<FOverride, TInlineAllocator<8>> Overrides;
};

/** Little helper struct to manage overrides from derived classes **/
struct FSubobjectsToInit
{
	/** Add a subobject **/
	void Add(UObject* Subobject, UObject* Template)
	{
		for (int32 Index = 0; Index < SubobjectInits.Num(); Index++)
		{
			check(SubobjectInits[Index].Subobject != Subobject);
		}
		SubobjectInits.Emplace(Subobject, Template);
	}

	/** Element of the SubobjectInits array **/
	struct FSubobjectInit
	{
		UObject* Subobject;
		UObject* Template;

		FSubobjectInit(UObject* InSubobject, UObject* InTemplate)
			: Subobject(InSubobject)
			, Template(InTemplate)
		{}
	};

	/** The SubobjectInits array **/
	TArray<FSubobjectInit, TInlineAllocator<8>> SubobjectInits;
};

    /** object to initialize, from static allocate object, after construction **/
    UObject* Obj;
    /** object to copy properties from **/
    UObject* ObjectArchetype;
    /** if true, copy the transients from the DefaultsClass defaults, otherwise copy the transients from DefaultData **/
    bool bCopyTransientsFromClassDefaults;
    /** If true, initialize the properties **/
    bool bShouldInitializePropsFromArchetype;
    /** Only true until ObjectInitializer has not reached the base UObject class **/
    bool bSubobjectClassInitializationAllowed = true;
    /** Instance graph **/
    struct FObjectInstancingGraph* InstanceGraph;
    /** List of component classes to override from derived classes **/
    mutable FOverrides SubobjectOverrides;
    /** List of component classes to initialize after the C++ constructors **/
    mutable FSubobjectsToInit ComponentInits;

#if !UE_BUILD_SHIPPING
    /** List of all subobject names constructed for this object **/
    mutable TArray<FName, TInlineAllocator<8>> ConstructedSubobjects;
#endif

    /** Previously constructed object in the callstack **/
    UObject* LastConstructedObject = nullptr;

    /** Callback for custom property initialization before PostInitProperties gets called **/
    TFunction<void()> PropertyInitCallback;

    friend struct FStaticConstructObjectParameters;
};

```

The header code from `UObjectGlobals.h` primarily focuses on the member fields and helper structs used for `Subobjects`. The best way to understand the `Sub Object` initialization process is by examining `AActor` and `ActorComponent`, where an object is defined as a `SubObject` if its direct `Outer` isn't a `UPackage`. Therefore, a `Sub Object` can take on many different types.

In `FObjectInitializer`, the `ComponentInits` array is crucial for the post-initialization of subobjects. During my debugging exploration, `ComponentInits` appears to be populated when the Blueprint VM calls its constructor and subsequently invokes `CreateDefaultSubobject`.

```cpp
// CreateDefaultSubObject.cpp

Result = StaticConstructObject_Internal(Params);

if (Params.Template)
{
    ComponentInits.Add(Result, Params.Template);
}
else if (!bIsTransient && Outer->GetArchetype()->IsInBlueprint())
{
    UObject* MaybeTemplate = Result->GetArchetype();
    if (MaybeTemplate && Template != MaybeTemplate && MaybeTemplate->IsA(ReturnType))
    {
        ComponentInits.Add(Result, MaybeTemplate);
    }
}

if (Outer->HasAnyFlags(RF_ClassDefaultObject) && Outer->GetClass()->GetSuperClass())
{
    #if WITH_EDITOR
        // Default subobjects on the CDO should be transactional, so that we can undo/redo changes made to those objects.
        // One current example of this is editing natively defined components in the Blueprint Editor.
        Result->SetFlags(RF_Transactional);
    #endif
    Outer->GetClass()->AddDefaultSubobject(Result, ReturnType);
}
```

There are three distinct scenarios at play here:

1. If the `SubObjectOverride` struct contains a template object set in the `FObjectInitializer` constructor with `StaticConstructParams` passed, and if an `Archetype` is assigned, then that archetype is used for copying.
    
2. Blueprints parented to a native class undergo construction during deserialization. When `CreateDefaultSubobject` is invoked from a Blueprint-generated object, it checks if the `Outer` is an `Archetype`. Subsequently, `GetArchetype` recursively searches for the specific archetype and verifies if `MaybeTemplate` matches the archetype used for the override.
    
3. The last scenario occurs during a sub-object check. This happens when `CreateDefaultSubobject` is called within the native CDO constructor. It's important to distinguish this from `AddDefaultSubobject`, which refers to the default sub-object added in the global `UObject` hash map. This process occurs within `StaticConstructObject_Internal`, and the subobject is marked as `RF_Transactional`.
    

One might question why `CreateDefaultSubobject` handles post-initialization work instead of the reflection system. Theoretically, the reflection system could populate `ComponentInits` when `UECodeGen_Private::Construct` is called during the registration phase.

However, the responsibility for initialization lies with the user to call `CreateDefaultSubobject` in the default constructor. This approach supports a more deferred initialization strategy to minimize unnecessary memory allocations by the engine. For instance, a gameplay programmer might choose to create components only when the game is running in the editor.

`CreateDefaultSubobject` can initiate complex nested call chains starting from `UObject::CreateDefaultSubobject` to `FObjectInitializer::CreateDefaultSubobject()`. This nested constructor calling is a common occurrence in the lifecycle of the `AActor` and `Component` system.

As observed previously, `FObjectInitializer` interfaces with `UObject`, which will be further explored later on, particularly how `FObjectInitializer` can override or replace a component.
![[Unreal Engine UObject Construction & Post Initalization/Diagrams/PostConstructInitSequence.svg]]
The source of `PostConstructInit` is provided below with comments describing some of the logic. Additionally, this code is summarized at the end in case changes may happen in the future, as learning the logic is more valuable in the long run.

```cpp
void FObjectInitializer::PostConstructInit()
{
    // we clear the Obj pointer at the end of this function, so if it is null
    // then it most likely means that this is being ran for a second time
    if (Obj == nullptr)
    {
#if USE_DEFERRED_DEPENDENCY_CHECK_VERIFICATION_TESTS
        checkf(Obj != nullptr, TEXT("Looks like you're attempting to run FObjectInitializer::PostConstructInit() twice, and that should never happen."));
#endif // USE_DEFERRED_DEPENDENCY_CHECK_VERIFICATION_TESTS
        return;
    }

    SCOPE_CYCLE_COUNTER(STAT_PostConstructInitializeProperties);
    const bool bIsCDO = Obj->HasAnyFlags(RF_ClassDefaultObject);
    UClass* Class = Obj->GetClass();
    UClass* SuperClass = Class->GetSuperClass();

#if USE_CIRCULAR_DEPENDENCY_LOAD_DEFERRING
    if (bIsDeferredInitializer)
    {
        const bool bIsDeferredSubObject = Obj->HasAnyFlags(RF_InheritableComponentTemplate);
        if (bIsDeferredSubObject)
        {
            // when this sub-object was created it's archetype object (the
            // super's sub-obj) may not have been created yet (thanks cyclic
            // dependencies). in that scenario, the component class's CDO would
            // have been used in its place; now that we're resolving the defered
            // sub-obj initialization we should try to update the archetype
            if (ObjectArchetype->HasAnyFlags(RF_ClassDefaultObject))
            {
                ObjectArchetype = UObject::GetArchetypeFromRequiredInfo(Class, Obj->GetOuter(), Obj->GetFName(), Obj->GetFlags());
                // NOTE: this may still be the component class's CDO (like when
                // a component was removed from the super, without resaving the child)
            }
        }

        UClass* ArchetypeClass = ObjectArchetype->GetClass();
        // If the class has been forgotten as part of a blueprint recompile and there is a newer version available.
        const bool bSuperHasBeenRegenerated = ArchetypeClass->HasAnyClassFlags(CLASS_NewerVersionExists);
#if USE_DEFERRED_DEPENDENCY_CHECK_VERIFICATION_TESTS
        check(bIsCDO || bIsDeferredSubObject);
        check(ObjectArchetype->GetOutermost() != GetTransientPackage());
        check(!bIsCDO || (ArchetypeClass == SuperClass && !bSuperHasBeenRegenerated));
#endif // USE_DEFERRED_DEPENDENCY_CHECK_VERIFICATION_TESTS
		// The archetype of the object has been regenerated, we cannot properly initialize inherited properties because the class layout may have changed.
        if (!ensureMsgf(!bSuperHasBeenRegenerated, TEXT("The archetype for %s has been regenerated, we cannot properly initialize inherited properties, as the class layout may have changed."), *Obj->GetName()))
        {
            // attempt to complete initialization/instancing as best we can, but
            // it would not be surprising if our CDO was improperly initialized          // as a result...

            // iterate backwards, so we can remove elements as we go
			// Traverse the list of component classes to be initialized after the C++ constructor, filling in the corresponding archetype for each subobject in the SubObjInitInfo structure
            for (int32 SubObjIndex = ComponentInits.SubobjectInits.Num() - 1; SubObjIndex >= 0; --SubObjIndex)
            {
                FSubobjectsToInit::FSubobjectInit& SubObjInitInfo = ComponentInits.SubobjectInits[SubObjIndex];
                const FName SubObjName = SubObjInitInfo.Subobject->GetFName();
				// Get the archetype of the parent object first. Then see if the class of the archetype of the parent object has written what the default object of this subobject should look like
                UObject* OuterArchetype = SubObjInitInfo.Subobject->GetOuter()->GetArchetype();
                UObject* NewTemplate = OuterArchetype->GetClass()->GetDefaultSubobjectByName(SubObjName);

                if (ensure(NewTemplate != nullptr))
                {// Template, refers to the object that this property should be copied from.
                    SubObjInitInfo.Template = NewTemplate;
                }
                else
                {// If this subobject has no archetype, remove it.
                    ComponentInits.SubobjectInits.RemoveAtSwap(SubObjIndex);
                }
            }
        }
    }
#endif // USE_CIRCULAR_DEPENDENCY_LOAD_DEFERRING
	// If properties should be initialized from the archetype
    if (bShouldInitializePropsFromArchetype)
    {
	    // If the current object is a CDO and there is currently no use of SDO on UClass or CDO for real-time reinstancing, assign BaseClass to the parent class of the object.
        UClass* BaseClass = (bIsCDO && !GIsDuplicatingClassForReinstancing) ? SuperClass : Class;
        if (BaseClass == NULL)
        {
            check(Class == UObject::StaticClass());
            BaseClass = Class;
        }
        // Check if the subobject has an archetype; if not, use the default object of the subobject's class as the default object.
        // GetDefaultObject here passes false, meaning "don't create it if the CDO doesn't exist"
        UObject* Defaults = ObjectArchetype ? ObjectArchetype : BaseClass->GetDefaultObject(false); // we don't create the CDO here if it doesn't already exist
        // Important* Call Init Properties to initialize properteis of object.
        InitProperties(Obj, BaseClass, Defaults, bCopyTransientsFromClassDefaults);
    }
    // If initializing properties from the archetype is not allowed
    const bool bAllowInstancing = IsInstancingAllowed(); // Indicates whether components of the object can be instantiated
    // This function is called for any default subobjects created through ObjectInitializer [InitProperties function].
    // Returns true if any subobject needs instantiation.
    bool bNeedSubobjectInstancing = InitSubobjectProperties(bAllowInstancing);

    // Restore class information if replacing native class.
    if (ObjectRestoreAfterInitProps != nullptr)
    {
        ObjectRestoreAfterInitProps->Restore();
        delete ObjectRestoreAfterInitProps;
        ObjectRestoreAfterInitProps = nullptr;
    }
    bool bNeedInstancing = false;
    // if HasAnyFlags(RF_NeedLoad), we do these steps later
#if !USE_CIRCULAR_DEPENDENCY_LOAD_DEFERRING
    if (!Obj->HasAnyFlags(RF_NeedLoad))
#else   
    // we defer this initialization in special set of cases (when Obj is a CDO   
    // and its parent hasn't been serialized yet)... in those cases, Obj (the   
    // CDO) wouldn't have had RF_NeedLoad set (not yet, because it is created   
    // from Class->GetDefualtObject() without that flag); since we've deferred  
    // all this, it is likely that this flag is now present... we want to run   
    // all this as if the object was just created, so we check   
    // bIsDeferredInitializer as well
    if ((!Obj->HasAnyFlags(RF_NeedLoad) || bIsDeferredInitializer)
#endif // !USE_CIRCULAR_DEPENDENCY_LOAD_DEFERRING
    {
        if (bIsCDO || Class->HasAnyClassFlags(CLASS_PerObjectConfig))
        {
            Obj->LoadConfig(NULL, NULL, bIsCDO ? UE::LCPF_ReadParentSections : UE::LCPF_None);
        }
        if (bAllowInstancing) // If components are allowed to be instantiated
        {
            // Instance subobject templates for non-cdo blueprint classes or when using non-CDO template.
            // If the class of the instance subobject was created in a blueprint or if its CDO object is different from the archetype. This indicates that it should be initialized from the archetype.
            const bool bInitPropsWithArchetype = Class->GetDefaultObject(false) == NULL || Class->GetDefaultObject(false) != ObjectArchetype || Class->HasAnyClassFlags(CLASS_CompiledFromBlueprint);
            if ((!bIsCDO || bShouldInitializePropsFromArchetype) && Class->HasAnyClassFlags(CLASS_HasInstancedReference) && bInitPropsWithArchetype)
            {
                // Only blueprint generated CDOs can have their subobjects instanced.
                check(!bIsCDO || !Class->HasAnyClassFlags(CLASS_Intrinsic | CLASS_Native));

                bNeedInstancing = true;
            }
        }
    }
    // Allow custom property initialization to happen before PostInitProperties is called
    if (PropertyInitCallback)
    {
        // autortfm todo: if this transaction aborts and we are in a transaction's open nest,
        // we need to have a way of propagating out that abort
        if (AutoRTFM::IsTransactional())
        {
            AutoRTFM::EContextStatus Status =

 AutoRTFM::Close([&]
            {
                PropertyInitCallback();
            });
        }
        else
        {
            PropertyInitCallback();
        }
    }
    // After the call to `PropertyInitCallback` to allow the callback to modify the instancing graph
    if (bNeedInstancing || bNeedSubobjectInstancing)
    {// Instantiate subobjects, copying from templates or archetypes.
        InstanceSubobjects(Class, bNeedInstancing, bNeedSubobjectInstancing);
    }
    // Make sure subobjects know that they had their properties overwritten
    for (int32 Index = 0; Index < ComponentInits.SubobjectInits.Num(); Index++)
    {
        SCOPE_CYCLE_COUNTER(STAT_PostReinitProperties);
        UObject* Subobject = ComponentInits.SubobjectInits[Index].Subobject;
        Subobject->PostReinitProperties();
    }
    {
        SCOPE_CYCLE_COUNTER(STAT_PostInitProperties);
        // Called after C++ constructor and property initialization, including properties loaded from config.
        // Called before any serialization or other setup happens. Translated as initialization properties afterward.
        Obj->PostInitProperties();
    }
    // Called during object construction after PostInitProperties to allow class-specific initialization of object instances.
    // It's a virtual function in UClass, not implemented. Currently found only in Unreal classes generated from Python types.
    Class->PostInitInstance(Obj, InstanceGraph);

#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)
    if (!FUObjectThreadContext::Get().PostInitPropertiesCheck.Num() || (FUObjectThreadContext::Get().PostInitPropertiesCheck.Pop(false) != Obj))
    {
        UE_LOG(LogUObjectGlobals, Fatal, TEXT("%s failed to route PostInitProperties. Call Super::PostInitProperties() in %s::PostInitProperties()."), *Obj->GetClass()->GetName(), *Obj->GetClass()->GetName());
    }
#endif // !(UE_BUILD_SHIPPING || UE_BUILD_TEST)

// Check if all TSubobjectPtr properties have been initialized.
#if !USE_CIRCULAR_DEPENDENCY_LOAD_DEFERRING
    if (!Obj->HasAnyFlags(RF_NeedLoad)
#else   
    // we defer this initialization in special set of cases (when Obj is a CDO   
    // and its parent hasn't been serialized yet)... in those cases, Obj (the   
    // CDO) wouldn't have had RF_NeedLoad set (not yet, because it is created   
    // from Class->GetDefualtObject() without that flag); since we've deferred  
    // all this, it is likely that this flag is now present... we want to run   
    // all this as if the object was just created, so we check   
    // bIsDeferredInitializer as well
    if ((!Obj->HasAnyFlags(RF_NeedLoad) || bIsDeferredInitializer)
#endif // !USE_CIRCULAR_DEPENDENCY_LOAD_DEFERRING
        // if component instancing is not enabled, then we leave the components in an invalid state, which will presumably be fixed by the caller
        && ((InstanceGraph == NULL) || InstanceGraph->IsSubobjectInstancingEnabled()))
    {
        Obj->CheckDefaultSubobjects();
    }
    Obj->ClearFlags(RF_NeedInitialization);

    // clear the object pointer so we can guard against running this function again
    Obj = nullptr;
}
```

#### Summary

The `PostConstructInit` function orchestrates the initialization process for properties, managing their archetypes and handling child objects. Additionally, it triggers `InitProperties` for default sub-objects associated with `FObjectInitializer`. In subsequent sections, the instantiation of sub-objects using `FObjectInstanceGraph` will be discussed. The general algorithm is outlined below:

- If a deferred initialization occurs due to circular dependencies, the prototype may not have been instantiated yet. In such cases, update the prototype/CDO and initiate further processing. This involves traversing the `ComponentInits` array to recursively refill prototypes for each sub-object.
- When initializing a sub-object using a prototype, locate the prototype and decide whether to use the CDO instead. Then, invoke `InitProperties` to perform a shallow copy of the memory for all properties, focusing on initialization rather than instantiation.
- `InitSubobjectProperties` is always invoked to ensure all sub-objects have `InitProperties` called on them for a specific object. This step initializes properties for all sub-objects and determines if they can be instantiated, including checking if any components need instantiation.
- If a component is configured to load from a file, perform a shallow copy. Use `LoadConfig` to read file data and assign values to the component.
- For objects represented by `BlueprintGeneratedClass`, determine if instantiation is necessary. Resolving one of the conditions where instantiation is required triggers a call to `InstanceSubobjects` for the sub-objects.
- Users have access to a `PropertyInitCallback` delegate just before `PostInitProperties` is called. `PostInitProperties` itself is invoked immediately after the C++ constructor, once properties, including those loaded from configuration, have been initialized. It precedes any serialization setup and serves to prepare memory for temporary data. Being a `virtual` function, it allows overriding or implementing custom logic, such as checking the Remote Role of an `AActor`.
- Objects not in a loadable state from disk or being loaded lazily must check their default sub-objects for proper lifecycle management, ensuring no improper references like pointers outside their outer chain.
- Finally, set the object's initialization flag (using `ClearFlags`), and clear the pointer held by the object initializer (`FObjectInitializer`) to prevent improper re-initialization.

#### InitProperties

The code logic behind `InitProperties` leverages `FProperty` along side the reflection system to copy property values from a template to the current property. In different circumstances the `PostConstructLink` or `PropertyLink` may be used for faster path traversal of properties and depending on template usage, (either a `CDO` or a `DefaultData` parameter) a memory copy is still performed. The `FProperty` class has a few public member functions used to handle different property operations. One of these public functions used in the code segment below is `CopyCompleteValue_InContainer`.

```cpp
public:  
    /**  
     * Copy the value for all elements of this property.  
     *  
     * @param Dest              the address where the value should be copied to.  
     *                          This should always correspond to the BASE + OFFSET, where  
     *                          BASE = (for member properties) the address of the UObject which contains this data,  
     *                          (for locals/parameters) the address of the space allocated for the function's locals  
     *                          OFFSET = the Offset of this FProperty  
     * @param Src               the address of the value to copy from. should be evaluated the same way as Dest  
     * @param InstancingParams  contains information about instancing (if any) to perform  
     */    
    FORCEINLINE void CopyCompleteValue(void* Dest, void const* Src) const  
    {  
        if (Dest != Src)  
        {  
            if (PropertyFlags & CPF_IsPlainOldData)  
            {  
                FMemory::Memcpy(Dest, Src, static_cast<size_t>(ElementSize) * ArrayDim);  
            }  
            else  
            {  
                CopyValuesInternal(Dest, Src, ArrayDim);  
            }  
        }  
    }  

    FORCEINLINE void CopyCompleteValue_InContainer(void* Dest, void const* Src) const  
    {  
        return CopyCompleteValue(ContainerPtrToValuePtr<void>(Dest), ContainerPtrToValuePtr<void>(Src));  
    }
```

This function straightforwardly includes a `MemCpy` operation, where the container pointer is passed as the `void* Dest` parameter.

Without delving deeply into the reflection system, Unreal Header Tool (UHT) parses `UPROPERTY` types within a container, setting the `Offset_Internal` for each type. This offset is relative to the starting address of the class. Therefore, `ContainerPtr + Offset_Internal` yields the memory address of that property.

Why is this important? When copying from an already instantiated template object, its `FObjectInitializer` has been destroyed, along with any useful override information like `ComponentsInit`. Thus, traversing the type information becomes the most viable approach for copying values from a prototype.

Below is the code for `InitProperties`, along with comments explaining segments of the code. This is also summarized.

```cpp
// Binary initialize object properties to zero or defaults.
void FObjectInitializer::InitProperties(UObject* Obj, UClass* DefaultsClass, UObject* DefaultData, bool bCopyTransientsFromClassDefaults)
{
    check(!GEventDrivenLoaderEnabled || !DefaultsClass || !DefaultsClass->HasAnyFlags(RF_NeedLoad));
    check(!GEventDrivenLoaderEnabled || !DefaultData || !DefaultData->HasAnyFlags(RF_NeedLoad));

    SCOPE_CYCLE_COUNTER(STAT_InitProperties);

    check(DefaultsClass && Obj);

    UClass* Class = Obj->GetClass();

    // bool to indicate that we need to initialize any non-native properties (native ones were done when the native constructor was called by the code that created and passed in a FObjectInitializer object)
    bool bNeedInitialize = !Class->HasAnyClassFlags(CLASS_Native | CLASS_Intrinsic);

    // bool to indicate that we can use the faster PostConstructLink chain for initialization.
    bool bCanUsePostConstructLink = !bCopyTransientsFromClassDefaults && DefaultsClass == Class;

    if (Obj->HasAnyFlags(RF_NeedLoad))
    {
        bCopyTransientsFromClassDefaults = false;
    }
    //If non-native class initalization is not needed then the initalization can be done using PostConstructLink chain. 
    if (!bNeedInitialize && bCanUsePostConstructLink)
    {
        // This is just a fast path for the below in the common case that we are not doing a duplicate or initializing a CDO and this is all native.
        // We only do it if the DefaultData object is NOT a CDO of the object that's being initialized. CDO data is already initialized in the object's constructor.
        if (DefaultData)
        {   // DefaultData is not the CDO for the object initalized.
            if (Class->GetDefaultObject(false) != DefaultData)
            {
	            // Traverse the list of properties using reflection from most derived class to the base class. *Using Property List*
                for (FProperty* P = Class->PropertyLink; P; P = P->PropertyLinkNext)
                {
                    bool bIsTransient = P->HasAnyPropertyFlags(CPF_Transient | CPF_DuplicateTransient | CPF_NonPIEDuplicateTransient);
                    	                //If the property points to a UObject reference marked CPF_NeedCtorLink, then if the property is not transient or does not hold a ref to a component that has been instantiated.
                    if (!bIsTransient || !P->ContainsInstancedObjectProperty()) // Check if Property is indeed contained in it's owning class
                    {
                        if (P->IsInContainer(DefaultsClass))
                        {
	                        // Copy the default property value from memeory, This is can be either a prototype or it's CDO. 
                            P->CopyCompleteValue_InContainer(Obj, DefaultData);
                        }
                    }
                }
            }
            else // Otherwise the defaultData object is a CDO being initalized.
            {
                // Copy all properties that require additional initialization (e.g. CPF_Config).
                for (FProperty* P = Class->PostConstructLink; P; P = P->PostConstructLinkNext)
                {
                    bool bIsTransient = P->HasAnyPropertyFlags(CPF_Transient | CPF_DuplicateTransient | CPF_NonPIEDuplicateTransient);
                    if (!bIsTransient || !P->ContainsInstancedObjectProperty())
                    {
                        if (P->IsInContainer(DefaultsClass))
                        {
                            P->CopyCompleteValue_InContainer(Obj, DefaultData);
                        }
                    }
                }
            }
        }
    }
    else // Here We're dealing with non-native class initalization
    {
        // As with native classes, we must iterate through all properties (slow path) if default data is pointing at something other than the CDO.
        bCanUsePostConstructLink &= (DefaultData == Class->GetDefaultObject(false));
		// If ClassDefaults is true, copy transients from CDO otherwise copy the transients from the Default Data.
        UObject* ClassDefaults = bCopyTransientsFromClassDefaults ? DefaultsClass->GetDefaultObject() : NULL;
        check(!GEventDrivenLoaderEnabled || !bCopyTransientsFromClassDefaults || !DefaultsClass->GetDefaultObject()->HasAnyFlags(RF_NeedLoad));
		// Check if we can initalize things using PostConstructLink since it's faster. If so Traverse it. The chain of properties that need to have post constructor initialization defined. Otherwise we have to traverse the property chain using property Link from dervived to base.
        for (FProperty* P = bCanUsePostConstructLink ? Class->PostConstructLink : Class->PropertyLink; P; P = bCanUsePostConstructLink ? P->PostConstructLinkNext : P->PropertyLinkNext)
        {
            if (bNeedInitialize)
            { //Initalize the non native property to it's initalization rules.
                bNeedInitialize = InitNonNativeProperty(P, Obj);
            }
            // Transient Subobject check
            bool bIsTransient = P->HasAnyPropertyFlags(CPF_Transient | CPF_DuplicateTransient | CPF_NonPIEDuplicateTransient);
            if (!bIsTransient || !P->ContainsInstancedObjectProperty())
            {
                if (bCopyTransientsFromClassDefaults && bIsTransient)
                {
                    // This is a duplicate. The value for all transient or non-duplicatable properties should be copied
                    // from the source class's defaults.
                    P->CopyCompleteValue_InContainer(Obj, ClassDefaults);
                }
                else if (P->IsInContainer(DefaultsClass))
                {
                    P->CopyCompleteValue_InContainer(Obj, DefaultData);
                }
            }
        }
        // This step is only necessary if we're not iterating the full property chain.
        if (bCanUsePostConstructLink)
        {
            // Initialize remaining property values from defaults using an explicit custom post-construction property list returned by the class object.
            Class->InitPropertiesFromCustomList((uint8*)Obj, (uint8*)DefaultData);
        }
    }
}
```

#### Summary

`InitProperties` is where property initialization takes place, specifically through a shallow copy of property values. The process unfolds as follows:

- Initially, `UClass` flags are checked to ascertain if any non-native properties require initialization (native properties are already handled during the native constructor's invocation).
- The `PostConstructLink` chain is evaluated for potential initialization, which represents the faster path.
- If non-native class initialization isn't necessary, the `PostConstructLink` pointers are used for initialization.
- If `DefaultData` passed isn't the CDO for the object being initialized, traverse the `Property List` using `PropertyLink`.
- While traversing `DefaultData` properties, ascertain if the property references a `UObject`. If the `UObject` isn't `CPF_Transient` or is `CPF_NeedCtorLink` (indicating it's not transient or doesn't hold a reference to an instantiated component), copy the value.
- If `DefaultData` is indeed a CDO, copy all properties by traversing the `PostConstructLink` chain.
- In cases of non-native class initialization, again evaluate if the `PostConstructLink` chain suffices. Similar to previous logic, check if `DefaultData` is the CDO or not. Additionally, verify if instantiation is required, check for transience, and determine if the property type is a `UObject` holding a sub-object reference. After these checks, copy the property value.
#### InitSubObjectProperties

For sub-objects found within the `ComponentInits` array, their data also needs initialization. Therefore, when a sub-object is encountered, its template is passed to the `InitProperties` function to copy properties from.

```cpp
bool FObjectInitializer::InitSubobjectProperties(bool bAllowInstancing) const
{
#if USE_CIRCULAR_DEPENDENCY_LOAD_DEFERRING
    bool bNeedSubobjectInstancing = bAllowInstancing && bIsDeferredInitializer;
#else
    bool bNeedSubobjectInstancing = false;
#endif

    // initialize any subobjects, now that the constructors have run
    for (int32 Index = 0; Index < ComponentInits.SubobjectInits.Num(); Index++)
    {
        UObject* Subobject = ComponentInits.SubobjectInits[Index].Subobject;
        UObject* Template = ComponentInits.SubobjectInits[Index].Template;

        // Initialize each subobject in the list by calling the InitProperties function.
        InitProperties(Subobject, Template->GetClass(), Template, false);

        if (bAllowInstancing && !Subobject->HasAnyFlags(RF_NeedLoad))
        {
            bNeedSubobjectInstancing = true;
        }
    }

    return bNeedSubobjectInstancing;
}
```

"The `InitProperties` function is called within the `PostConstructInit` function to copy the property list of the main object, while the `InitSubobjectProperties` function is called to copy the property lists of all sub-objects. This process simply involves a shallow copy."

#### Instance Sub-Objects 

Object instantiation in Unreal Engine involves several stages, particularly within the `NewObject` function. The process begins with memory allocation followed by property initialization from a prototype object. When it comes to sub-object initialization, shallow copies are made from the property list of the child object prototype. Sub-object instantiation, on the other hand, is a deep copy process. Since `FProperty` is decoupled from `UObject`, type checking derives from a container type like `FObjectPropertyBase`, which holds references to `UObject`.

From debugging the engine source code, the process of "instancing sub-objects" was observed in two key areas:

Firstly, within the `FObjectInitializer` class, there's a member function named `InstanceSubObjects`. This function ensures that sub-objects required by the main object being constructed are properly instantiated and initialized. Leveraging the instantiation and initialization system of `FObjectInitializer` adheres to the builder pattern for handling sub-objects.

For instance, when an actor is placed into a level, the level invokes `NewObject` to start the construction chain for that actor. If the actor includes component sub-objects, their inner chain needs initialization as well. The challenge here is managing the hierarchy of sub-objects and ensuring proper instantiation matching with their template objects, a task handled by the `FObjectInstancingGraph`.

Secondly, `InstanceSubObjects` is also a virtual member function within the `FProperty` class. This allows properties to instantiate their own sub-objects based on their specific implementation. Examples of properties that implement this virtual function include `FObjectPropertyBase`, `FArrayProperty`, `FMapProperty`, and others.

Another scenario where `InstanceSubObjects` is crucial occurs during the finding or loading of a `UObject` for a class. During loading, `ConditionalPostLoadSubobjects` is called to verify all instanced properties of a class, checking if any new object property instances were added since the last save. Additionally, it ensures that all sub-object instance properties have values consistent with their defaults. Subsequently, an `InstanceGraph` is allocated for serialized sub-objects, and `InstanceSubobjectTemplates` is invoked on the loaded `UObject` to instantiate its sub-objects for initialization.

#### Instance Sub-Objects FObjectInitializer

As discussed in previous sections, `FObjectInitializer` is instantiated with the class constructor to allow for customized post-initialization, enabling different representations of the default state. Therefore, the builder pattern is applied to sub-objects because their instantiation is tied to the initializer itself. The commented code below explains what's going on, and a summary is provided at the end.

```cpp
void FObjectInitializer::InstanceSubobjects(UClass* Class, bool bNeedInstancing, bool bNeedSubobjectInstancing) const  
{  
    SCOPE_CYCLE_COUNTER(STAT_InstanceSubobjects);  
	//"FObjectInstancingGraph" structure, which contains mapping from instantiated objects and components to their templates. Used to instantiate components owned by a newly instantiated object.
    FObjectInstancingGraph TempInstancingGraph;
    // If the member variable InstanceGraph of FObjectInitializer is not empty, assign it to UseInstancingGraph, otherwise use a new InstanceGraph.
    FObjectInstancingGraph* UseInstancingGraph = InstanceGraph ? InstanceGraph : &TempInstancingGraph;  

    // Add any default subobjects or new mapping (from the currently instantiated object (not its child object to its template))
    UseInstancingGraph->AddNewObject(Obj, ObjectArchetype);  
	// Add mappings from all defaults child objects for this object to their templates
    for (const FSubobjectsToInit::FSubobjectInit& SubobjectInit : ComponentInits.SubobjectInits)  
    {       
        UseInstancingGraph->AddNewObject(SubobjectInit.Subobject, SubobjectInit.Template);  
    }    

    if (bNeedInstancing)  
    { 
        UObject* Archetype = ObjectArchetype ? ObjectArchetype : Obj->GetArchetype();
        // Class of this Object being initialized.
        // InstanceSubObjectTemplates will iterate over the property list
        // saved in UClass and call the instatiation function on that property.  
        Class->InstanceSubobjectTemplates(Obj, Archetype, Archetype ? Archetype->GetClass() : NULL, Obj, UseInstancingGraph);  
    }    

    if (bNeedSubobjectInstancing)  
    {       
        // initialize any subobjects, now that the constructors have run
        // Really this means initialize any child objects.  
        for (int32 Index = 0; Index < ComponentInits.SubobjectInits.Num(); Index++)  
        {          
            UObject* Subobject = ComponentInits.SubobjectInits[Index].Subobject;  
            UObject* Template = ComponentInits.SubobjectInits[Index].Template;  

#if USE_CIRCULAR_DEPENDENCY_LOAD_DEFERRING  
            if ( !Subobject->HasAnyFlags(RF_NeedLoad) || bIsDeferredInitializer )  
#else   
            if ( !Subobject->HasAnyFlags(RF_NeedLoad) )  
#endif  
            {// InstanceSubobjectTemplates iterates over the properties of the UClass for subobject itself. Subsequently calls the instantiation function for each property. More specifically the properties for child objects and instantiates them. Recall, the property has it's own implementation on how its instantiated.
                Subobject->GetClass()->InstanceSubobjectTemplates(Subobject, Template, Template->GetClass(), Subobject, UseInstancingGraph);  
            }       
        }    
    }
}
```

The `FObjectInstancingGraph` contains mappings of instantiated objects and their template objects, defining components for newly instantiated objects.

- `UseInstancingGraph` determines if the member variable `InstanceGraph` of `FObjectInitializer` is empty (indicating it wasn't initialized in the constructor); if so, it creates a new one.
- If there are any default sub-objects or new mappings from the currently instantiated object, they are added.
- Then, mappings for all default child objects of this object, including its template, are added.
- If instancing is required, the class of this object is traversed, and `InstanceSubObjectTemplates` iterates over any child objects to call their instantiation functions.
- For sub-objects needing instancing, such as a component, whose constructor has already been called, initialization of its child objects proceeds.
- Initialization of child object properties iterates with the `UClass` of the sub-object. `InstanceSubobjectTemplates` then manages calls to the instantiation function for each property.
#### InstanceSubobjectTemplates

`UObject` actually has a `InstanceSubobjectTemplate` member function also but it's just a wrapper to the `UStruct` function that actually does the work.

```cpp
void UStruct::InstanceSubobjectTemplates(void* Data, void const* DefaultData, UStruct* DefaultStruct, UObject* Owner, FObjectInstancingGraph* InstanceGraph)
{
    checkSlow(Data);
    checkSlow(Owner);

    for (FProperty* Property = RefLink; Property != NULL; Property = Property->NextRef)
    {
        if (Property->ContainsInstancedObjectProperty() && (!InstanceGraph || !InstanceGraph->IsPropertyInSubobjectExclusionList(Property)))
        {
            Property->InstanceSubobjects(Property->ContainerPtrToValuePtr<uint8>(Data), (uint8*)Property->ContainerPtrToValuePtrForDefaults<uint8>(DefaultStruct, DefaultData), Owner, InstanceGraph);
        }
    }
}
```

For each property, a determination is made to check if it has a reference to an already-instanced property. If it does, the instantiation function of that property is called.

- Additionally, `ContainsInstanceObjectProperty` ensures that a temporary object doesn't have a reference to a non-temporary property, likely to prevent dangling pointers.
- `InstanceSubobjects` takes several parameters, with the first being the current value for the property in the child object.
- `ContainerPtrToValuePtr` returns the value of a property in its owning object instance.
- `ContainerPtrToValuePtrForDefaults` retrieves the value of this property in the template of the child object.

#### FProperty->InstanceSubObjects

The `InstanceSubObjects` version for `FProperty`, as mentioned before, is called to instantiate itself, reducing the boilerplate code needed for different object types. We've also learned that `FProperty` is decoupled from the `UObject` system and serves as a container holding type information as an attribute inside `UCLASS`. In the example below, `FObjectPropertyBase` is one of many classes that has a `virtual` function to define its own instantiation implementation. Since it's a container for a `UObject`, the instantiation code needs to convert this attribute into an actual object and copy over its data. Any derived classes may require the user to implement this logic to cater to their initialization needs. Below is an example of `FObjectPropertyBase`'s default implementation, followed by a summary.

```cpp
void FObjectPropertyBase::InstanceSubobjects(void* Data, void const* DefaultData, UObject* InOwner, FObjectInstancingGraph* InstanceGraph)
{
	// ArrayDim determines the dimension of the property, this just means that when iterating over the dimension it handles container-type properties; If the property isn't a container (Array etc..) then the for loop will execute once. 
    for (int32 ArrayIndex = 0; ArrayIndex < ArrayDim; ArrayIndex++)
    {
	    // CurrentValue holds the current property value in the instanced object. Adding ArrayIndex * ElementSize determines the offset for each element in the container.
		// Due to Dynamic Dispatch, GetObjectPropertyValue is actually implemented by FObjectProperty, it doesn't return Null.
		// GetObjectPropertyValue actually calls
		// static FORCEINLINE TCppType const* GetPropertyValuePtr(void const* A)
		// This converts the address of the property value to the correct type and returns it.
        UObject* CurrentValue = GetObjectPropertyValue((uint8*)Data + ArrayIndex * ElementSize);
        if (CurrentValue)
        {
	        //If not null, generate a template subobject. Afterwards check if the passed default template is valid; If yes, retrieve its value from the instance of the default template object
            UObject* SubobjectTemplate = DefaultData ? GetObjectPropertyValue((uint8*)DefaultData + ArrayIndex * ElementSize) : nullptr;
            //!! This is where the instantiation of the property happens with the InstanceGraph.
            UObject* NewValue = InstanceGraph->InstancePropertyValue(SubobjectTemplate, CurrentValue, InOwner, HasAnyPropertyFlags(CPF_InstancedReference) ? EInstancePropertyValueFlags::CausesInstancing : EInstancePropertyValueFlags::None);
            /* Similarly due to Dynamic Dispatch the actual function called is:
                static FORCEINLINE void SetPropertyValue(void* A, TCppType const& Value)
                {
                    *GetPropertyValuePtr(A) = Value;
                }
               Retrieves the value of the property from the address and then assigns NewValue to it.
               It's not a simple use of the assignment operator = to perform the assignment operation, for flexibility reasons.*/
            SetObjectPropertyValue((uint8*)Data + ArrayIndex * ElementSize, NewValue);
        }
    }
}
```

- The initial for loop traverses the `ArrayDim` of the property, allowing handling of container-type properties (like arrays and maps).
- `CurrentValue` retrieves the current property value in the instantiated object. `ArrayIndex * ElementSize` manages elements within the container. `GetObjectPropertyValue` is overridden by `FObjectProperty`. Due to polymorphism and dynamic dispatch, `GetObjectPropertyValue` calls `static FORCEINLINE TCppType const* GetPropertyValuePtr(void const* A)`, which converts the address of the property value to the appropriate type and returns it.
- If `CurrentValue` isn't null, then create the `SubObjectTemplate` from the `DefaultData`.
- The `InstanceGraph` member function `InstancePropertyValue` is the crucial step in the process where the instantiation of the property actually occurs.
- `SetObjectPropertyValue` is also overridden, and again, due to Dynamic Dispatch, `static FORCEINLINE void SetPropertyValue` is actually called.
#### FObjectInstancingGraph

The `FObjectInstancingGraph` is the structure used to create a sub-object tree, instantiating components owned by new objects. The structure is well-documented, providing various examples of source-to-destination objects stored in a `TMap`. The key of the map is the template object, and the value is the instance created from that template.

```cpp
struct FObjectInstancingGraph
{
public:
    /** 
     * Default Constructor 
     * @param bDisableInstancing - if true, start with component instancing disabled 
     **/
    COREUOBJECT_API explicit FObjectInstancingGraph(bool bDisableInstancing = false);

    /** 
     * Constructor with options
     * @param InOptions Additional options to modify the behavior of this graph
     **/
    COREUOBJECT_API explicit FObjectInstancingGraph(EObjectInstancingGraphOptions InOptions);

    /** 
     * Standard constructor
     * @param DestinationSubobjectRoot the top-level object that is being created
     * @param InOptions Additional options to modify the behavior of this graph 
     **/
    COREUOBJECT_API explicit FObjectInstancingGraph(class UObject* DestinationSubobjectRoot, EObjectInstancingGraphOptions InOptions = EObjectInstancingGraphOptions::None);

    /** 
     * Returns the component that has SourceComponent as its archetype, instancing the component as necessary.
     * @param SourceComponent the component to find the corresponding component instance for
     * @param CurrentValue the component currently assigned as the value for the component property being instanced.
     * Used when updating archetypes to ensure that the new instanced component replaces the existing component instance in memory.
     * @param CurrentObject the object that owns the component property currently being instanced;
     * this is NOT necessarily the object that should be the Outer for the new component.
     * @param Flags reinstancing flags - see EInstancePropertyValueFlags
     * @return As with GetInstancedSubobject, above, but also deals with archetype creation and a few other special cases
     */
    COREUOBJECT_API class UObject* InstancePropertyValue(class UObject* SourceComponent, class UObject* CurrentValue, class UObject* CurrentObject, EInstancePropertyValueFlags Flags = EInstancePropertyValueFlags::None);

private:
    /** 
     * Returns the component that has SourceComponent as its archetype, instancing the component as necessary.
     * @param SourceComponent the component to find the corresponding component instance for
     * @param CurrentValue the component currently assigned as the value for the component property being instanced.
     * Used when updating archetypes to ensure that the new instanced component replaces the existing component instance in memory.
     * @param CurrentObject the object that owns the component property currently being instanced;
     * this is NOT necessarily the object that should be the Outer for the new component.
     * @param Flags reinstancing flags - see EInstancePropertyValueFlags
     * @return if SourceComponent is contained within SourceRoot, returns a pointer to a unique component instance corresponding to
     * SourceComponent if SourceComponent is allowed to be instanced in this context, or NULL if the component isn't allowed to be
     * instanced at this time (such as when we're a client and the component isn't loaded on clients)
     * if SourceComponent is not contained by SourceRoot, return INVALID_OBJECT, indicating that the that has SourceComponent as its ObjectArchetype,
     * or NULL if SourceComponent is not contained within SourceRoot.
     */
COREUOBJECT_API class UObject* GetInstancedSubobject(class UObject* SourceSubobject, class UObject* CurrentValue, class UObject* CurrentObject, EInstancePropertyValueFlags Flags);

    /** 
     * The root of the object tree that is the source used for instancing components;
     * - when placing an instance of an actor class, this would be the actor class default object
     * - when placing an instance of an archetype, this would be the archetype
     * - when creating an archetype, this would be the actor instance
     * - when duplicating an object, this would be the duplication source
     */
    class UObject* SourceRoot;
    
    /** 
     * The root of the object tree that is the destination used for instancing components
     * - when placing an instance of an actor class, this would be the placed actor
     * - when placing an instance of an archetype, this would be the placed actor
     * - when creating an archetype, this would be the actor archetype
     * - when updating an archetype, this would be the source archetype
     * - when duplicating an object, this would be the copied object (destination)
     */
    class UObject* DestinationRoot;

    /** Subobject instancing options */
    EObjectInstancingGraphOptions InstancingOptions;

    /** 
     * Indicates whether we are currently instancing components for an archetype. 
     * true if we are creating or updating an archetype.
     */
    bool bCreatingArchetype;

    /** 
     * true when loading object data from disk.
     */
    bool bLoadingObject;

    /** 
     * Maps the source (think archetype) to the destination (think instance)
     */
    TMap<class UObject*, class UObject*> SourceToDestinationMap;

    /** 
     * List of member variable properties that should not instantiate subobjects 
     */
    TSet<const FProperty*> SubobjectInstantiationExclusionList;
};

```

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/FObjectInstancingGraph.svg]]
The diagram describes the engine comments of different `SourceRoot` and `DestinationRoot` pairs. You can now imagine what's going on behind the scenes. For example, when duplicating an `Actor` instance in the editor or dragging an `Actor` into a level, the `FObjectInstancingGraph` is used to reconstruct the structure of that `Actor` and perform a `deep copy` when all the nested sub-objects instantiate themselves. The `FObjectInstancingGraph` is also used inside `PostLoadSubobjects` when objects are loaded from disk. Briefly speaking, during deserialization, the serialized components and their default sub-objects are collected as `SourceRoot` to later generate the `DestinationRoot` value.

##### InstancePropertyValue

After reading `DarkflameMasters` articles and reviewing the engine source for `FObjectInstancingGraph`, it's evident that the graph is responsible for instantiating property values. Remember, the instantiation of instanced sub-objects is defined by `FProperty` itself. This design choice avoids writing boilerplate code inside `UObject`, which makes sense for modular and maintainable code. Additionally, additional source code comments have been incorporated with a summary.

```cpp
UObject* FObjectInstancingGraph::InstancePropertyValue(class UObject* ComponentTemplate, class UObject* CurrentValue, class UObject* Owner, bool bIsTransient, bool bCausesInstancing, bool bAllowSelfReference)
{
    UObject* NewValue = CurrentValue;

    check(CurrentValue);

    // The current value flag is checked to determine if it references the class of this property default to be instanced. Previously, this was for subclasses of UComponent, but now can be any UObject.
    if (CurrentValue->GetClass()->HasAnyClassFlags(CLASS_DefaultToInstanced))
    {
        bCausesInstancing = true; // These are instanced no matter what
    }

    // If subobject instancing is not enabled or 
    // if it doesn't cause instancing and no self-reference is allowed or 
    // if it's not a delegate
    // then we don't instantiate, just return the current value as it is.
    if (!IsSubobjectInstancingEnabled() || 
        (!bCausesInstancing && 
         !bAllowSelfReference)) 
    {
        return NewValue;
    }

    // If we're instancing a component for an object that has its prototype chain and the prototype has this component property as nullptr,
    // it means the prototype didn't instantiate it, so neither should we.
    if (ComponentTemplate == nullptr && CurrentValue != nullptr && (Owner && Owner->IsBasedOnArchetype(CurrentValue->GetOuter())))
    {   
        // Handle the case where the prototype is a nullptr but the current property is not
        // In this case, the value needs to be set to a nullptr. This is in place to not overdiligently assign a value. This is like giving an answer to a question without being asked to.
        NewValue = nullptr;
    }
    else
    {
        if (ComponentTemplate == nullptr)
        {  
            // Only need to be here if our prototype doesn't have this component property
            ComponentTemplate = CurrentValue;
        }
        // **Core:** The actual "instantiation" work is done here in GetInstancedSubobject.
        UObject* MaybeNewValue = GetInstancedSubobject(ComponentTemplate, CurrentValue, Owner, bAllowSelfReference, bAllowSelfReference);
        if (MaybeNewValue != INVALID_OBJECT)
        {
            NewValue = MaybeNewValue;
            // ReplaceMap is a map of instantiated objects that need their references updated.
            ReplaceMap.Add(CurrentValue, NewValue);
        }
    }
    return NewValue;
}
```

- The current value flag is checked to determine if it references the class of this property default to be instanced. This means that references to this class by default create their own separate instances instead of sharing one. In the legacy days, this applied to subclasses of `UComponent`, but now it can apply to any `UObject`.
    
- Secondly, a check is made to determine if sub-object instancing is enabled. If the object doesn't require instancing or self-reference is allowed and it's not a delegate, then instantiation is skipped, and the current value is returned immediately.
    
- If we're instantiating a component for an object that has its prototype chain, and the prototype has this component property set to nullptr, it implies that the prototype didn't instantiate it either. In this case, the value is set to nullptr. This prevents overzealous assignment of values without explicit need.
    
- Otherwise, `GetInstancedSubobject` is called, where the actual initialization work is performed for the sub-object.
    
- Lastly, the `ReplaceMap` is utilized to fix or update references of instantiated objects.

##### GetInstancedSubobject 

We're at the endgame in the code where `StaticConstructObject_Internal` finally returns an `InstancedSubobject`. Throughout the code, you'll encounter both original and additional comments that describe the process in detail. The crucial parameter to track here is `CurrentValue`. Its state changes in various scenarios during the function call before eventually being assigned the returned `InstancedSubobject`. The objective of this function is to locate the sub-object instance from the `FObjectInstancingGraph`. If it's not found, a new instance is created.

```cpp
// SourceSubobject, the template object being copied, could be an archetype
// or a class default object.
// We need to find its corresponding component instance in FObjectInstancingGraph.

// CurrentValue, the current value of the subobject to be instantiated.
// It needs to copy from SourceSubobject.
// CurrentObject, the object we initially want to initialize with
// FObjectInitializer. CurrentValue is a member variable of this object.

// bDoNotCreateNewInstance, whether to allow creating new instances
// during object instantiation. If true, it does not create a new
// instance, but reallocates one if a mapping already exists in the table.

// bAllowSelfReference, whether to allow self-reference (in other words,
// whether it's a delegate)

// If SourceComponent is contained within SourceRoot, returns a pointer 
// to the unique component instance corresponding to SourceComponent 
// (if it's allowed to be instantiated in this context, otherwise returns
// NULL)

//* If this function is called, and the component is not allowed to be
//  instantiated (e.g., when we're not loading the client), or
//* if SourceComponent is not contained within SourceRoot, returns INVALID_OBJECT

UObject* FObjectInstancingGraph::GetInstancedSubobject(UObject* SourceSubobject, UObject* CurrentValue, UObject* CurrentObject, EInstancePropertyValueFlags Flags)
{
    checkSlow(SourceSubobject);
    bool bDoNotCreateNewInstance = !!(Flags & EInstancePropertyValueFlags::DoNotCreateNewInstance);
    bool bAllowSelfReference = !!(Flags & EInstancePropertyValueFlags::AllowSelfReference);

    UObject* InstancedSubobject = INVALID_OBJECT;

    // If the component's template is not empty, and the currently instantiated component is not empty.
    if (SourceSubobject != nullptr && CurrentValue != nullptr && !CurrentValue->IsIn(CurrentObject))
    {
        // Here the source subobject is actually the ComponentTemplate passed in when calling the GetInstancedSubobject function last time, which is the template object to be copied.
        // If the source object tree's root is the same as the passed-in template object, it means it's self-referential
        const bool bAllowedSelfReference = bAllowSelfReference && SourceSubobject == SourceRoot;
        // If SourceSubobject is contained within SourceRoot (i.e., the template object is a node in the source object tree)
        // or self-reference is allowed (such as delegates), then allow instantiating this component
        bool bShouldInstance = bAllowedSelfReference || SourceSubobject->IsIn(SourceRoot);

        // If the template object of this subobject is not in the root of the source object tree, and the parent object of this subobject is the prototype of the main object,
        // This part of the exception handling is not very clear to me. If any reader understands, please feel free to comment.
        if (!bShouldInstance && CurrentValue->GetOuter() == CurrentObject->GetArchetype())
        {
            // This code is intended to capture the following scenarios: 1. SourceRoot contains subobjects assigned to instantiated object properties
            // 2. The class of the subobject contains components, and the class of the subobject is not outside the inheritance hierarchy of SourceRoot.
            // For example: a weapon class, which includes UIObject subobject definitions in its default properties,
            // where the properties referencing UIObjects are marked for instantiation.
            bShouldInstance = true;

            // If this situation occurs, please make sure the CurrentValue of the component property still points to the template component.
            check(SourceSubobject == CurrentValue);
        }

        // If instantiation of the component is allowed
        if (bShouldInstance)
        {
            // Search in the mapping saved in the object instantiation graph for the unique component instance corresponding to this component template
            // SourceToDestinationMap is a member variable of FObjectInstancingGraph, establishing a mapping from source objects to destination objects
            InstancedSubobject = GetDestinationObject(SourceSubobject);

            if (InstancedSubobject == nullptr)// If it's not found
            {
                // And if it's not allowed to create a new instance (note that allowing instantiation and allowing creation of new instances are two different things)
                if (bDoNotCreateNewInstance)
                {
                    InstancedSubobject = INVALID_OBJECT; // Keep it unchanged
                }
                else
                {
                    // If the current Outer assigned to this component is the same as the object we are instantiating the component with,
                    // then the component does not need to be instantiated; otherwise, there are two possibilities:
                    // 1. CurrentValue is a template and needs to be instantiated
                    // 2. CurrentValue is an instantiated component, in which case, it should
                    // already be in the InstanceGraph, unless the component is created at runtime (e.g., editinline export property).
                    // If this is the case, CurrentValue will be an instance of the component template that is not linked to the prototype referenced by CurrentObject
                    // In this case, we also do not want to reinstantiate the component template

                    // Determine if the component is created at runtime.
                    bool bIsRuntimeInstance = CurrentValue != SourceSubobject && CurrentValue->GetOuter() == CurrentObject;

                    if (bIsRuntimeInstance)
                    {   // As mentioned in the comment above, if it's created at runtime, the engine does not want to reinstantiate the component template, so the return value remains unchanged.
                        InstancedSubobject = CurrentValue;
                    }
                    else
                    {
                        // If the component template is relevant in this context (client vs. server vs. editor), then instantiate it.
                        const bool bShouldLoadForClient = SourceSubobject->NeedsLoadForClient();
                        // If the object should load for the client, it's true... the same for the other two.
                        const bool bShouldLoadForServer = SourceSubobject->NeedsLoadForServer();
                        const bool bShouldLoadForEditor = (GIsEditor && (bShouldLoadForClient || !CurrentObject->RootPackageHasAnyFlags(PKG_PlayInEditor)));

                        if (((GIsClient && bShouldLoadForClient) || (GIsServer && bShouldLoadForServer) || bShouldLoadForEditor))
                        {
                            // This is the first request for the instance corresponding to SourceSubobject
                            // Get the object instance corresponding to the Outer of the source component, which is the object that will be used as the Outer for the target component (i.e., this copy will include the Outer as well)
                            UObject* SubobjectOuter = GetDestinationObject(SourceSubobject->GetOuter());

                            // If our template is detached from the deep nested UObject hierarchy, with several objects linked nested in the object graph,
                            // it is entirely possible that we encounter a UObject whose Outer copy we have not yet discovered and instantiated.
                            // In this case, we need to continue and instantiate that outer.
                            if (SubobjectOuter == nullptr)
                            {
                                SubobjectOuter = GetInstancedSubobject(SourceSubobject->GetOuter(), SourceSubobject->GetOuter(), CurrentObject, Flags);

                                checkf(SubobjectOuter && SubobjectOuter != INVALID_OBJECT, TEXT("No corresponding destination object found for '%s' while attempting to instance subobject '%s'"), *SourceSubobject->GetOuter()->GetFullName(), *SourceSubobject->GetFullName());
                            }

                            FName SubobjectName = SourceSubobject->GetFName();

                            // Don't search for the existing subobjects on Blueprint-generated classes. What we'll find is a subobject  
                            // created by the constructor which may not have all of its fields initialized to the correct value (which should be coming from a blueprint). 
                            if (!SubobjectOuter->GetClass()->HasAnyClassFlags(CLASS_CompiledFromBlueprint))
                            {
                                InstancedSubobject = StaticFindObjectFast(nullptr, SubobjectOuter, SubobjectName);
                            }

                            if (InstancedSubobject && IsCreatingArchetype() && !InstancedSubobject->HasAnyFlags(RF_LoadCompleted))
                            {   // since we are updating an archetype, this needs to reconstruct as that is the mechanism used to copy properties  
                                // it will destroy the existing object and overwrite it      
                                InstancedSubobject = nullptr;
                            }

                            if (!InstancedSubobject)
                            {   // finally, create the subobject instance
                                FStaticConstructObjectParameters Params(SourceSubobject->GetClass());
                                Params.Outer = SubobjectOuter;
                                Params.Name = SubobjectName;
                                Params.SetFlags = SubobjectOuter->GetMaskedFlags(RF_PropagateToSubObjects);
                                Params.Template = SourceSubobject;
                                Params.bCopyTransientsFromClassDefaults = true;
                                Params.InstanceGraph = this;
                                InstancedSubobject = StaticConstructObject_Internal(Params);
                            }
                        }
                    }
                }
            }
            else if (IsLoadingObject() && InstancedSubobject->GetClass()->HasAnyClassFlags(CLASS_HasInstancedReference))
            {
                /* When loading an object from disk, in some cases, we have a component that has a reference to another component in DestinationObject, which has not been serialized or instantiated yet.
                   For example, the PointLight class declares two component templates:

                        Begin DrawLightRadiusComponent0
                        End
                        Components.Add(DrawLightRadiusComponent0)

                        Begin MyPointLightComponent
                            SomeProperty=DrawLightRadiusComponent
                        End
                        LightComponent=MyPointLightComponent

                   After handling the LightComponent property, the array of components will be handled by UClass::InstanceSubobjectTemplates.
                   If the instance of DrawLightRadiusComponent0 was created in the last session (i.e., when the object was saved),
                   and it is identical to the component template from the PointLight class,
                   and the instance of MyPointLightComponent was serialized,
                   then the instance of MyPointLightComponent will exist in the InstanceGraph, but the instance of DrawLightRadiusComponent0 will not.
                   To handle this case, ensure that the variables of the instance of MyPointLightComponent (such as SomeProperty, which is the drawing light radius component mentioned above)
                   are set to the correct value - set to the instance of DrawLightRadiusComponent0 - it will be used as
                   the return value from the ConditionalPostLoad function on the point light object.
                   We must call ConditionalPostLoadSubobjects on every existing component instance encountered,
                   while still being able to access all component instances owned by PointLight.
                */
                // This function recursively calls ConditionalPostLoadSubobjects on the object being loaded for all its parent classes and archetypes.
                // It then adds its subobjects and subobject archetypes to the mapping in the instance graph. It sets itself as the root of the destination object tree in the instance graph.
                // I'm not very clear on the rest of this, but if the reader allows me to take a little liberty and be confused on this part, it would be appreciated _(:з」∠)_
                // However, if the reader is interested in this, I have also commented on the PostLoad-related functions and put them in the link to the cloud drive for download to see.
                // If any reader understands this part, I would greatly appreciate it if the reader could enlighten me.
                InstancedSubobject->ConditionalPostLoadSubobjects(this);
            }
        }
    }

    // If the search in FObjectInstancingGraph finds the unique instance corresponding to the component template, we just return at the end of the function.
    return InstancedSubobject;
}

```

- The first step is to check if the component's template is not empty and the instantiated component is not empty. This ensures that the template exists in the Source Tree of the object `FObjectInstanceGraph`. Self-reference of the sub-object is also checked to ensure the object type is not a delegate (since it's bindable). If this check fails, it should not be instanced and marked as `INVALID_OBJECT`.
    
- Another check is made to see if the templated object of this sub-object is not the root of the object tree and the parent of this sub-object is the prototype of the main object. This translates to checking if we're dealing with a class default sub-object that has sub-objects marked for instantiation and any sub-objects defined as default properties point to the main class as the source root, as it's template.
    
- A search is made when calling `GetDestinationObject` to determine if the template object is a source object in the `FObjectInstancingGraph`. If found, just return the instance. Otherwise, if it's not found (`nullptr`), additional logic is required to instantiate it correctly.
    
- If `bDoNotCreateNewInstance` is true, don't instance it and assign `INVALID_OBJECT`.
    
- If the `CurrentValue` is an instanced sub-object, then it should be in the instancing graph. However, the sub-object can be an instance created at runtime and will not be linked to a `SourceSubobject` template from its archetype object. If this is true, return `CurrentValue` as a shallow copy.
    
- If it's not a runtime case, then determine if the object needs to be loaded for client/server or editor use. At this point, an instance is required according to the `SourceSubobject`. Look for the `SourceSubobject`'s outer instance, which will be used as the outer of the destination object.
    
- If the outer `SourceSubobject` instance is null, then the engine comments mention that it's possible we're in a deeply nested `UObject` hierarchy with several links to objects nested in the object graph. In that case, make a recursive call to instance the outer.
    
- If the sub-object is generated from a blueprint class, call `StaticFindObjectFast` to return a value.
    
- The last two cases occur during serialization: firstly, if an archetype is being updated after a load, then it's purposely destroyed by setting a `nullptr` to the object. In the next if statement, `StaticConstructObject_Internal` is called to recursively build the sub-object.
    
- Lastly, during deserialization, the `FObjectInstancingGraph` could be loaded from disk where the class instance of the sub-object has references to other components. `ConditionalPostloadSubobjects` is called to ensure the sub-object still needs to instance its own components and fix up serialization references up the outer hierarchy chain.

#### Final Summary

Unreal Engine's unique construction and initialization process of objects follows many layers of abstraction, starting with the early stages of the Reflection system. UHT generates C++ code needed to capture all the types marked with `UClass`. The macros `IMPLEMENT_CLASS` and `DECLARE_CLASS` define boilerplate code needed for `StaticClass()` to construct a `UClass`.

During `Preinit()` at editor startup, the `CoreUObject` module is loaded first, and `StartupModule` binds delegates used by `ProcessNewlyLoadedObjects` to be notified when to process/register `UClass` objects loaded after `CoreUObject` is fully initialized. The allocation and construction of the first `UObject` begins in the Registration phase. `UClassRegisterAllCompiledInClasses` retrieves the statically allocated Class Registry to process all `UClasses`, starting with native types (UClass, UStruct, etc.). Function pointers set forth by the reflection system are called, and `StaticClass()` begins the construction process for an object. Inside `GetPrivateStaticClassBody()`, `GUObjectAllocator` calls `AllocatedUObject` to preallocate the `UClass` using `placement new` and sets aside a memory block with the appropriate alignment for `UObjectBase`.

Once the `UClass` layout is allocated, the `UObject` initialization begins. In the case of `UObject`, its `Outer` package is constructed first using `NewObject` to create a transient package. This is part of the early construction process of `UObject` to set up the garbage collector and allocate the global object memory pool. After `UObjectBaseInit` is called, the `UObject` system is deemed initialized. Next, the pending registrant linked list is traversed by `UObjectProcessRegistrant` to force registration of the objects by calling `DeferredRegister`. `DeferredRegister` creates the outer package and hashes the `UObject` for later access.

`NewObject` is Unreal's method of constructing objects in a ubiquitous fashion. It introduces two key functions: `StaticConstructObject_Internal` and `StaticAllocateObject`. `StaticConstructObject_Internal` calls the templated default constructor of `UClass`, wrapped inside `InternalConstructor`. `InternalConstructor` is ultimately aliased to `ClassConstructorType` as a function pointer, assigned to the `ClassConstructor` member variable of `UClass`. The `DefaultConstructor` can be invoked with or without an initializer passed into the constructed `UClass`. Before construction sets any initial values, `StaticAllocateObject` determines whether there are any old objects that can be recycled. If the object exists, the already allocated memory block at that address is reused; otherwise, a new block is allocated. The `NewObject` process is called twice, each time invoking different overloaded versions of the constructor. The `UObject` created is added to the global object array pool, and the second invocation calls the parent class constructor.

After the `UObject` system is initialized, the remaining class objects loaded from additional modules are constructed inside `ProcessNewlyLoadedObjects`. In the final stages of `UClass` construction, the CDO `Template` object is created with a `GetDefaultObject()` function call and is referenced inside the `UClass`. The CDO represents a `Template` object that defines the default state for all instances created with `NewObject`. The template object follows the Prototype design pattern; this special object is used in different initialization and construction scenarios, allowing runtime instancing while enabling customization of the construction process.

The `Template` object is often misunderstood and isn't always a CDO object. A templated object can be an archetype loaded from disk or even a custom instanced object provided by the user, which can override the CDO as the primary template object in the initialization process.

The last sections cover the initialization system. `FProperty`, `FObjectInitializer`, and `Serialization` work together to emulate the Builder design pattern. `FObjectInitializer` is an instantiated object parameter used to perform custom construction and initialization.

Property initialization logic is part of the `FObjectInitializer` lifecycle. It begins when an object constructor gets a `ThreadContext` singleton to access an initializer stack stored in thread-specific memory. The Initializer objects are pushed onto the initializer stack to cache the order of constructor calls made from the most derived class all the way up to the base, including any nested default sub-objects contained within that object. Upon destruction of the initializer object, post-initialization logic starts processing object properties recursively until all Initializers are popped off the initializer stack.

Due to the complexity of initializing many `UPROPERTY` types within a class or structure, the traversal and processing of these properties can be slow and cumbersome. Therefore, `UProperty` was refactored from `UObject` and redesigned into `FProperty` to optimize overall performance.

The property initialization system relies on the setup and creation of `FProperty` types. The creation starts with the Unreal Header Tool parsing property macros into generated code. The generated code collects the property types into an array of `PropPointers`. These `PropPointers` are `FPropertyParamsBase` types that later become type-casted to the appropriate `FProperty` during construction. The construction code is defined inside the `DECLARE_FIELD` macro, similar to the `DECLARE_CLASS` macro.

`FField` and `FFieldClass` expose reflection functionality used by the initialization system to traverse `FProperties` aggregated within `UStruct`. Furthermore, `Serialization` links the `UStruct` with an `FArchive` wrapper to fix up any object references and set memory offsets for each property. This step assigns the correct references to pointers: `PropertyLinkPtr`, `DestructorLinkPtr`, `RefLinkPtr`, and `PostConstructLinkPtr`. These pointers are tied to a set of linked lists that hold the order of derived properties, object references, destruction, and properties requiring post-initialization after construction. These linked lists are integral to the initialization process.

`Serialization` is also involved in property initialization during the saving or loading of assets from disk. `Tofu Theory` describes the basics of a serialization system, where an interpretable format can be deduced by slicing a byte array into sections. This format is later interpreted to determine how to load the data and their types back to their original state on any target machine. However, creating a byte array format is not straightforward for Unreal's `FProperty` and `UObject` types. To address this, Unreal uses two different serialization methods: `TaggedPropertySerializer` and `UnversionedPropertySerializer` are header formats used to tag serialized properties. When `FPropertyTags` are extracted during deserialization, the `UClass` is searched based on that tag information so the initialization process can map the data object back into memory.

To customize the serialization of object types, `FArchive` acts as a visitor interface that passes through the virtual function `void Serialize(FArchive& Ar)`. Following the Visitor Pattern, `FArchive` traverses a complex object tree to apply different types of serialization methods. `UObject` serialization depends on the `FArchive` passed. Ultimately, the order of serialization follows the outer relationship of an object's hierarchy; this is crucial when an object is initialized and re-instanced with `NewObject` into memory. During deserialization of object instances, determining pointer references between other objects can be challenging to recreate correctly into memory. This is addressed by `Pointer Swizzling`, with unique `Virtual Paths` generated by the engine. `UAssets` contain `Export` and `Import` tables that map these `Virtual Paths`. These tables recreate the reference relationship between the object contained in the package (Export) and the information of the class objects references (Import) back into memory. Usually, this involves reconstruction and initialization of loaded objects unless they're found in memory with their `UClass` information. If no such object exists in memory, then the construction of a dummy object is performed. Post-construction initialization begins, traversing `FProperties` and setting default values for native types and instancing any sub-objects recursively with their template objects.

Therefore, when creating a new object—whether deserializing a newly loaded object, duplicating, modifying, or re-instancing an object in the editor—the instantiation process performs shallow or deep copy (recursively) with the object initializer passed to the main object constructor inside `StaticConstructObject_Internal`. The construction of sub-object attributes can be complex when dealing with a main object that contains sub-object components. These sub-objects can have their own deep inheritance hierarchy and contain their own instanced sub-objects. The `FObjectInitializer` stack is used to cache this recursive process.

When post-initializing sub-object components, both the main object and sub-objects are added to an `FObjectInstancingGraph`. The `FObjectInstancingGraph` is a hashmap that holds a key-value pair between the template object and its instance. The `FObjectInstancingGraph` is traversed based on the references stored inside the `ComponentInits` list; this is where each sub-object added has post-initialization work done. To clarify, the `FObjectInstancingGraph` isn't just used to hold references for specific sub-objects that require custom post-initialization. The data structure is built to define a main object's template objects and its instanced objects, ensuring correct deep copies of objects.

Instantiation of all sub-objects starts inside `PostConstructInit`. The `InstanceSubobjectTemplates` is a member function of `UStruct` called upon to traverse all `UClass` properties defined by the main object and the sub-objects. `FProperty` has its own version of `InstanceSubObjects`, which is called to instantiate itself for each property. The `FProperty` type `FObjectPropertyBase` is the base class for a `UObject` type. It contains a virtual function where derived properties can override how to instantiate themselves when called. Finally, when `InstanceSubobject` is called on the property, it attempts to read the value for the property in the current object by calling `InstanceGraph` `InstancePropertyValue`. Once instantiated, it writes the value back. It's apparent that this instantiation logic is encapsulated within the instance graph. This continues to follow the Builder design pattern. Lastly, `GetInstanceSubObject` gets `MaybeNewValue`. This is the last step in the instantiation process of a sub-object, where the state of the variable may change in different scenarios before assigning the actual value. If the sub-object is not found, it will instantiate a new one.

Unreal Engine has to deal with many different construction scenarios, and it does its best to not overcomplicate the creation of nested objects. While no system is perfect, the system does a great job of covering most construction scenarios a programmer may encounter.