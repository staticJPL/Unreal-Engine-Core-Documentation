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

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/Constructor Macros.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/1992221c656a4b68d79513732f0c302c2feaa9ff/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/Constructor%20Macros.png)
