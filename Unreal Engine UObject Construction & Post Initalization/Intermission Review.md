#### Intermission Review

Before delving into the Post Construction Initialization process, there are several concepts I needed to grasp for a better understanding of how sub-objects and properties are instantiated and initialized. Additionally, topics such as Serialization and the usage of FProperty will be discussed.

Let's quickly review what has been covered so far to recall the concepts and their relevance to the upcoming discussion on post-constructor initialization:

**Reflection**: UHT generates and inserts code defined by macros to gather type information and construct a UClass. During post-initialization, UClass's FProperties are traversed for initialization, overwriting, or reading.

**UObject Architecture**: During the registration phase, we observed the construction process of native types and how UObjects are statically allocated in memory and registered into a global UObject memory pool.

**NewObject**: Once the UObject system is initialized, NewObject becomes available for instantiating new objects. This is where the default constructor is called, and where FObjectInitializer may first be used.

**Template Objects**: Template objects follow a prototype design pattern to instantiate objects (via Archetypes or CDOs). These objects determine default values of properties or sub-objects. The InstanceGraph loads template objects into a graph structure and maps them to their instantiated objects, enabling modification or copying.

**Create Default Object**: Towards the end of the registration phase, ProcessNewlyLoadedObjects creates CDOs for all UObjects within the loaded module. The CDO is associated with its UClass and defines the default state from which objects are instantiated. The CDO often serves as the template object during post-initialization.

**FObjectInitializer**: Following a builder design pattern, FObjectInitializer provides flexibility in the engine's construction process while separating the representation of the object being constructed. FObjectInitializer is pushed to a thread-local storage stack array for every constructor call. Upon leaving scope from the most derived class, the destructor recursively pops the initializer from the stack array and executes post-initialization work. Depending on the FObjectInitializer function signature, different post-initialization behaviors can be observed.