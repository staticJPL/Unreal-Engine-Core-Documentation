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

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/NewObjectCallStack.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/e415abbcf731f82657768593ba319e9d0e54885b/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/NewObjectCallStack.png)

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
