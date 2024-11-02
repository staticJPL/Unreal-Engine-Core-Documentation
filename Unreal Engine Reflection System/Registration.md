# Registration

#### Pre-Registration

In the collection section, the pre-registration phase begins with the collection of `ClassInfo`, `ScriptStructInfo`, and `EnumInfo`. These collected pieces of information are then passed into `TDeferredRegistry` during static automatic initialization. Before the engine starts running, `Z_CompiledinDeferFile` forwards all the reflected type information to `TDeferredRegistry` for each generated translation unit. It's worth noting that there are separate registries for each type.

```cpp
using FClassDeferredRegistry = TDeferredRegistry<FClassRegistrationInfo>;  
using FEnumDeferredRegistry = TDeferredRegistry<FEnumRegistrationInfo>;  
using FStructDeferredRegistry = TDeferredRegistry<FStructRegistrationInfo>;  
using FPackageDeferredRegistry = TDeferredRegistry<FPackageRegistrationInfo>;
```

`TDeferredRegistry` uses the Singleton design pattern to load data into it's registry as it's needed. 

```cpp
/**  
* Return the registry singleton  
*/  
static TDeferredRegistry& Get()  
{  
    static TDeferredRegistry Registry;  
    return Registry;  
}
```

  
In the singleton pattern, which is frequently used to manage static initialization order challenges in C++, Unreal Engine leverages this approach to populate a registry singleton. This registry ensures items are added in a specific order by encapsulating the singleton within a getter function. This ensures the singleton is created on first use or when explicitly needed. Specifically, the Deferred Registry in Unreal Engine manages the loading of prerequisites in a defined sequence, where CoreUObject types are prioritized over others. This ensures that critical types are initialized and registered early during engine startup.

While examining the engine's source code, I encountered challenges in pinpointing the exact mechanism that maintains this initialization order and manages static initialization. However, by setting breakpoints on the singleton and observing the behavior in Unreal Engine versions 5.3 and later, I observed that approximately 3343 items were initialized and added to the Deferred Registry array before the engine's main function was called.

Further investigation into the call stack revealed that DLL main is invoked during allocation. This suggests that the order of allocation for the deferred registry is managed through dynamic linking before the engine's startup. This dynamic loading of modules indicates a specific order of linking DLLs, while static initialization occurs locally within the DLL libraries, adhering to the engine's modular design principles.
#### UObject Static initialization

For each UClass in Unreal Engine, the `StaticClass() `function is crucial as it internally invokes `GetPrivateStaticClass`. This invocation is facilitated by the `IMPLEMENT_CLASS` macro during the static initialization phase. It's important to highlight that UObject's `UClass` is among the first objects to undergo static initialization using the deferred registry mechanism. `UObject` itself has a dedicated reflection file, and this initialization process establishes the fundamental framework for subsequent type constructions and registrations within the Unreal Engine. This approach ensures that UObject types are properly initialized and prepared for use throughout the engine's runtime.
##### NoExportTypes

NoExportTypes.h holds the declaration that marks the UObject as a UClass with special metadata tags.

```cpp
/**  
 * Direct base class for all UE objects * @note The full C++ class is located here: Engine\Source\Runtime\CoreUObject\Public\UObject\Object.h  
 */
UCLASS(abstract, noexport, MatchedSerializers)  
class UObject  
{  
    GENERATED_BODY()  
public:  
  
    UObject(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());  
    UObject(FVTableHelper& Helper);  
    ~UObject();  
    /**  
     * Executes some portion of the ubergraph.     *     * @param  EntryPoint The entry point to start code execution at.     */    UFUNCTION(BlueprintImplementableEvent, meta=(BlueprintInternalUseOnly = "true"))  
    void ExecuteUbergraph(int32 EntryPoint);  
};
```

  
Moreover, the reflection file for UObject includes declarations for intrinsic types inherited from UObject that lack the `UClass` macro and don't possess their own **gen.cpp** and **generated.h** files. Delving deeper into intrinsic types leads to the discovery of `UnrealTypePrivate.h,` which contains declarations for the macro.

`DECLARE_CASTED_CLASS_INTRINSIC_WITH_API`
  
This macro is employed to manually introduce intrinsic types and link them with an API; in this case, it would be the `CoreModule`. Ultimately, the `IMPLEMENT_CORE_INTRINSIC_CLASS` macro is utilized to produce the reflection information of classes inherited from `UObject`.

The file` NoExportTypes.gen.cpp` is where UHT inserts the static collection code pattern we've seen for the reflected type information of a `UObject`.

```cpp
UClass* Z_Construct_UClass_UObject()  
{  
    if (!Z_Registration_Info_UClass_UObject.OuterSingleton)  
    {       
	    UECodeGen_Private::ConstructUClass(Z_Registration_Info_UClass_UObject.OuterSingleton, Z_Construct_UClass_UObject_Statics::ClassParams);  
    }    
    return Z_Registration_Info_UClass_UObject.OuterSingleton;  
}
```

`IMPLEMENT_CLASS(UObject,0)` is associated with construction of the outer singleton for UClass of UObject.

**GetPrivateStaticClass**

1. Sets the package name to be passed as the `OuterPrivate` for the UClass* object, associating it with a UPackage* object after constructing the `OuterSingleton` of UClass. "/Script/" indicates the module associated with the UObject, typically CoreUObject.
    
2. `StaticRegisterNativesUMyClass` sets up the Native Funcs associated with the UClass (`execCallableFunc`, `execNativeFunc`).
    
3. The `InternalConstructor` template wrapper facilitates C++ constructor invocation (due to the absence of function pointers for C++ constructors), enabling the calling of the constructor for our class.
    
4. `Super` is defined as the base class, and `WithinClass` refers to the type of outer object for the UObject. "Super" signifies that the type must depend on the base class to build a UClass before any subclass can be built from it. Therefore, `WithinClass` indicates that a UObject is within it after being built, and should be restricted to the kind of outer it's placed under. This is why UClass* must be built for the outer it belongs to in advance.

**GetPrivateStaticClassBody**

1. Responsible for allocating UObject memory, `GUObjectAllocator` handles the allocation of memory to store UClass Objects.
    
2. There are several UClass constructor types with different overloaded parameters. One includes the signature of `EC_StaticConstructor`, used for static initialization. Later in the registration phase, the construction process occurs in two steps, involving `GUObjectAllocator` to manage memory allocation without directly calling new in standard C++.
    
3. `InitializePrivateStaticClass`, when invoked, takes in `TClass_Super_StaticClass`, which then sets up `ClassWithin` by calling `TClass_WithinClass_StaticClass` first, followed by `Super::StaticClass()` and `WithinClass::StaticClass()`.

```cpp
COREUOBJECT_API void InitializePrivateStaticClass(  
    class UClass* TClass_Super_StaticClass,  
    class UClass* TClass_PrivateStaticClass,  
    class UClass* TClass_WithinClass_StaticClass,  
    const TCHAR* PackageName,  
    const TCHAR* Name  
    )  
{  
    TRACE_LOADTIME_CLASS_INFO(TClass_PrivateStaticClass, Name);  
  
    /* No recursive ::StaticClass calls allowed. Setup extras. */  
    if (TClass_Super_StaticClass != TClass_PrivateStaticClass)  
    {       
	    TClass_PrivateStaticClass->SetSuperStruct(TClass_Super_StaticClass);  
    }
    else  
    {  
       TClass_PrivateStaticClass->SetSuperStruct(NULL);  
    }
    
    TClass_PrivateStaticClass->ClassWithin = TClass_WithinClass_StaticClass;  
  
    // Register the class's dependencies, then itself.  
    TClass_PrivateStaticClass->RegisterDependencies();  
    {  
	    // Defer  
       TClass_PrivateStaticClass->Register(PackageName, Name);  
    }}
}
```

  
In the provided code snippet, `SuperStruct` is set as a `UStruct*`, which is defined in `UStruct` and points to the base class of this type. The `ClassWithin` parameter is utilized to constrain the type for the outer object.

During the initial stages of registration, `UObjectBase::Register()` is invoked. Later in the registration phase, this function is called in conjunction with `UClassRegisterAllCompiledInClasses`.
##### UObjectBase::Register()

```cpp
/** Enqueue the registration for this object. */  
void UObjectBase::Register(const TCHAR* PackageName,const TCHAR* InName)  
{  
    LLM_SCOPE(ELLMTag::UObject);  
    TMap<UObjectBase*, FPendingRegistrantInfo>& PendingRegistrants = FPendingRegistrantInfo::GetMap();  
  
    FPendingRegistrant* PendingRegistration = new FPendingRegistrant(this);  
    PendingRegistrants.Add(this, FPendingRegistrantInfo(InName, PackageName));  
  
#if USE_PER_MODULE_UOBJECT_BOOTSTRAP  
    if (FName(PackageName) != FName("/Script/CoreUObject"))  
    {       
		TMap<FName, TArray<FPendingRegistrant*>>& PerModuleMap = GetPerModuleBootstrapMap();  
	    PerModuleMap.FindOrAdd(FName(PackageName)).Add(PendingRegistration);  
    }
    else  
#endif  
    {  
       if (GLastPendingRegistrant)  
       {          
	       GLastPendingRegistrant->NextAutoRegister = PendingRegistration;  
       }
       else  
       {  
          check(!GFirstPendingRegistrant);  
          GFirstPendingRegistrant = PendingRegistration;  
       } 
       
		GLastPendingRegistrant = PendingRegistration;  
    }}
```

The `UObjectBase` Register function records and tracks registrations in a comprehensive global singleton Map and global linked list. Further details on the rationale behind this approach will be explored later.

#### Engine Startup Process

The objective at this stage is to progress to the `PreInit` step in the startup process. During PreInit, the initial module, CoreUObject, is loaded, which concludes with the final registration steps of populating type data into an allocated `UObject` of a `UClass`. The diagram below illustrates this engine startup process up to `AppInit`, which will be discussed in more detail.

![[Unreal Engine Reflection System/Reflection Diagrams/EngineStartupProcess.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/8945f68b9e6edf96ab6f9af87f93d9003b82a088/Unreal%20Engine%20Reflection%20System/Reflection%20Diagrams/EngineStartupProcess.svg)
During static initialization, translation units statically allocate type data structures. Reviewing the call stack during a deferred registry call reveals a sequence that includes DLLMain(), which may precede WinMain(). Understanding DLL linking and loading is complex; I'll outline key aspects here.

The DLL entry point involves establishing the C Runtime environment, which reserves memory and initiates thread creation for DLL linkage. When a DLL is loaded, the operating system invokes its entry point, often "DllMainCRTStartup", to execute DLLMain().

In Unreal Engine, investigating how DLLMain is invoked reveals a custom approach, potentially implementing a custom CRTStartup. This setup aids in initializing constructors for both static and non-local data.

In modular builds, the loader executable manages the loading order of UE module DLLs independently from its own operation. The loader undergoes static initialization, subsequently loading DLLs. DLL statics initialize when WinAPI's LoadModule is called by the engine.

This intricate process ensures that static initialization and dynamic DLL loading are orchestrated methodically, maintaining a structured setup for the Unreal Engine.

##### main(), WinMain(), DLLMain()

**WinMain()**: This is the entry point for Windows applications ending with .exe. It signals the start of the process to the operating system (OS) and encapsulates the call to `int main()` from the OS perspective.

**main()**: In C++, `main()` is case-sensitive; `Main()` and `main()` are distinct. It must return an `int` and serves as the primary entry point for a C++ program.

**DllMain()**: Specifically for DLLs, `DllMain()` is the entry point. DLLs cannot run independently but attach to a process initiated by `main()` or `WinMain()`.

In summary, these entry points serve distinct roles, each crucial for starting processes and encapsulating engine operations. This encapsulation is evident in the engine loop, where GuardedMain manages startup phases.

```cpp
int32 WINAPI WinMain(_In_ HINSTANCE hInInstance, _In_opt_ HINSTANCE hPrevInstance, _In_ char* pCmdLine, _In_ int32 nCmdShow)  
{  
    int32 Result = LaunchWindowsStartup(hInInstance, hPrevInstance, pCmdLine, nCmdShow, nullptr);  
    LaunchWindowsShutdown();  
    return Result;  
}
```

Inside `WinMain`, the program proceeds with a call to `LaunchWindowsStartup`, which handles essential Windows setup and boilerplate operations. After this initialization, GuardedMainWrapper() is invoked, which then calls GuardedMain() to initiate the core operations of the program.

```cpp
/**  
 * Static guarded main function. Rolled into own function so we can have error handling for debug/ release builds depending * on whether a debugger is attached or not. */
int32 GuardedMain( const TCHAR* CmdLine )  
{  
    FTrackedActivity::GetEngineActivity().Update(TEXT("Starting"), FTrackedActivity::ELight::Yellow);  
  
    FTaskTagScope Scope(ETaskTag::EGameThread);  
  
#if !(UE_BUILD_SHIPPING)  
  
    // If "-waitforattach" or "-WaitForDebugger" was specified, halt startup and wait for a debugger to attach before continuing  
    if (FParse::Param(CmdLine, TEXT("waitforattach")) || FParse::Param(CmdLine, TEXT("WaitForDebugger")))  
    { 
	    while (!FPlatformMisc::IsDebuggerPresent())  
	       {          
		       FPlatformProcess::Sleep(0.1f);  
	       }  
	       UE_DEBUG_BREAK();  
    }  
#endif  
  
    BootTimingPoint("DefaultMain");  
  
    // Super early init code. DO NOT MOVE THIS ANYWHERE ELSE!  
    FCoreDelegates::GetPreMainInitDelegate().Broadcast();  
  
	// make sure GEngineLoop::Exit() is always called.
	struct EngineLoopCleanupGuard 
	{ 
		~EngineLoopCleanupGuard()
		{
			// Don't shut down the engine on scope exit when we are running embedded
			// because the outer application will take care of that.
			if (!GUELibraryOverrideSettings.bIsEmbedded)
			{
				EngineExit();
			}
		}
	} CleanupGuard;
  
// Set up minidump filename. We cannot do this directly inside main as we use an FString that requires   
// destruction and main uses SEH.  
// These names will be updated as soon as the Filemanager is set up so we can write to the log file.    // That will also use the user folder for installed builds so we don't write into program files or whatever.#if PLATFORM_WINDOWS  
FCString::Strcpy(MiniDumpFilenameW, *FString::Printf(TEXT("unreal-v%i-%s.dmp"), FEngineVersion::Current().GetChangelist(), *FDateTime::Now().ToString()));  
#endif  
  
    FTrackedActivity::GetEngineActivity().Update(TEXT("Initializing"));  
    int32 ErrorLevel = EnginePreInit( CmdLine );  
  
    // exit if PreInit failed.  
    if ( ErrorLevel != 0 || IsEngineExitRequested() )  
    {       
	    return ErrorLevel;  
    }  
    
    {
    
	FScopedSlowTask SlowTask(100, NSLOCTEXT("EngineInit", "EngineInit_Loading", "Loading..."));  
  
// Set up minidump filename. We cannot do this directly inside main as we use an FString that requires 
// destruction and main uses SEH.
// These names will be updated as soon as the Filemanager is set up so we can write to the log file.
// That will also use the user folder for installed builds so we don't write into program files or whatever.

#if PLATFORM_WINDOWS
	FCString::Strcpy(MiniDumpFilenameW, *FString::Printf(TEXT("unreal-v%i-%s.dmp"), FEngineVersion::Current().GetChangelist(), *FDateTime::Now().ToString()));
#endif

	FTrackedActivity::GetEngineActivity().Update(TEXT("Initializing"));
	int32 ErrorLevel = EnginePreInit( CmdLine );

	// exit if PreInit failed.
	if ( ErrorLevel != 0 || IsEngineExitRequested() )
	{
		return ErrorLevel;
	}

	{
		FScopedSlowTask SlowTask(100, NSLOCTEXT("EngineInit", "EngineInit_Loading", "Loading..."));

	// EnginePreInit leaves 20% unused in its slow task.
	// Here we consume 80% immediately so that the percentage value on the splash screen doesn't change from one slow task to the next.
	// (Note, we can't include the call to EnginePreInit in this ScopedSlowTask, because the engine isn't fully initialized at that point)
		SlowTask.EnterProgressFrame(80);

		SlowTask.EnterProgressFrame(20);

#if WITH_EDITOR
		if (GIsEditor)
		{
			ErrorLevel = EditorInit(GEngineLoop);
		}
		else
#endif
		{
			ErrorLevel = EngineInit();
		}
	}

	double EngineInitializationTime = FPlatformTime::Seconds() - GStartTime;
	UE_LOG(LogLoad, Log, TEXT("(Engine Initialization) Total time: %.2f seconds"), EngineInitializationTime);

#if WITH_EDITOR
	UE_LOG(LogLoad, Log, TEXT("(Engine Initialization) Total Blueprint compile time: %.2f seconds"), BlueprintCompileAndLoadTimerData.GetTime());
#endif

	ACCUM_LOADTIME(TEXT("EngineInitialization"), EngineInitializationTime);

	BootTimingPoint("Tick loop starting");
	DumpBootTiming();

	FTrackedActivity::GetEngineActivity().Update(TEXT("Ticking loop"), FTrackedActivity::ELight::Green);

	// Don't tick if we're running an embedded engine - we rely on the outer
	// application ticking us instead.
	if (!GUELibraryOverrideSettings.bIsEmbedded)
	{
		while( !IsEngineExitRequested() )
		{
			EngineTick();
		}
	}

	TRACE_BOOKMARK(TEXT("Tick loop end"));

#if WITH_EDITOR
	if( GIsEditor )
	{
		EditorExit();
	}
#endif
	return ErrorLevel;
}
```

GuardedMain's role is to set up the Engine Loop and transfer control to GEngineLoop. Within this context, you can observe the engine tick function being invoked. The subsequent step is EnginePreInit.

```cpp
/**   
 * PreInits the engine loop   
 */  
int32 EnginePreInit( const TCHAR* CmdLine )  
{  
    int32 ErrorLevel = GEngineLoop.PreInit( CmdLine );  
  
    return( ErrorLevel );  
}
```
 
During `PreInit`, the function `PreInitPreStartupScreen` is called with command line arguments as parameters. Within PreInitPreStartupScreen, a critical step involves the invocation of LoadCoreModules().

```cpp
bool FEngineLoop::LoadCoreModules()  
{  
    // Always attempt to load CoreUObject. It requires additional pre-init which is called from it's module's StartupModule method.  
#if WITH_COREUOBJECT  
#if USE_PER_MODULE_UOBJECT_BOOTSTRAP // otherwise do it later  
    FModuleManager::Get().OnProcessLoadedObjectsCallback().AddStatic(ProcessNewlyLoadedUObjects);  
#endif  
    return FModuleManager::Get().LoadModule(TEXT("CoreUObject")) != nullptr;  
#else  
    return true;  
#endif  
}
```

As observed in the code, the LoadCoreModules function prioritizes the loading of CoreUObject first. Once the pre-initialization process, including PreInitPreStartupScreen, is completed, the subsequent step involves initializing UObjects for registration within the type system. This marks the completion of the startup process for UObjects, facilitating the subsequent loading of all other engine modules.

#### Type System Registration

![[Unreal Engine Reflection System/Reflection Diagrams/Registration.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/8945f68b9e6edf96ab6f9af87f93d9003b82a088/Unreal%20Engine%20Reflection%20System/Reflection%20Diagrams/Registration.svg)

A type system registration diagram to visualize the process. This diagram purpose is to aid in following registration process.
#### Startup Module

```cpp
virtual void StartupModule() override  
{  
   // Register all classes that have been loaded so far. This is required for CVars to work.       
UClassRegisterAllCompiledInClasses();  

   void InitUObject();  
   FCoreDelegates::OnInit.AddStatic(InitUObject);  

   // Substitute Core version of async loading functions with CoreUObject ones.  
   IsInAsyncLoadingThread = &IsInAsyncLoadingThreadCoreUObjectInternal;  
   IsAsyncLoading = &IsAsyncLoadingCoreUObjectInternal;  
   SuspendAsyncLoading = &SuspendAsyncLoadingInternal;  
   ResumeAsyncLoading = &ResumeAsyncLoadingInternal;  
   IsAsyncLoadingSuspended = &IsAsyncLoadingSuspendedInternal;  
   IsAsyncLoadingMultithreaded = &IsAsyncLoadingMultithreadedCoreUObjectInternal;  

   // Register the script callstack callback to the runtime error logging  
#if UE_RAISE_RUNTIME_ERRORS  
   FRuntimeErrors::OnRuntimeIssueLogged.BindStatic(&FCoreUObjectModule::RouteRuntimeMessageToBP);  
#endif  

   // Make sure that additional content mount points can be registered after CoreUObject loads  
   FPackageName::OnCoreUObjectInitialized();  

#if DO_BLUEPRINT_GUARD  
   FFrame::InitPrintScriptCallstack();  
#endif  
}
```
  
In Unreal Engine, during the startup of the `CoreUObject` module, the `StartupModule` entry point is crucial. It facilitates the initialization process through mechanisms like `FCoreDelegate`, ensuring systematic execution of initialization routines such as `InitUObject`. This setup ensures that `InitUObject` is registered to be called by `AppInit` at a later stage, underscoring the organized and structured approach to initialization within the UObject system. This methodology helps maintain order and reliability during the engine startup process.
#### UClassRegisterAllCompiledInClasses

```cpp
/** Register all loaded classes */  
void UClassRegisterAllCompiledInClasses()  
{  
    SCOPED_BOOT_TIMING("UClassRegisterAllCompiledInClasses");  
    LLM_SCOPE(ELLMTag::UObject);  
  
    FClassDeferredRegistry& Registry = FClassDeferredRegistry::Get();  
  
    Registry.ProcessChangedObjects();  
  
    for (const FClassDeferredRegistry::FRegistrant& Registrant : Registry.GetRegistrations())  
    {       
		 UClass* RegisteredClass = FClassDeferredRegistry::InnerRegister(Registrant);
	}
}
```

During this stage in the call stack, the construction process for UClass begins. Initially, all elements from the deferred registry that were added during the static initialization phase are loaded. After loading these registrants, the core `UObject` becomes the first `UClass` to have its `inner` registrant called in `StaticClass`.

In more precise terms, UClass instances are created for all reflected types from .gen.cpp files.

**Editor vs Runtime**

UClass can vary between editor and runtime environments due to differences in DLL link order, especially in monolithic builds where all pending registrants are processed simultaneously by `ProcessNewlyLoadedObjects`. Unreal Engine incorporates protective measures to ensure necessary classes are loaded for preprocessing.

`UClass` relies on dependencies such as `SuperStruct` and `WithinClass`. In scenarios where the static initialization order is uncertain, this dependency order becomes crucial. However, since the UObject module is statically loaded first, these dependencies are typically resolved during initialization.

In editor mode, `CoreUObject` is statically linked first, followed by other critical engine modules. Conversely, during runtime for a game module, the DLL loading order is reversed. The executable loads other DLLs internally after its own execution begins, initializing static variables within those DLLs.

Metadata for registered information is stored within a UScriptStruct or UEnum. Before constructing any other UObject types like UScriptStruct or UEnum, UClass must be instantiated because it stores crucial class information. Once the `CoreUObject` module is fully loaded, all types described by UClass are available for use.
#### AppInit

During AppInit, a broadcast of a multicast delegate is made to notify systems within Unreal Engine of the initialization stage.

```cpp
bool FEngineLoop::AppInit()
// Init other systems.  
{  
    SCOPED_BOOT_TIMING("FCoreDelegates::OnInit.Broadcast");  
    FCoreDelegates::OnInit.Broadcast();  
}
```

More importantly, this broadcast will call `InitUObject`, which was bound inside the Startup Module of `CoreUObject`.

#### InitUObject

```cpp
void InitUObject()  
{  
    LLM_SCOPE(ELLMTag::InitUObject);  
  
    FGCCSyncObject::Create();  
  
    // Initialize redirects map  
    FCoreRedirects::Initialize();  
    for (const FString& Filename : GConfig->GetFilenames())  
    {       
		CoreRedirects::ReadRedirectsFromIni(Filename);  
		FLinkerLoad::CreateActiveRedirectsMap(Filename);  
    }  
    
    FCoreDelegates::OnShutdownAfterError.AddStatic(StaticShutdownAfterError); 
    FCoreDelegates::OnExit.AddStatic(StaticExit);  
    
#if !USE_PER_MODULE_UOBJECT_BOOTSTRAP && !IS_MONOLITHIC  
// Not necessary when USE_PER_MODULE_UOBJECT_BOOTSTRAP==0 since the callback gets installed elsewhere  
// Also not necessary for monolithic builds as all pending registrants are available at once on app start    // so ProcessNewlyLoadedUObjects needs to only ever be invoked once, not for each module
        FModuleManager::Get().OnProcessLoadedObjectsCallback().AddStatic(ProcessNewlyLoadedUObjects);  
        
#endif  
    struct Local  
    {  
       static bool IsPackageLoaded( FName PackageName )  
       {          
	       return FindPackage( NULL, *PackageName.ToString() ) != NULL;  
       }    
    };    
	FModuleManager::Get().IsPackageLoadedCallback().BindStatic(Local::IsPackageLoaded);  
	FCoreDelegates::NewFileAddedDelegate.AddStatic(FLinkerLoad::OnNewFileAdded);  
	FCoreDelegates::GetOnPakFileMounted2().AddStatic(FLinkerLoad::OnPakFileMounted);  
	
	// Object initialization.  
	StaticUObjectInit();  
}
```

ProcessNewlyLoaded Objects is bound to OnProcessLoadedObjectsCallback 

```cpp
#if !USE_PER_MODULE_UOBJECT_BOOTSTRAP && !IS_MONOLITHIC  
    // Not necessary when USE_PER_MODULE_UOBJECT_BOOTSTRAP==0 since the callback gets installed elsewhere  
    // Also not necessary for monolithic builds as all pending registrants are available at once on app start 
    // so ProcessNewlyLoadedUObjects needs to only ever be invoked once, not for each module 
FModuleManager::Get().OnProcessLoadedObjectsCallback().AddStatic(ProcessNewlyLoadedUObjects); 
        
#endif
```

This callback is defined inside the Module Manager

```cpp
DECLARE_EVENT_TwoParams(FModuleManager, ProcessLoadedObjectsEvent, FName, bool);  
ProcessLoadedObjectsEvent& OnProcessLoadedObjectsCallback()  
{  
    return ProcessLoadedObjectsCallback;  
}
```

Following the diagram, you might be wondering why `ProcessLoadedObjects` is called twice. For `CoreUObject`, it's invoked once in `StartupModule` and a second time during `ProcessNewlyLoadedObjects`. The code snippets below guide us through `AppInit()` and the callback setup for `ProcessNewlyLoadedObjects`. This event facilitates the construction of a type object for the native class in the DLL after the new module is loaded.
#### StaticUObjectInit

Following the next step in the diagram we've arrived to StaticUObjectInit().

```cpp
void StaticUObjectInit() {
  UObjectBaseInit();

  // Allocate special packages.
  GObjTransientPkg = NewObject<UPackage>(nullptr, TEXT("/Engine/Transient"), RF_Transient);
  GObjTransientPkg->AddToRoot();

  if (IConsoleVariable* CVarVerifyGCAssumptions = IConsoleManager::Get().FindConsoleVariable(TEXT("gc.VerifyAssumptions"))) {
    if (FParse::Param(FCommandLine::Get(), TEXT("VERIFYGC"))) {
      CVarVerifyGCAssumptions->Set(true, ECVF_SetByCommandline);
    }
    if (FParse::Param(FCommandLine::Get(), TEXT("NOVERIFYGC"))) {
      CVarVerifyGCAssumptions->Set(false, ECVF_SetByCommandline);
    }
  }
  UE_LOG(LogInit, Log, TEXT("Object subsystem initialized"));
}
```

The `StaticUObjectInit` function invokes `UObjectInit`. Once initialization completes, a special transient package can be created using `NewObject`. `GObjTransientPkg` serves as a global variable designated for all objects lacking an Outer, ensuring they are assigned to this temporary package. This requirement enforces that every UObject must be contained within a `UPackage`. Furthermore, it's important to note that adding an object to the root within the transient package prevents it from being garbage collected.

#### UObjectBaseInit

```cpp
void UObjectBaseInit(){

	int32 MaxObjectsNotConsideredByGC = 0;  
	int32 SizeOfPermanentObjectPool = 0;  
	int32 MaxUObjects = 2 * 1024 * 1024; // Default to ~2M UObjects  
	bool bPreAllocateUObjectArray = false;

....

	GUObjectAllocator.AllocatePermanentObjectPool(SizeOfPermanentObjectPool);  
	GUObjectArray.AllocateObjectPool(MaxUObjects, MaxObjectsNotConsideredByGC, bPreAllocateUObjectArray);

	UObjectProcessRegistrants();
```
  
This marks the final phase of UObject initialization, incorporating the auto-registration of objects into main data structures. The function may allocate memory for a permanent object pool, utilized in cooked builds and non-GC UObject scenarios. Here, the object hash system is established, and `GConfig` values are read to configure garbage collection behavior. Additionally, the maximum allowable UObjects is defined, determining the size of the `UObject` pool. Moreover, an asynchronous loading thread is initiated for subsequent package loading, managed within `AsyncPackageLoader.cpp`.
##### UObjectProcessRegistrants

```cpp
static void UObjectProcessRegistrants()
{
	SCOPED_BOOT_TIMING("UObjectProcessRegistrants");

	check(UObjectInitialized());
	// Make list of all objects to be registered.
	TArray<FPendingRegistrant> PendingRegistrants;
	DequeuePendingAutoRegistrants(PendingRegistrants);

	for(int32 RegistrantIndex = 0;RegistrantIndex < PendingRegistrants.Num();++RegistrantIndex)
	{
		const FPendingRegistrant& PendingRegistrant = PendingRegistrants[RegistrantIndex];

		UObjectForceRegistration(PendingRegistrant.Object, false);

		check(PendingRegistrant.Object->GetClass()); // should have been set by DeferredRegister

		// Register may have resulted in new pending registrants being enqueued, so dequeue those.
		DequeuePendingAutoRegistrants(PendingRegistrants);
	}
}
```

Recall that `UClassRegisterAllCompiledInClasses` enqueues registrants into the global Pending Registrant Link List. `UObjectProcessRegistrants` traverses this link list and extracts registrants into an array during `DequeuePendingAutoRegistrants`. This dequeue process occurs twice because when a UObject is registered—such as when creating a CDO or loading a package—it may be referenced in other modules. This can trigger the loading of another module, adding more pending registrants to the link list. These additional registrants are also extracted into the array for registration.

```cpp
static void DequeuePendingAutoRegistrants(TArray<FPendingRegistrant>& OutPendingRegistrants)
{
	// We process registrations in the order they were enqueued, since each registrant ensures
	// its dependencies are enqueued before it enqueues itself.
	FPendingRegistrant* NextPendingRegistrant = GFirstPendingRegistrant;
	GFirstPendingRegistrant = NULL;
	GLastPendingRegistrant = NULL;
	while(NextPendingRegistrant)
	{
		FPendingRegistrant* PendingRegistrant = NextPendingRegistrant;
		OutPendingRegistrants.Add(*PendingRegistrant);
		NextPendingRegistrant = PendingRegistrant->NextAutoRegister;
		delete PendingRegistrant;
	};
}
```

#### UObjectForceRegistration

```cpp
void UObjectForceRegistration(UObjectBase* Object, bool bCheckForModuleRelease)
{
	LLM_SCOPE(ELLMTag::UObject);
	TMap<UObjectBase*, FPendingRegistrantInfo>& PendingRegistrants = FPendingRegistrantInfo::GetMap();

	FPendingRegistrantInfo* Info = PendingRegistrants.Find(Object);
	if (Info)
	{
		const TCHAR* PackageName = Info->PackageName;
#if USE_PER_MODULE_UOBJECT_BOOTSTRAP
		if (bCheckForModuleRelease)
		{
			UObjectReleaseModuleRegistrants(FName(PackageName));
		}
#endif
		const TCHAR* Name = Info->Name;
		PendingRegistrants.Remove(Object);  // delete this first so that it doesn't try to do it twice
		Object->DeferredRegister(UClass::StaticClass(),PackageName,Name);
	}
}
```

The function `UObjectForceRegistration` serves as a helper function, invoking deferred registration on objects that need to be registered and removing them from the pending registrant list. Additionally, `UObjectForceRegistration` can be called in two other scenarios.

**CreateDefaultSubObject**

```cpp
// Class.cpp
if ( ParentClass != NULL )  
{  
    UObjectForceRegistration(ParentClass);  
    ParentDefaultObject = ParentClass->GetDefaultObject(); // Force the default object to be constructed if it isn't already  
    check(GConfig);  
    if (GEventDrivenLoaderEnabled && EVENT_DRIVEN_ASYNC_LOAD_ACTIVE_AT_RUNTIME)  
    {check(ParentDefaultObject && !ParentDefaultObject->HasAnyFlags(RF_NeedLoad));  
    }}
```

it's call by the CDO to ensure that the base class associated with it is registered.

**ConstructUClass**

```cpp
// UObjectGlobals.cpp
if ( ParentClass != NULL )  
{  
    UObjectForceRegistration(ParentClass);  
    ParentDefaultObject = ParentClass->GetDefaultObject(); // Force the default object to be constructed if it isn't already  
    check(GConfig);  
    if (GEventDrivenLoaderEnabled && EVENT_DRIVEN_ASYNC_LOAD_ACTIVE_AT_RUNTIME)  
    {
	    check(ParentDefaultObject && !ParentDefaultObject->HasAnyFlags(RF_NeedLoad));  
    }}
....
```

NewClass object is passed to `UObjectForceRegistration` to determine if the type for this class has been registered. In essence, you can call `UObjectForceRegistration` and pass an object to check if that object is still in the pending registrants list.

#### Deferred Register

```cpp
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

This is the stage where an object is effectively registered. It's crucial to remember the distinction between deferred registration and `UObjectBase::Register()`. During deferred registration, GUObjectAllocator and GUObjectArray for the type system are not yet set up. Therefore, assigning or constructing a `NewObject` or `Package` is not possible at this point. However, functions defined by the UObject system can still be utilized.

Subsequent calls to `UObjectBase::Register()` are made by newly loaded modules after the type system has been allocated. These calls aim to allocate new memory for UObject types and add them to the global UObject array map.

During the deferred registration stage, all necessary initializations should be completed, allowing the creation of a Package. Once the package is created, it can be assigned as the `Outer` of UClass, set as `ClassPrivate`, and undergo additional steps such as calling `AddObject` to set internal flags, hashing the object, and assigning `NamePrivate`.

It's important to note that, during this stage, the properties and functions for UClass objects are not yet set up.

  
Several crucial stages have been accomplished:

1. The base type system is prepared, and the `ProcessNewlyLoadedUObjects` callback is configured to be invoked for subsequent loaded modules.
2. The memory allocation system, comprising GUObjectAllocator and GUObjectArray, is initialized.
3. Packages can be generated for items in `UObjectProcessRegistrants` and can be endowed with an `OuterPrivate` and `NamePrivate` before being added to the global object array.

While properties and functions have not been assigned to the UClass yet, the memory layout has been established for `OuterPrivate`, `ClassPrivate`, `SuperStruct`, and `NamePrivate`, as illustrated below.

##### SuperStruct & Class Private Tree

![[Unreal Engine Reflection System/Reflection Diagrams/SuperStructLayout.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/8945f68b9e6edf96ab6f9af87f93d9003b82a088/Unreal%20Engine%20Reflection%20System/Reflection%20Diagrams/SuperStructLayout.svg)
The `SuperStruct` plays a crucial role in establishing the inheritance relationships within the type tree of `UClass` objects. The relationships between types are determined by the `ClassPrivate` attribute.

![[Unreal Engine Reflection System/Reflection Diagrams/ClassPrivate.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/8945f68b9e6edf96ab6f9af87f93d9003b82a088/Unreal%20Engine%20Reflection%20System/Reflection%20Diagrams/ClassPrivate.svg)

- When `UClass::StaticClass()` is invoked, it generates the `UClass` object, and the `ClassPrivate` attribute points to itself.
- Type relationships are managed through `UClass`, as discussed earlier.
- For all UObject-derived objects within `UPackage`, the outer object typically points to a `UPackage`, thereby forming a dependency tree. It's important to distinguish this from an `AActor`'s owner, which is specific to actors.

For a `UObjectBase` the memory layout relationships are defined by these four variables:

1. **NamePrivate** : defines the name of the object
2. **OuterPrivate** : defines the affiliation of the object
3. **ClassPrivate** : defines the type relationship of the object
4. **SuperStruct** : defines the inheritance relationship of the type

#### ProcessNewlyLoadedUObjects

For all `UPackage` objects, the outer of an object is usually a UPackage, forming the dependency tree. It's important to note that this is distinct from an `AActor` owner defined for an actor.

```cpp
void ProcessNewlyLoadedUObjects(FName Package, bool bCanProcessNewlyLoadedObjects)
{
	SCOPED_BOOT_TIMING("ProcessNewlyLoadedUObjects");
#if USE_PER_MODULE_UOBJECT_BOOTSTRAP
	if (Package != NAME_None)
	{
		UObjectReleaseModuleRegistrants(Package);
	}
#endif
	if (!bCanProcessNewlyLoadedObjects)
	{
		return;
	}
	LLM_SCOPE(ELLMTag::UObject);
	DECLARE_SCOPE_CYCLE_COUNTER(TEXT("ProcessNewlyLoadedUObjects"), STAT_ProcessNewlyLoadedUObjects, STATGROUP_ObjectVerbose);

	FPackageDeferredRegistry& PackageRegistry = FPackageDeferredRegistry::Get();
	FClassDeferredRegistry& ClassRegistry = FClassDeferredRegistry::Get();
	FStructDeferredRegistry& StructRegistry = FStructDeferredRegistry::Get();
	FEnumDeferredRegistry& EnumRegistry = FEnumDeferredRegistry::Get();

	PackageRegistry.ProcessChangedObjects(true);
	StructRegistry.ProcessChangedObjects();
	EnumRegistry.ProcessChangedObjects();

	UClassRegisterAllCompiledInClasses();

	bool bNewUObjects = false;
	TArray<UClass*> AllNewClasses;
	while (GFirstPendingRegistrant ||
		ClassRegistry.HasPendingRegistrations() ||
		StructRegistry.HasPendingRegistrations() ||
		EnumRegistry.HasPendingRegistrations())
	{
		bNewUObjects = true;
		UObjectProcessRegistrants();
		UObjectLoadAllCompiledInStructs();

		FCoreUObjectDelegates::CompiledInUObjectsRegisteredDelegate.Broadcast(Package);

		UObjectLoadAllCompiledInDefaultProperties(AllNewClasses);
	}

	PackageRegistry.EmptyRegistrations();
	EnumRegistry.EmptyRegistrations();
	StructRegistry.EmptyRegistrations();
	ClassRegistry.EmptyRegistrations();

	if (TMap<UObjectBase*, FPendingRegistrantInfo>& PendingRegistrants = FPendingRegistrantInfo::GetMap(); PendingRegistrants.IsEmpty())
	{
		PendingRegistrants.Empty();
	}

	if (bNewUObjects && !GIsInitialLoad)
	{
		for (UClass* Class : AllNewClasses)
		{
			// Assemble reference token stream for garbage collection/ RTGC.
			if (!Class->HasAnyFlags(RF_ClassDefaultObject) && !Class->HasAnyClassFlags(CLASS_TokenStreamAssembled))
			{
				Class->AssembleReferenceTokenStream();
			}
		}
	}
}

```

- The` Deferred Registry` for packages, enums, structs, and classes are global arrays, populated during the static allocation phase of type information.
- `UClassRegisterAllCompiledInClasses` calls `TClass::StaticClass()` for each compiled class to construct the UClass* object.
- The while loop iterates through all pending registrants allocated previously; thus, a call to `UObjectProcessRegistrants` ensures all relevant UClass* objects are registered in memory before proceeding.
- `UObjectLoadAllCompiledInStructs` is called to generate UEnum and UScriptStructs for enums and structs respectively.
- `UObjectLoadAllCompiledDefaultProperties` is where the CDO (CreateDefaultObject) for UClass* is created.
- Finally, determine if the new UClass* has been generated and is not in an initial loading stage. This is determined by `GIsInitialLoad`, which is true only after Garbage Collection is turned on. Otherwise, if it's false, the initial loading process is over. The reference token for `UClass` stream, which is a data structure used for how GC analyzes object references, is also handled here.
- Since `ProcessNewlyLoadedUObjects` is triggered after subsequent modules are loaded, `AssembleReferenceTokenStreams` will only be called once for each class in the class hierarchy, with flags set to avoid duplicate work.
#### UObjectLoadAllCompiledInStructs

```cpp
/**  
 * Call StaticStruct for each struct...this sets up the internal singleton, and important works correctly with hot reload */
static void UObjectLoadAllCompiledInStructs()  
{  
    SCOPED_BOOT_TIMING("UObjectLoadAllCompiledInStructs");  
  
    FEnumDeferredRegistry& EnumRegistry = FEnumDeferredRegistry::Get();  
    FStructDeferredRegistry& StructRegistry = FStructDeferredRegistry::Get();  
  
    { SCOPED_BOOT_TIMING("UObjectLoadAllCompiledInStructs -  CreatePackages (could be optimized!)");  
       EnumRegistry.DoPendingPackageRegistrations();  
       StructRegistry.DoPendingPackageRegistrations();  
    }  
    // Load Structs  
    EnumRegistry.DoPendingOuterRegistrations(true);  
    StructRegistry.DoPendingOuterRegistrations(true);  
}
```

The `EnumRegistry` and `StructRegistry` getters are called once more to ensure the UClass* is set up. Then there are a couple of steps here to create the Outer Packages.

1. `CreatePackage` is called for all the registrations of Enum and Struct, checking if the associated package name doesn't already exist to ensure no duplication occurs.
2. Calls are made to the Outer Registrant, which will invoke `StaticStruct` and `StaticEnum` from the gen.cpp file, filling the `OuterSingleton`.
3. The order here matters! Enum is first for a reason, as it is the most basic type and is constructed first. Structs can contain Enum variables, so when constructing a `UScriptStruct`, it's clear that Enum is available first if it needs to be nested inside a struct type.
  
You might wonder, what if a struct contains a UClass variable? 

In this situation, because UClass objects, for which each class belongs to, have already been registered during `UObjectProcessRegistrants`, it means that all object reference types only find a UClass through the Class name. UClass is not registered yet, so it doesn't matter if the structure is complete; it just needs to find it by Name.
#### UObjectLoadAllCompiledInDefaultProperties

```cpp
/**
 * Load any outstanding compiled in default properties
 */
static void UObjectLoadAllCompiledInDefaultProperties(TArray<UClass*>& OutAllNewClasses)
{
    TRACE_LOADTIME_REQUEST_GROUP_SCOPE(TEXT("UObjectLoadAllCompiledInDefaultProperties"));

    static FName LongEnginePackageName(TEXT("/Script/Engine"));

    FClassDeferredRegistry& ClassRegistry = FClassDeferredRegistry::Get();

    if (ClassRegistry.HasPendingRegistrations())
    {
        SCOPED_BOOT_TIMING("UObjectLoadAllCompiledInDefaultProperties");
        TArray<UClass*> NewClasses;
        TArray<UClass*> NewClassesInCoreUObject;
        TArray<UClass*> NewClassesInEngine;

        ClassRegistry.DoPendingOuterRegistrations(true, [&OutAllNewClasses, &NewClasses, &NewClassesInCoreUObject, &NewClassesInEngine](const TCHAR* PackageName, UClass& Class) -> void
        {
            UE_LOG(LogUObjectBootstrap, Verbose, TEXT("UObjectLoadAllCompiledInDefaultProperties After Registrant %s %s"), PackageName, *Class.GetName());

            if (Class.GetOutermost()->GetFName() == GLongCoreUObjectPackageName)
            {
                NewClassesInCoreUObject.Add(&Class);
            }
            else if (Class.GetOutermost()->GetFName() == LongEnginePackageName)
            {
                NewClassesInEngine.Add(&Class);
            }
            else
            {
                NewClasses.Add(&Class);
            }
            OutAllNewClasses.Add(&Class);
        });

        auto NotifyClassFinishedRegistrationEvents = [](TArray<UClass*>& Classes)
        {
            for (UClass* Class : Classes)
            {
                TCHAR PackageName[FName::StringBufferSize];
                TCHAR ClassName[FName::StringBufferSize];
                Class->GetOutermost()->GetFName().ToString(PackageName);
                Class->GetFName().ToString(ClassName);
                NotifyRegistrationEvent(PackageName, ClassName, ENotifyRegistrationType::NRT_Class, ENotifyRegistrationPhase::NRP_Finished, nullptr, false, Class);
            }
        };

        // notify async loader of all new classes before creating the class default objects
        {
            SCOPED_BOOT_TIMING("NotifyClassFinishedRegistrationEvents");
            NotifyClassFinishedRegistrationEvents(NewClassesInCoreUObject);
            NotifyClassFinishedRegistrationEvents(NewClassesInEngine);
            NotifyClassFinishedRegistrationEvents(NewClasses);
        }

        {
            SCOPED_BOOT_TIMING("CoreUObject Classes");
            for (UClass* Class : NewClassesInCoreUObject) // we do these first because we assume these never trigger loads
            {
                UE_LOG(LogUObjectBootstrap, Verbose, TEXT("GetDefaultObject Begin %s %s"), *Class->GetOutermost()->GetName(), *Class->GetName());
                Class->GetDefaultObject();
                UE_LOG(LogUObjectBootstrap, Verbose, TEXT("GetDefaultObject End %s %s"), *Class->GetOutermost()->GetName(), *Class->GetName());
            }
        }
        {
            SCOPED_BOOT_TIMING("Engine Classes");
            for (UClass* Class : NewClassesInEngine) // we do these second because we want to bring the engine up before the game
            {
                UE_LOG(LogUObjectBootstrap, Verbose, TEXT("GetDefaultObject Begin %s %s"), *Class->GetOutermost()->GetName(), *Class->GetName());
                Class->GetDefaultObject();
                UE_LOG(LogUObjectBootstrap, Verbose, TEXT("GetDefaultObject End %s %s"), *Class->GetOutermost()->GetName(), *Class->GetName());
            }
        }
        {
            SCOPED_BOOT_TIMING("Other Classes");
            for (UClass* Class : NewClasses)
            {
                UE_LOG(LogUObjectBootstrap, Verbose, TEXT("GetDefaultObject Begin %s %s"), *Class->GetOutermost()->GetName(), *Class->GetName());
                Class->GetDefaultObject();
                UE_LOG(LogUObjectBootstrap, Verbose, TEXT("GetDefaultObject End %s %s"), *Class->GetOutermost()->GetName(), *Class->GetName());
            }
        }

        FFeedbackContext& ErrorsFC = UClass::GetDefaultPropertiesFeedbackContext();
        if (ErrorsFC.GetNumErrors() || ErrorsFC.GetNumWarnings())
        {
            TArray<FString> AllErrorsAndWarnings;
            ErrorsFC.GetErrorsAndWarningsAndEmpty(AllErrorsAndWarnings);

            FString AllInOne;
            UE_LOG(LogUObjectBase, Warning, TEXT("-------------- Default Property warnings and errors:"));
            for (const FString& ErrorOrWarning : AllErrorsAndWarnings)
            {
                UE_LOG(LogUObjectBase, Warning, TEXT("%s"), *ErrorOrWarning);
                AllInOne += ErrorOrWarning;
                AllInOne += TEXT("\n");
            }
            FMessageDialog::Open(EAppMsgType::Ok, FText::Format(NSLOCTEXT("Core", "DefaultPropertyWarningAndErrors", "Default Property warnings and errors:\n{0}"), FText::FromString(AllInOne)));
        }
    }
}
```

This is where the construction of `UClass*` objects is performed, following these steps:

1. The `ClassRegistry` getter retrieves elements from the global deferred object array, statically initialized for class types.
2. If there are pending registrations, three arrays for class subtypes (CoreUObject, Engine, Other) are created.
3. `DoPendingOuterRegistrations` iterates through objects, calling `Z_Construct_UClass` functions. A lambda function captures `TArray` types and processes `OuterRegistrant` results into subarrays, filling `OutAllNewClasses`.
4. Events are fired for default object creation preparation, ensuring `async` loader dependency order: `CoreUObject` first, followed by Engine, then others. `CoreUObject` is the bottom layer (without references to other packages), Engine is the second (may reference underlying objects), and others are uncertain about references. This order prevents dependency inversion and reduces search calls.
5. CDOs are created for each `UClass` type.
6. `UObjectLoadAllCompiledInDefaultProperties` follows `UClass` object construction, potentially containing nested `UEnum*` or `UScriptStruct` objects.

### Post Initialization

##### CloseDisregardForGC

Inside `InitCoreUObject` after `ProcessNewlyLoadedObjects` is called

```cpp
//CoreUObjectUtilities.cpp
FGCObject::StaticInit();  
if (GUObjectArray.IsOpenForDisregardForGC())  
{  
    GUObjectArray.CloseDisregardForGC();  
}
```

After `CloseDisregardForGC`, the construction of `UClass*` is complete, and the garbage collector can be enabled. CDOs are configured, and package objects are set up for the core components of the engine.

According to Dazhao:

*"These necessary objects are only destroyed when the game is launched; hence, they are not managed by the GC. Therefore, GC is turned off in the beginning (`OpenForDisregardForGc = True`), and after the type system is built, GC can be enabled since `NewObject` may be used to generate objects later."*

At this point, various types of objects have been constructed in memory, and type information has been collected. The next step involves finalizing the construction (`Z_Construct`) for these common types and binding/linking the objects together.
#### Binding & Linking

The final step involves sorting and additional post-initialization work after constructing the type objects. The subsequent sections will delve into the binding and linking process.
##### Bind
 
The purpose of binding is to associate a function pointer with the correct address.

The `Bind` operation is defined as a virtual function inside an `FField`. Consequently, all fields may need to overload this binding operation. However, the two types that specifically override this function are `UClass` and `UFunction`.

```cpp
void UFunction::Bind()
{
    UClass* OwnerClass = GetOwnerClass();

    // If this isn't a native function, or this function belongs to a native interface class (which has no C++ version),
    // use ProcessInternal (call into script VM only) as the function pointer for this function
    if (!HasAnyFunctionFlags(FUNC_Native))
    {
        // Use processing function.
        Func = &UObject::ProcessInternal;
    }
    else
    {
        // Find the function in the class's native function lookup table.
        FName Name = GetFName();
        FNativeFunctionLookup* Found = OwnerClass->NativeFunctionLookupTable.FindByPredicate([=](const FNativeFunctionLookup& NativeFunctionLookup) { return Name == NativeFunctionLookup.Name; });
        if (Found)
        {
            Func = Found->Pointer;
        }
#if USE_COMPILED_IN_NATIVES
        else if (!HasAnyFunctionFlags(FUNC_NetRequest))
        {
            UE_LOG(LogClass, Warning, TEXT("Failed to bind native function %s.%s"), *OwnerClass->GetName(), *GetName());
        }
#endif
    }
}
```

Examining the code for `UFunction::Bind`

- If there isn't a native function or if the function belongs to an interface, a call to `ProcessInternal` is made through the `DECLARE_FUNCTION` macro, returning the address of the function.
- Otherwise, it searches for a native function inside the `Native Function Lookup `table and assigns the function pointer accordingly.

So simply find and bind the correct function pointer.

```cpp
/**
 * Find the class's native constructor.
 */
void UClass::Bind()
{
    UStruct::Bind();

    if (!ClassConstructor && IsNative())
    {
        UE_LOG(LogClass, Fatal, TEXT("Can't bind to native class %s"), *GetPathName());
    }

    UClass* SuperClass = GetSuperClass();
    if (SuperClass && (ClassConstructor == nullptr || !CppClassStaticFunctions.IsInitialized() || ClassVTableHelperCtorCaller == nullptr))
    {
        // Chase down constructor in parent class.
        SuperClass->Bind();
        if (!ClassConstructor)
        {
            ClassConstructor = SuperClass->ClassConstructor;
        }
        if (!ClassVTableHelperCtorCaller)
        {
            ClassVTableHelperCtorCaller = SuperClass->ClassVTableHelperCtorCaller;
        }
        if (!CppClassStaticFunctions.IsInitialized())
        {
            CppClassStaticFunctions = SuperClass->CppClassStaticFunctions;
        }

        // propagate flags.
        // we don't propagate the inherit flags, that is more of a header generator thing
        ClassCastFlags |= SuperClass->ClassCastFlags;
    }

    // if( !Class && SuperClass )
    //{
    //}
    if (!ClassConstructor)
    {
        UE_LOG(LogClass, Fatal, TEXT("Can't find ClassConstructor for class %s"), *GetPathName());
    }
}
```

The binding of `UClass` is crucial during blueprint compilation and when loading a class within a package. If we recall, the function pointer for construction is already provided for native classes through `GetPrivateStaticClassBody`. Therefore, for classes lacking C++ code, the constructor needs to be bound in the base class to ensure proper functionality. In the source code, there are three binding functions, essentially identical, which are passed to `GetPrivateStaticClassBody`.
##### Link

Finally, the last step in constructing `UScriptStruct` and `UClass` occurs when `StaticLink` is called. To understand this process at a lower level, it's essential to know how the serialization system operates. A wrapped empty serialized archive class object is forwarded to the `UStruct::Link` function during this step.

```cpp
void UStruct::StaticLink(bool bRelinkExistingProperties)  
{  
    FNullArchive ArDummy;  
    Link(ArDummy, bRelinkExistingProperties);  
}
```

In the Unreal reflection system, the term `link` is analogous to the linking phase in regular C++, encompassing several meanings:

1. **Symbol Address Linking:** This involves linking and replacing symbol addresses. It typically occurs after structural changes or compilation to update references to functions or variables.
    
2. **Reference Links (RefLink):** Refers to the process of dividing subfields into chains based on attribute characteristics. This helps in efficiently managing and accessing attributes within data structures.
    
3. **Serialization Linking:** During serialization, linking is used to establish mappings between objects stored on disk and their corresponding representations in memory. This ensures that serialized data can be correctly loaded and interpreted by the engine.
    
4. **Post-Serialization Linking:** After an object is serialized and loaded into memory, post-serialization linking deals with attribute offsets and memory alignment. This phase uses FArchive to handle the final setup of objects for runtime use.
    

Over time, changes to the reflection system in Unreal Engine have reduced the reliance on `UProperty`, though it remains essential for legacy blueprint compatibility. However, the linking process for both `UProperty` and `FProperty` follows similar procedures. Before discussing `UStruct::Link`, it's crucial to understand the linking process for properties, which includes phases like `LinkInternal` and `SetupOffset`.

###### Property Link

Inside UnrealTypes.cpp this is the base class type definition for `LinkInternal`

```cpp
//  
// Link property loaded from file.  
//  
void FProperty::LinkInternal(FArchive& Ar)  
{  
    check(0); 
	// Link shouldn't call super...and we should never link an abstract property, like this base class  
}
```

  
The `check(0)` macro serves as a sanity check to prevent bugs and potential crashes in case the `LinkInternal` is not correctly overridden by derived types utilizing the `FProperty` class.

Example of what a derived class using the `Link` Internal is seen here.

```cpp
void FBoolProperty::LinkInternal(FArchive& Ar)
{
    check(FieldSize != 0);
    ElementSize = FieldSize;

    if (IsNativeBool())
    {
        PropertyFlags |= (CPF_IsPlainOldData | CPF_NoDestructor | CPF_ZeroConstructor);
    }
    else
    {
        PropertyFlags &= ~(CPF_IsPlainOldData | CPF_ZeroConstructor);
        PropertyFlags |= CPF_NoDestructor;
    }

    PropertyFlags |= CPF_HasGetValueTypeHash;
}
```

It can be observed that  the `LinkInternal` function is utilized to configure the property flags based on attribute characteristics.
 
```cpp
int32 FProperty::SetupOffset()
{
    UObject* OwnerUObject = GetOwner<UObject>();
    if (OwnerUObject && (OwnerUObject->GetClass()->ClassCastFlags & CASTCLASS_UStruct))
    {
        UStruct* OwnerStruct = (UStruct*)OwnerUObject;
        Offset_Internal = Align(OwnerStruct->GetPropertiesSize(), GetMinAlignment());
    }
    else
    {
        Offset_Internal = Align(0, GetMinAlignment());
    }

    uint32 UnsignedTotal = (uint32)Offset_Internal + (uint32)GetSize();
    if (UnsignedTotal >= (uint32)MAX_int32)
    {
        UE::CoreUObject::Private::OnInvalidPropertySize(UnsignedTotal, this);
    }
    return (int32)UnsignedTotal;
}
```

The `SetupOffset` function precisely defines its purpose, which is to establish the memory offsets for attributes after serialization.

I've extracted the key segments of `UStruct::Link` to concentrate on its main components.

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

The steps in the linking process for a `UFunction`, which is a subtype of UStruct, involve invoking `InitializeDerivedMembers` to calculate the information offsets of parameters and return values, as illustrated below:

1. **Internal Link:** This step initializes and links internal properties within the UFunction. It ensures that properties such as parameters and return values are correctly set up for runtime use.
    
2. **RefLink:** Properties that contain object references are processed during this phase. RefLink is crucial for efficient garbage collection, as it establishes relationships between objects and manages their lifetimes appropriately.
    
3. **PostConstructLink:** This step retrieves original default values from the Class Default Object (CDO) for the function. It ensures that function attributes are initialized correctly after serialization, using values from configuration files or the CDO.
    
4. **DestructorLink:** During object destruction, properties requiring additional cleanup are identified and processed by `DestructorLink`. This ensures that resources associated with function attributes are properly released.
    

These linking steps optimize performance in various application scenarios by minimizing the need to traverse `FProperty` attributes repeatedly. Each phase plays a critical role in preparing UFunction objects for efficient execution and resource management within the Unreal Engine framework.

```cpp
void UFunction::InitializeDerivedMembers()
{
    NumParms = 0;
    ParmsSize = 0;
    ReturnValueOffset = MAX_uint16;
    FProperty** ConstructLink = &FirstPropertyToInit;

    for (FProperty* Property = CastField<FProperty>(ChildProperties); Property; Property = CastField<FProperty>(Property->Next))
    {
        if (Property->PropertyFlags & CPF_Parm)
        {
            NumParms++;
            ParmsSize = IntCastChecked<uint16>(Property->GetOffset_ForUFunction() + Property->GetSize());
            if (Property->PropertyFlags & CPF_ReturnParm)
            {
                ReturnValueOffset = IntCastChecked<uint16>(Property->GetOffset_ForUFunction());
            }
        }
        else if ((FunctionFlags & FUNC_HasDefaults) == 0)
        {
            // we're done with parms and we've not been tagged as FUNC_HasDefaults, so we can abort
            // this potentially costly loop:
            break;
        }
        else if (!Property->HasAnyPropertyFlags(CPF_ZeroConstructor))
        {
            *ConstructLink = Property;
            Property->PostConstructLinkNext = nullptr;
            ConstructLink = &Property->PostConstructLinkNext;
        }
    }
}
```

####  UMetaData

The `UMetaData` type is only used in Editor mode. However, those exploring the Reflection system in Unreal Engine might want to add functionality to the engine and integrate new tools or systems.

Below are getters and setters for a UMetaData Type:

```cpp
#if WITH_METADATA  
void AddMetaData(UObject* Object, const FMetaDataPairParam* MetaDataArray, int32 NumMetaData)  
{       
    if (NumMetaData)  
    {          
        UMetaData* MetaData = Object->GetOutermost()->GetMetaData();  
        for (const FMetaDataPairParam* MetaDataParam = MetaDataArray, *MetaDataParamEnd = MetaDataParam + NumMetaData; MetaDataParam != MetaDataParamEnd; ++MetaDataParam)  
        {             
            MetaData->SetValue(Object, UTF8_TO_TCHAR(MetaDataParam->NameUTF8), UTF8_TO_TCHAR(MetaDataParam->ValueUTF8));  
        }       
    }   
}  
#endif
```

```cpp
/**
 * Gets (after possibly creating) a metadata object for this package
 *
 * @return A valid UMetaData pointer for all objects in this package
 */
UMetaData* UPackage::GetMetaData()
{
    checkf(!FPlatformProperties::RequiresCookedData(), TEXT("MetaData is only allowed in the Editor."));

#if WITH_EDITORONLY_DATA
    PRAGMA_DISABLE_DEPRECATION_WARNINGS
    UMetaData* LocalMetaData = MetaData;
    PRAGMA_ENABLE_DEPRECATION_WARNINGS

    // If there is no MetaData, try to find it.
    if (LocalMetaData == nullptr)
    {
        LocalMetaData = FindObjectFast<UMetaData>(this, FName(NAME_PackageMetaData));

        // If MetaData is null then it wasn't loaded by linker, so we have to create it.
        if (LocalMetaData == nullptr)
        {
            LocalMetaData = NewObject<UMetaData>(this, NAME_PackageMetaData, RF_Standalone | RF_LoadCompleted);
        }
        SetMetaData(LocalMetaData);
    }
    check(LocalMetaData);

    if (LocalMetaData->HasAnyFlags(RF_NeedLoad))
    {
        FLinkerLoad* MetaDataLinker = LocalMetaData->GetLinker();
        check(MetaDataLinker);
        MetaDataLinker->Preload(LocalMetaData);
    }
    return LocalMetaData;
#else
    return nullptr;
#endif
}

```

From digging into the source, UMetaData is a UObject associated with a UPackage but not bound to an `FField`.

```cpp
/**  
 * An object that holds a map of key/value pairs. */  
class UMetaData : public UObject  
{  
    DECLARE_CASTED_CLASS_INTRINSIC_WITH_API(UMetaData, UObject, CLASS_MatchedSerializers, TEXT("/Script/CoreUObject"), CASTCLASS_None, COREUOBJECT_API);  
  
public:  
    /**  
     * Mapping between an object, and its key->value meta-data pairs.     */  
    TMap< FWeakObjectPtr, TMap<FName, FString> > ObjectMetaDataMap;  
  
    /**  
     * Root-level (not associated with a particular object) key->value meta-data pairs.     * Meta-data associated with the package itself should be stored here. */   
	 TMap< FName, FString > RootMetaDataMap;
```

Notice the macro `DECLARE_CASTED_CLASS_INTRINIC_WITH_API`, which drills down to the macro `DECLARE_CLASS` for class declarations and serialization functions. This follows the same layout as `DECLARE_CLASS_INTRINSIC`.

```cpp
const FString& FField::GetMetaData(const FName& Key) const  
{  
    // If not found, return a static empty string  
    static FString EmptyString;  
  
    // Every key needs to be valid, and meta data needs to exist  
    if (Key == NAME_None || !MetaDataMap)  
    {  
        return EmptyString;  
    }  
  
    // Look for the property  
    const FString* ValuePtr = MetaDataMap->Find(Key);  
  
    // If we didn't find it, return NULL  
    if (!ValuePtr)  
    {  
        return EmptyString;  
    }  
  
    // If we found it, return the pointer to the character data  
    return *ValuePtr;  
}
```

`ObjectMetaData` utilizes an `FWeakObjectPtr` to reference an existing UObject, which is commonly used to refer to a UObject in the `GUObjectAllocator` array.

The question arises: why isn't this MetaDataMap added directly to the UObject?

According to Dazhao:

It aligns with the concept of decoupling from UObject and avoids unnecessary bloat by introducing another field. Embedding this might lead to certain disadvantages:

- The memory footprint of a UObject increases, regardless of whether the metadata is used.
- Tight coupling occurs because the metadata information is exclusively intended for the editor and should be excluded during cooking. This contrasts with having `MetaDataMap` as an independent object that can be saved in a file and is easier to deconstruct. If tightly coupled, it becomes challenging to extract it from a binary stream.

Additionally, Dazhao mentions the use case for `UMetaData`.

*"removing it will create an additional layer of indirect access, which will increase the efficiency burden and the opportunity for a CacheMiss. But judging from the situation, UMetaData can only be used in the editor. It doesn't matter if the editor is a little slower. There are no frame rate requirements like games. And the frequency of accessing UMetaData is not high. It is only obtained when initializing the interface to change the UI. UMetaData is redesigned so that UPackage is associated (Outer is UPackage), and UPackage is the unit of serialized storage, so that UMetaData can be loaded or released as an independent part."*
