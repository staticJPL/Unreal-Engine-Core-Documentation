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