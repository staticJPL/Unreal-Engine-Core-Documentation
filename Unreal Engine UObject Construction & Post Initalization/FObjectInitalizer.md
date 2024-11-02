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
![[Unreal Engine UObject Construction & Post Initalization/Diagrams/TLSExample.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/df8203e0b86a76a5627ca7d527dab6e353bd20d7/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/TLSExample.svg)
The concept is fairly simple. Imagine two situations side by side in a diagram: on the left-hand side, threads 1-3 have their own local methods and variables that write to a single global memory space. Since the memory space is shared among those threads, it obviously creates a race condition. To solve this issue, the thread local storage design pattern is employed. Since C++11, `thread_local` has been a keyword storage class specifier. For thread-local types, initialization and destruction occur within the thread's lifetime.

Thread local storage isolates its own global variables specific to each individual thread, operating on its own copy of the global variable. This reduces processing overhead and eliminates the need for a mutex to lock a single global memory pool across multiple threaded operations.

Each thread stores TLS data in a slot of a TLS array. Before usage, each index must be allocated by that thread. Typically, threads store their data directly in the TLS slot. However, for larger datasets, separate storage may be allocated to minimize TLS slot usage.

From the Microsoft documentation the image illustrates this below.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/ThreadContextMicrosoft.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/df8203e0b86a76a5627ca7d527dab6e353bd20d7/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/ThreadContextMicrosoft.png)

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

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/FObjectInitalizer.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/df8203e0b86a76a5627ca7d527dab6e353bd20d7/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/FObjectInitalizer.svg)
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
