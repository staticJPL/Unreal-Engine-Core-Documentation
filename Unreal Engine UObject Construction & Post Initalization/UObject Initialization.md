### UObject Initialization

The UObject Initialization phase begins in `AppInit()` with the following call stack order:

1. **InitUObject**: Sets up initialization and processes the `ProcessNewLoadedUObjects` delegate.
    
2. **StaticUObjectInit**: Calls `UObjectBaseInit` to initialize the garbage collector with a transient package.
    
3. **UObjectBaseInit**: Allocates a global object array pool in memory and confirms that the UObject system is initialized.
    
4. **UObjectProcessRegistrants**: Traverses a pending registrant link list and passes registrants to `UObjectForceRegistration`.
    
5. **UObjectForceRegistration**: Retrieves a Registrant Object and calls `DeferredRegister` with parameters such as `UClass::StaticClass`, `PackageName`, and Name.