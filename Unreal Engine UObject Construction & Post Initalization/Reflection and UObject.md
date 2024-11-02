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
