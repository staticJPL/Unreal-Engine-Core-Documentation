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

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/FObjectInstancingGraph.svg]](https://github.com/staticJPL/Unreal-Engine-Core-Documentation/blob/c7569c23d36d1ff200fd5d86e41460cea36fb057/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/FObjectInstancingGraph.svg)

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
