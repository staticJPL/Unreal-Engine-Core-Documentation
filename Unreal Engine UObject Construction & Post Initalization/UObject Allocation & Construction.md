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