# Generation

#### Reflection and Type Systems

"Reflection is a system used to acquire type information at runtime. This type information conveys static details associated with an object. Garbage collection will still operate effectively even without reflection for creating class objects. The type system can be defined as the process of obtaining the type at runtime, and an object can be created in reverse by utilizing the type information to read or modify attributes. Currently, C++ does not support a reflection type system (except for C++ RTTI), while C# does."

Example of C# Reflection and important points.

![[Unreal Engine Reflection System/Reflection Diagrams/AssemlyExample.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/ac3c872774ee20ecbf2652512b4ce7ea6e214ee5/Unreal%20Engine%20Reflection%20System/Reflection%20Diagrams/AssemlyExample.png)

1. **Assembly:** Refers to a DLL, usually associated with assembly.
    
2. **Module:** Represents the sub-module division inside an assembly.
    
3. **Type:** Describes the Class object, providing comprehensive information about an object's type. Types can establish relationships with each other through attributes such as BaseType and DeclaringType.
    
4. **ConstructorInfo:** Describes the constructors within the Type, allowing the invocation of specific constructors.
    
5. **EventInfo:** Describes events defined in the Type, analogous to delegates in Unreal Engine.
    
6. **FieldInfo:** Describes fields in Type, equivalent to C++ member variables, enabling dynamic reading and modification of values.
    
7. **PropertyInfo:** Describes properties in Type, similar to the combination of get/set methods in C++. After obtaining it, you can access the property values.
    
8. **MethodInfo:** Describes methods in Type. Once obtained, methods can be dynamically invoked.
    
9. **ParameterInfo:** Describes each parameter in a method.
    
10. **Attributes:** Denote additional features on the Type, not present in C++. They can be understood as metadata information defined on the class.

##### Dynamic_Cast in C++

The `dynamic_cast` operator is used to convert a base class pointer or reference pointing to a derived class into a pointer or reference of the derived class. This operation is only applicable to classes containing virtual functions. If the reference conversion fails, a `bad_cast` exception will be thrown, and if the pointer conversion fails, a null value will be returned.

Internally, the `dynamic_cast` operator utilizes the type information stored in the virtual function table to determine whether a base class pointer points to an object of the derived class. Its primary purpose is to ascertain, at runtime, whether an object pointer is an instance of a specific subclass.

##### UHT Solution in Unreal Engine

The Unreal Header Tool (UHT) serves as the current solution in Unreal Engine.

In the early days Unreal Engine had to implement their own custom reflection system with C++. In the later versions the Unreal Header Tool logic was moved to the C# language because of better library support.

The key advantage with this design is the ability to introduce relatively small changes to the C++ code by simply adding empty tag macros. This allows programmers to seamlessly associate metadata and code without disrupting the original class declaration structure.

To better understand the inspiration behind the Unreal Header Tool the image illustrates the `Clang Compilation Driver`. In a nutshell the compiler driver parses CPP files into `Tokens` which is part of `Lexical Analysis` process. After which the Abstract Syntax Tree is traversed and the source code is generated into "LLVM intermediate representation" to be later compiled with a the tool chain of a specific platform.

![[Unreal Engine Reflection System/Reflection Diagrams/LLVMExample.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/a828c7cea077bfda510380dbca8773c8544b3562/Unreal%20Engine%20Reflection%20System/Reflection%20Diagrams/LLVMExample.png)

Without going too deep into Compiler Theory a typical lexical analyzer architecture looks like the following image below.


![[Unreal Engine Reflection System/Reflection Diagrams/LexicalExample.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/a828c7cea077bfda510380dbca8773c8544b3562/Unreal%20Engine%20Reflection%20System/Reflection%20Diagrams/LexicalExample.png)

**Token:** It is a sequence of characters that represents a unit of information in the source code.

**Pattern:** The description used by the token is known as a pattern.

**Lexeme**: A sequence of characters in the source code, as per the matching pattern of a token, is known as lexeme. It is also called the instance of a token.

Now with these concepts in mind we can being to dig into the basics Unreal Engine Tools 

### UHT

![[Unreal Engine Reflection System/Reflection Diagrams/UHTProcessEx 1.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/a828c7cea077bfda510380dbca8773c8544b3562/Unreal%20Engine%20Reflection%20System/Reflection%20Diagrams/UHTProcessEx%201.png)

The execute path for UHT is generalized into these steps.

1. Early on when UBT creates a make file for a module, the make file is passed to the UHT to create a manifest file. This holds information about a modules (`BaseDirectory`, `OutputDirectory`, `ClassHeaders`, `Public Headers` etc)
2. All modules hold an absolute path to the stored "`Classes Folde`r",  similarly `Public Folders` are placed inside "`PublicUObjectHeaders`. Following the same premise Private is also "`PrivateUObjectHeaders`" folder.
3. After the modules have been prepared, UHT creates a `UPackage` to mirror the same package outer hierarchy the engine uses. This encapsulates UObjects with it `Outer`.  At this point the  "Headers are prepared for the Parser"
4. Unreal Lexar parsers tokenizes string data into symbols and Identifiers. The Tokenizer Scopes out data structures from the start of  "{   to   };" parsing anything between the bracket symbols. The header parser delegates a specific parser for a UHT type. Each of which of the Marco types (UClass, UStruct, UProperty, UInterface, UFunction etc..).
5. The tokens are resolved into the symbols table which contains two sub tables (`Engine`, `Source`).
6. The symbols table is "Populated" with types then "Resolved". Resolved meaning each UHT type is processed uniquely. Each UHT type that is resolved has a function delegate that is called to perform unique processing. Once the headers are resolved the Exporter is ready to run.
7. The exporter creates an Export Factory that calls the `Run` delegate for a specific Exporter. Interestingly Epic provided a hook point where you run your own exporter as a plugin. This allows for some flexibility with the Engine to implement your own code generation system along side UHT. 
8. `CodeGen` is the specific exporter that generates the type system code into the generated.h, gen.cpp files which are stored in the `Intermediate` folder.

UHT.cs has the process well coded and it's process clearly visible.

```c#
StepReadManifestFile(manifestFilePath);  
StepPrepareModules();  
StepPrepareHeaders();  
StepParseHeaders();  
StepPopulateTypeTable();  
StepResolveInvalidCheck();  
StepBindSuperAndBases();  
RecursiveStructCheck();  
StepResolveBases();  
StepResolveProperties();  
StepResolveFinal();  
StepResolveValidate();  
StepCollectReferences();  
TopologicalSortHeaderFiles();
// If we are deleting the reference directory, then wait for that task to complete  
if (_referenceDeleteTask != null)  
{  
    Logger.LogTrace("Step - Waiting for reference output to be cleared.");  
    _referenceDeleteTask.Wait();  
}  
  
Logger.LogTrace("Step - Starting exporters.");  
StepExport();


Furthermore, CURRENT_FILE_ID and EVENT_PARAMS macros are defined
FNativeClassHeaderGenerator::FNativeClassHeaderGenerator(const UPackage*, const TSet<FUnrealSourceFile*>&, FClasses&, bool)
```

Inside ObjectMacros.h, the `UC` namespace is defined to provide syntax highlighting and auto complete information for IDE's.

```cpp
namespace UC
{
// valid keywords for the UCLASS macro
enum
{
/// This keyword is used to set the actor group that the class is show in, in the editor.
classGroup,
/// Declares that instances of this class should always have an outer of the specified class. This is inherited by
subclasses unless overridden.
Within, /* =OuterClassName */
/// Exposes this class as a type that can be used for variables in blueprints
BlueprintType,
/// Prevents this class from being used for variables in blueprints
NotBlueprintType,
/// Exposes this class as an acceptable base class for creating blueprints. The default is NotBlueprintable, unless
inherited otherwise. This is inherited by subclasses.
Blueprintable,
/// Specifies that this class is *NOT* an acceptable base class for creating blueprints. The default is
NotBlueprintable, unless inherited otherwise. This is inherited by subclasses.
NotBlueprintable,
...
```

#### UObject Reflection Type System Structure

As previously mentioned, the design of the Object System in Unreal begins with the type system. Unreal utilizes its C# Unreal Header Tool to gather and generate reflection information. Understanding this reflection process provides insights into the construction and utilization of UObject, especially in conjunction with Blueprints, Garbage Collection, Serialization, and other aspects.

A critical reason for generating code instead of data files lies in the synchronization benefits. When reflection data is generated as C++ code, it can be compiled to ensure alignment with the binary. This synchronization becomes particularly significant when the static initialization and dynamic linking of a package are intricately connected to the generation of the reflection hierarchy of UClass with UObject.

Simple Hello Class Reflection

"#include "Hello.generated.h"

```cpp
UClass()  
class Hello  
{  
public:  
    UPROPERTY()  
    int Count;  
    UFUNCTION()  
    void Say();  
};
```

#### Structure Diagram

![[Unreal Engine Reflection System/Reflection Diagrams/UObjectDiagram.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/05c9c6840edab10bd8dafc7b778a5a8b62801b33/Unreal%20Engine%20Reflection%20System/Reflection%20Diagrams/UObjectDiagram.png)

In simple terms any program is composed of types and nested functions, Types: Enums, structs, classes; fields, functions and subtypes can all be defined in classes.

In simple terms, any program consists of types and nested functions. Types, including Enums, structs, and classes, can define fields, functions, and subtypes within classes.

**Color Scheme:**

- **Yellow:** Declarations, encompassing UStruct, UEnum, UClass, UScriptStruct, and UFunction.
    
- **Green:** UProperty, representing field definitions.
    
- **Orange:** UField, utilized for storing a pointer to the Next Field, functioning as a Linked List Traversal.
    
- **Purple:** UMeta, holding UPackage Information.
    
- **Red:** Top-level UObject Type.

Types can be divided conceptually into two types: Aggregate and Atomic Types

- **Aggregate Type:** This C++ type groups together multiple individual values of possibly different types under a single name. Objects created from aggregate types hold multiple sub-objects, such as structs and arrays. These sub-objects can, in turn, be of any type, including other aggregates. An essential feature of an aggregate type is that its members are accessed using dot notation (for structs) or index notation (for arrays).
    
- **Atomic Type:** In contrast, an atomic type in C++ represents a single, indivisible value. These types, such as int, float, char, etc., serve as the foundational building blocks for data manipulation. Atomic types cannot be further divided into smaller meaningful components and are directly operated upon using arithmetic and logical operations.

##### Aggregate Types

- **UFunction:** This type can only contain attributes as input and output parameters of the function.
    
- **UScriptStruct:** Similar to a POD struct in C++, it can only contain properties. In Unreal Engine, it serves as a "lightweight" UObject with reflection support, serialization, replication, etc. Unlike a typical UObject, it is not managed by the Garbage Collector, requiring manual control over memory allocation and release. UScriptStruct inherits from UStruct.
    
- **UClass:** This type can contain both properties and functions, and it is the type encountered most frequently.
    

##### Atomic Types

- **UEnum:** Supports common enumerations and enum classes.
    
- **Basic Types:** Types like int and FString do not require explicit declaration; they are simply enumerated and supported by different UProperty subclasses.
    
- **Super Struct:** At the root of the UStruct inheritance hierarchy, the Super Struct serves as the parent/root. The type data generated by the UStruct Macro in C++ is represented by UScriptStruct, forming the unification of aggregate types under the UStruct base class.
##### UInterface

- Similar to a virtual class in C++, UInterface can inherit multiple interfaces. However, it should only contain functions.
    
- Ordinary classes should inherit from UObject. UInterface is a special class that is still stored in UClass.
##### FProperty

A field "type instance"  representing a field's type in Unreal Engine. Properties in Unreal have various sub-categories, and FProperty instantiates numerous subclasses through templates. This type inherits from the FField class.

When the Link() is called, FField receives an Int32 Offset. 

Further details on this process will be covered later in the document or can be delved into deeper in the `UObject Construction & Post Initalization Document`

This design facilitates copy/allocation in the Blueprint Virtual Machine without requiring knowledge of the specific type of UMyObject.

Structs, having their custom constructors, and TArrays, which cannot be copied with Memcopy. To address this, FProperty incorporates Virtual Functions like InitializeValue_Container and CopyCompleteValue.
##### UMetaData

UMedaData is a key-value pair within a TMap<FName, FString>, utilized exclusively for the editor. This pair provides the editor with classification, friendly name, and prompt information.
##### FField

FField describes the base class of reflected data objects.

The name "FField" implies that, whether a declaration or a definition, it can be considered a field within the type system. 

Note: It's important to note that FField is a newly introduced type. Upon examining the engine source, you'll observe a bridge being constructed to maintain compatibility with UField, ensuring data objects function seamlessly with blueprints. Looking ahead, there is an intention by Epic to transition from UProperty to FProperty to lower usage costs.

What is the Intention of the base class FField?

Dazhao proposes that directly inheriting UProperty, UStruct, and UEnum from UObject might seem intuitive, but there are specific reasons for not doing so:

1. **Unification of Data Types:** By having a common base class for all data types, such as UField, it becomes easier to reference and traverse all types of data using an array. This unification also helps maintain the original sequence of definitions, allowing for straightforward backtracking and coherence between the UClass-generated type data and the original code.
    
2. **Metadata Attachment:** All declarations or definitions (UProperty, UStruct, UEnum) can be augmented with additional metadata (UMetaData). Placing this in their common base class simplifies management.
    
3. **Convenient Extension:** Having a shared base class, UField, makes it convenient to add extra methods. For instance, adding a Print method to display the declaration of each field can be achieved by introducing a virtual method to UField, which can then be overloaded in the subclasses as needed. This approach facilitates the extension of functionality.

**Why should FField inherit from UObject?**

1. **Garbage Collection (GC):** Dispensable; the type data, once allocated, should not be released. The current GC utilizes the type system to support object reference traversal.
    
2. **Reflection:** Slightly applicable.
    
3. **Editor Integration:** Utilized for integrated editing, particularly when creating function variables and other operations in the blueprint.
    
4. **Class Default Object (CDO):** Not needed; typically, there's only one copy of each type of type data, and CDO is used on objects.
    
5. **Serialization:** Essential for preserving type data, such as the type created by blueprints.
    
6. **Replication:** Not highly useful, as network replication currently utilizes type data.
    
7. **Remote Procedure Call (RPC):** Inconsequential.
    
8. **Automatic Attribute Update:** Not required, as type data generally does not change frequently.
    
9. **Statistics:** Optional.

Furthermore Serialization is found to be the most important functionality reflection provides.

The underlying principle is to combine dispensable elements with necessary functions, all based on a unified idea: having all types of data inherit from UObject. This design ensures that serialization and other operations do not require maintaining two separate sets of functionalities. While this approach may not appear entirely pure, its advantages generally outweigh the disadvantages. On an object, you can utilize `Instance->GetClass()` to obtain the UClass object. Interestingly, calling `GetClass()` on the UClass itself returns the UClass, facilitating the distinction between objects and type data.

##### In Summary:

- **UStruct:** Contains a linked list of FField, indicating flexibility with fewer restrictions. Although there are currently no nested types, UStruct has the capability to include both nested types and attributes. UFunction is limited to attributes only, while UScriptStruct exclusively has attributes and cannot include functions. The pointer `UStruct* SuperStruct` is used in UStruct to reference the inherited base class.
    
- **MetaData:** Compiled in classes, unpacked, and set up at boot time. Generally, metadata is not commonly used, except by the editor.

### Type System Code Generation

Assuming that UHT has sufficiently analyzed and acquired type metadata information, the subsequent step involves utilizing this information to construct the preceding type system structure in the program's memory. This procedural step is termed "registration" and will be expounded upon in greater detail in a dedicated section later on.

**Important Concept:**

Similar to the construction process in general programs, which involves preprocessing, compilation, assembly, and linking, Unreal Engine conceptualizes the following stages to simulate the construction process in memory: generation, collection, registration, and linking.

#### Anatomy of generated Macros: UClass, UFunction, UProperty, UEnum & UInterface

```cpp
#pragma once
#include "CoreMinimal.h"
#include "UObject/NoExportTypes.h"
#include "MyClass.generated.h"
/**
*
*/
UCLASS()
class REFLECTIONTEST_API UMyClass : public UObject
{
    GENERATED_BODY()
public:
    UPROPERTY()
    int Health;
    UFUNCTION(BlueprintCallable, Category = "Hello")
    void CallableFunc();
    UFUNCTION(BlueprintNativeEvent, Category = "Hello")
    void NativeFunc();
    UFUNCTION(BlueprintImplementableEvent, Category = "Hello")
    void ImplementableFunc();
};

UENUM()
enum EMyEnum : int
{
    E_Num1,
    E_Num2,
    E_Num3,
};

USTRUCT()
struct REFLECTIONTEST_API FMyStruct
{
    GENERATED_USTRUCT_BODY();
    UPROPERTY()
    float Score;
};

UINTERFACE(BlueprintType)
class REFLECTIONTEST_API UMyInterface : public UInterface
{
    GENERATED_UINTERFACE_BODY()
};
class IMyInterface
{
    GENERATED_IINTERFACE_BODY()
public:
    UFUNCTION(BlueprintImplementableEvent)
    void BPFunc() const;
};
```


#### UClass Anatomy

GENERATED_BODY() - This macro is the main declaration with metadata definitions and can be see in the file ObjectMacros.h Line 689 (5.2)

```cpp
// This pair of macros is used to help implement GENERATED_BODY() and GENERATED_USTRUCT_BODY()
#define BODY_MACRO_COMBINE_INNER(A,B,C,D) A##B##C##D
#define BODY_MACRO_COMBINE(A,B,C,D) BODY_MACRO_COMBINE_INNER(A,B,C,D)
// Include a redundant semicolon at the end of the generated code block, so that intellisense parsers can start parsing
// a new declaration if the line number/generated code is out of date.
#define GENERATED_BODY_LEGACY(...) BODY_MACRO_COMBINE(CURRENT_FILE_ID,_,__LINE__,_GENERATED_BODY_LEGACY);
#define GENERATED_BODY(...) BODY_MACRO_COMBINE(CURRENT_FILE_ID,_,__LINE__,_GENERATED_BODY)
```

What is the macro BODY_MACRO_COMBINE? The macros are defined as BODY_MACRO_COMBINE_INNER, which concatenates parameters as one string. So an example here would be  
  
FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_13_PROLOG  
The format is CURRENT_FILE_ID_12_PROLOG. You see 12 is the actual line number in the file where the UCLASS is written!  
Therefore CURRENT_FILE_ID would be replaced before BODY_MACRO_COMBINE I won’t go into detail about keywords and specifiers from the macro metadata parameters.

##### MyClass.generated.h

At the beginning, there are `#ifdef` directives to guard against redundant inclusion of `MyClass.generated.h` in case `#pragma once` is not defined.

```cpp
PRAGMA_DISABLE_DEPRECATION_WARNINGS
#ifdef REFLECTIONTEST_MyClass_generated_h
#error "MyClass.generated.h already included, missing '#pragma once' in MyClass.h"
#endif
#define REFLECTIONTEST_MyClass_generated_h


#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_ACCESSORS
#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_CALLBACK_WRAPPERS
#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_INCLASS_NO_PURE_DECLS
private:
    static void StaticRegisterNativesUMyClass();
    friend struct Z_Construct_UClass_UMyClass_Statics;
public:
    DECLARE_CLASS(UMyClass, UObject, COMPILED_IN_FLAGS(0), CASTCLASS_None, TEXT("/Script/ReflectionTest"), NO_API)
    DECLARE_SERIALIZER(UMyClass)
#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_INCLASS
private:
    static void StaticRegisterNativesUMyClass();
    friend struct Z_Construct_UClass_UMyClass_Statics;
public:
    DECLARE_CLASS(UMyClass, UObject, COMPILED_IN_FLAGS(0), CASTCLASS_None, TEXT("/Script/ReflectionTest"), NO_API)
    DECLARE_SERIALIZER(UMyClass)
#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_STANDARD_CONSTRUCTORS
    /** Standard constructor, called after all reflected properties have been initialised */
    NO_API UMyClass(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
    DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(UMyClass)
    DECLARE_VTABLE_PTR_HELPER_CTOR(NO_API, UMyClass);
    DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER(UMyClass);
private:
    /** Private move- and copy-constructors, should never be used */
    NO_API UMyClass(UMyClass&&);
    NO_API UMyClass(const UMyClass&);
public:
    NO_API virtual ~UMyClass();
#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_ENHANCED_CONSTRUCTORS
    /** Standard constructor, called after all reflected properties have been initialised */
    NO_API UMyClass(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
private:
    /** Private move- and copy-constructors, should never be used */
    NO_API UMyClass(UMyClass&&);
    NO_API UMyClass(const UMyClass&);
public:
    DECLARE_VTABLE_PTR_HELPER_CTOR(NO_API, UMyClass);
    DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER(UMyClass);
    DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(UMyClass)
    NO_API virtual ~UMyClass();
#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_28_PROLOG
#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_GENERATED_BODY_LEGACY
PRAGMA_DISABLE_DEPRECATION_WARNINGS
public:
    FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_SPARSE_DATA
    FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_RPC_WRAPPERS
    FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_ACCESSORS
    FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_CALLBACK_WRAPPERS
    FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_INCLASS
    FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_STANDARD_CONSTRUCTORS
public:
PRAGMA_ENABLE_DEPRECATION_WARNINGS
#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_GENERATED_BODY
PRAGMA_DISABLE_DEPRECATION_WARNINGS
public:
    FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_SPARSE_DATA
    FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_RPC_WRAPPERS_NO_PURE_DECLS
    FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_ACCESSORS
  FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_CALLBACK_WRAPPERS
    FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_INCLASS_NO_PURE_DECLS
    FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_31_ENHANCED_CONSTRUCTORS
private:
PRAGMA_ENABLE_DEPRECATION_WARNINGS
```

CTRL+F Definitions:
```cpp
INCLASS_DECLS
INCLASS_NO_PURE_DECLS

#define DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(TClass)
static void __DefaultConstructor(const FObjectInitializer& X) { new((EInternal*)X.GetObj())TClass(X);


DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(TClass)
```

This macro is crucial because, in C++, having a function pointer directly to a native C++ constructor is not possible. Unreal Engine addresses this limitation by introducing a constructor wrapper with an ordinary function pointer. All constructors share the same signature and are stored in a function pointer within UClass.

**DECLARE_CLASS**

```cpp
DECLARE_CLASS(UMyClass, UObject, COMPILED_IN_FLAGS(0), CASTCLASS_None, TEXT("/Script/ReflectionTest"), NO_API)  
DECLARE_SERIALIZER(UMyClass)
```

The Declare Class macro holds significant importance as it defines the UClass skeleton and specifies where the `StaticClass()` function is defined. Key parameters within this macro include:

- **TClass:** Class name
- **TSuperClass:** Base class name
- **TStaticFlags:** Attribute flags of the class, with 0 indicating the most default state. Referencing the EClassFlags enumeration reveals other possible definitions.
- **TStaticCastFlags:** Specifies which classes the class can be converted to. A value of 0 means it cannot be converted to any default classes. Explore the EClassCastFlags statement for available default class conversions.
- **TPackage:** The package name of the class. All objects must belong to a package, and each package has a unique name. In this instance, "/Script/ReflectionTest" denotes the ReflectionTest package under Script, with Script representing the user's own implementation—whether in C++ or Blueprint. ReflectionTest is the project name, and all objects defined within this project reside in this package. The concept of a Package involves the organization of subsequent Objects.
- **TRequiredAPI:** This mark is used for DLL import and export. In this case, it is NO_API, indicating that no export is required for the final executable.

```cpp
/*-----------------------------------------------------------------------------
Class declaration macros.
-----------------------------------------------------------------------------*/
#define DECLARE_CLASS( TClass, TSuperClass, TStaticFlags, TStaticCastFlags, TPackage, TRequiredAPI )
private:
	TClass& operator=(TClass&&);
	TClass& operator=(const TClass&);
	TRequiredAPI static UClass* GetPrivateStaticClass();
public:
	/** Bitwise union of #EClassFlags pertaining to this class.*/
	static constexpr EClassFlags StaticClassFlags=EClassFlags(TStaticFlags);
	/** Typedef for the base class ({{ typedef-type }}) */
	typedef TSuperClass Super;
	/** Typedef for {{ typedef-type }}. */
	typedef TClass ThisClass;
/** Returns a UClass object representing this class at runtime */
inline static UClass* StaticClass()
{
	return GetPrivateStaticClass();
}
/** Returns the package this class belongs in */
inline static const TCHAR* StaticPackage()
{
	return TPackage;
}
/** Returns the static cast flags for this class */
inline static EClassCastFlags StaticClassCastFlags()
{
	return TStaticCastFlags;
}
/** For internal use only; use StaticConstructObject() to create new objects. */
inline void* operator new(const size_t InSize, EInternal InInternalOnly, UObject* InOuter =
(UObject*)GetTransientPackage(), FName InName = NAME_None, EObjectFlags InSetFlags = RF_NoFlags)
{
	return StaticAllocateObject(StaticClass(), InOuter, InName, InSetFlags);
	}
	/** For internal use only; use StaticConstructObject() to create new objects. */
	inline void* operator new( const size_t InSize, EInternal* InMem )
	{
	return (void*)InMem;
	}
	/* Eliminate V1062 warning from PVS-Studio while keeping MSVC and Clang happy. */
	inline void operator delete(void* InMem)
	{
	::operator delete(InMem);
    }
```

##### MyClass.generated.cpp

```cpp
#include "UObject/GeneratedCppIncludes.h"
#include "ReflectionTest/MyClass.h"
PRAGMA_DISABLE_DEPRECATION_WARNINGS
void EmptyLinkFunctionForGeneratedCodeMyClass() {}
// Cross Module References
	COREUOBJECT_API UClass* Z_Construct_UClass_UInterface();
	COREUOBJECT_API UClass* Z_Construct_UClass_UObject(); // Makes Sure UObject's UClass is registered
	REFLECTIONTEST_API UClass* Z_Construct_UClass_UMyClass(); // Reference to UObject's UClass.
	REFLECTIONTEST_API UClass* Z_Construct_UClass_UMyClass_NoRegister(); // Construct UClass object for UMyClass (without
	registering).
	REFLECTIONTEST_API UClass* Z_Construct_UClass_UMyInterface();
	REFLECTIONTEST_API UClass* Z_Construct_UClass_UMyInterface_NoRegister();
	REFLECTIONTEST_API UEnum* Z_Construct_UEnum_ReflectionTest_EMyEnum();
	REFLECTIONTEST_API UScriptStruct* Z_Construct_UScriptStruct_FMyStruct();
	UPackage* Z_Construct_UPackage__Script_ReflectionTest(); // Construct UPackage for MyClass, I think this is Unused. There
	is no implementation.U
// End Cross Module References
void UMyClass::StaticRegisterNativesUMyClass()
{
	UClass* Class = UMyClass::StaticClass();
	static const FNameNativePtrPair Funcs[] = {
		{ "CallableFunc", &UMyClass::execCallableFunc },
		{ "NativeFunc", &UMyClass::execNativeFunc },
};
FNativeFunctionRegistrar::RegisterFunctions(Class, Funcs, UE_ARRAY_COUNT(Funcs));
}

IMPLEMENT_CLASS_NO_AUTO_REGISTRATION(UMyClass); // Important Used to Implement UCLASS

UClass* Z_Construct_UClass_UMyClass_NoRegister()
{
return UMyClass::StaticClass(); // Here gives us Direct Access to UClass Object
}

UClass* Z_Construct_UClass_UMyClass()
{
	if (!Z_Registration_Info_UClass_UMyClass.OuterSingleton)
	{
	UECodeGen_Private::ConstructUClass(Z_Registration_Info_UClass_UMyClass.OuterSingleton,
	Z_Construct_UClass_UMyClass_Statics::ClassParams);
	}
	return Z_Registration_Info_UClass_UMyClass.OuterSingleton;
}


void ConstructUClass(UClass*& OutClass, const FClassParams& Params)
{
	if (OutClass && (OutClass->ClassFlags & CLASS_Constructed)) // Prevents Duplicate Registration
	{
	    return;
    }
	/**
	The For Loop below,
	Can see the Array of Function pointers are called for Macros Z_Construct_UClass_UObject
	Z_Construct_UPackage__Script Macro is also called to created UPackage
	UObject* (*const Z_Construct_UClass_UMyClass_Statics::DependentSingletons[])() = {
	(UObject* (*)())Z_Construct_UClass_UObject,
	(UObject* (*)())Z_Construct_UPackage__Script_ReflectionTest,
	};
	*/
	for (UObject* (*const *SingletonFunc)() = Params.DependencySingletonFuncArray, *(*const *SingletonFuncEnd)() =
	SingletonFunc + Params.NumDependencySingletons; SingletonFunc != SingletonFuncEnd; ++SingletonFunc)
	{
		(*SingletonFunc)();
	}
	// Params.ClassNoRegister comes from UECodeGen_Private::FClassParams
	// &UMyClass::StaticClass
	// This is the Function Pointer that gets us UMyClass Object which is called
	// by &UMyClass::StaticClass() at this point ::StaticClass() is already defined
	UClass* NewClass = Params.ClassNoRegisterFunc();
	OutClass = NewClass;
	if (NewClass->ClassFlags & CLASS_Constructed)
	{
		return;
	}
	UObjectForceRegistration(NewClass); // Register itself to extract information.
	UClass* SuperClass = NewClass->GetSuperClass();
	if (SuperClass)
	{
		NewClass->ClassFlags |= (SuperClass->ClassFlags & CLASS_Inherit);
	}
	// This Adds Class_Constructed Flags and Required API Flags
	NewClass->ClassFlags |= (EClassFlags)(Params.ClassFlags | CLASS_Constructed);
	// Make sure the reference token stream is empty since it will be reconstructed later on
	// This should not apply to intrinsic classes since they emit native references before AssembleReferenceTokenStream is called.
	
	if ((NewClass->ClassFlags & CLASS_Intrinsic) != CLASS_Intrinsic)
	{
		check((NewClass->ClassFlags & CLASS_TokenStreamAssembled) != CLASS_TokenStreamAssembled);
		NewClass->ReferenceTokens.Reset();
	}
	/**
	CreateLinkAndAddChildFunctionsToMap Creates the Function Pointer for Native Functions
	of this class. This then adds native functions to an internal native Function Table
	with a unicode name. This is used when generating code from blueprints.
	This struct maps a string anme to a native function.
	FPropertyParamsBase is the struct, we basically create Property pointers, See Collection Phase
	*/
	NewClass->CreateLinkAndAddChildFunctionsToMap(Params.FunctionLinkArray, Params.NumFunctions);
	/**
	Inside UObjectGlobals.cpp Line 5552 we see this function inside namespace UECodeGen_Private
	Creates the Prop objects. FProperties are Inherited from FField.
	*/
	ConstructFProperties(NewClass, Params.PropertyArray, Params.NumProperties);
	if (Params.ClassConfigNameUTF8)
	{
		NewClass->ClassConfigName = FName(UTF8_TO_TCHAR(Params.ClassConfigNameUTF8));
	}
	// Checks if it's abstract class
	NewClass->SetCppTypeInfoStatic(Params.CppClassInfo);
	if (int32 NumImplementedInterfaces = Params.NumImplementedInterfaces)
	{
		NewClass->Interfaces.Reserve(NumImplementedInterfaces);
		for (const FImplementedInterfaceParams* ImplementedInterface = Params.ImplementedInterfaceArray,
		*ImplementedInterfaceEnd = ImplementedInterface + NumImplementedInterfaces; ImplementedInterface != ImplementedInterfaceEnd;
		++ImplementedInterface)
		{
			UClass* (*ClassFunc)() = ImplementedInterface->ClassFunc;
			UClass* InterfaceClass = ClassFunc ? ClassFunc() : nullptr;
			NewClass->Interfaces.Emplace(InterfaceClass, ImplementedInterface->Offset, ImplementedInterface-
			>bImplementedByK2);
		}
	}
	#if WITH_METADATA
	AddMetaData(NewClass, Params.MetaDataArray, Params.NumMetaData);
	#endif
	// Perform "static" linking, Wrapper Function UStruct::Link(..)
	NewClass->StaticLink();
	// Uses some SideCar Design pattern to extend functionality of the USuperStruct?
	NewClass->SetSparseClassDataStruct(NewClass->GetSparseClassDataArchetypeStruct());
}
```

The macro `IMPLEMENT_CLASS_NO_AUTO_REGISTRATION(UMyClass)` serves the purpose of transmitting information about the class to the `GetPrivateStaticClassBody` function. This function is responsible for generating the skeleton object of type `UMyClass`. The provided explanation relates specifically to the CPP implementation of the `DECLARE_CLASS()` macro.

```cpp
// Implement the GetPrivateStaticClass and the registration info but do not auto register the class.
// This is primarily used by UnrealHeaderTool
#define IMPLEMENT_CLASS_NO_AUTO_REGISTRATION(TClass)
FClassRegistrationInfo Z_Registration_Info_UClass_##TClass;
UClass* TClass::GetPrivateStaticClass()
{
if (!Z_Registration_Info_UClass_##TClass.InnerSingleton)
{
	/* this could be handled with templates, but we want it external to avoid code bloat */
	GetPrivateStaticClassBody(
		StaticPackage(),
		(TCHAR*)TEXT(#TClass) + 1 + ((StaticClassFlags & CLASS_Deprecated) ? 11 : 0),
		Z_Registration_Info_UClass_##TClass.InnerSingleton,
		StaticRegisterNatives##TClass,
		sizeof(TClass),
		alignof(TClass),
		TClass::StaticClassFlags,
		TClass::StaticClassCastFlags(),
		TClass::StaticConfigName(),
		(UClass::ClassConstructorType)InternalConstructor<TClass>,
		(UClass::ClassVTableHelperCtorCallerType)InternalVTableHelperCtorCaller<TClass>,
		UOBJECT_CPPCLASS_STATICFUNCTIONS_FORCLASS(TClass),
		&TClass::Super::StaticClass,
		&TClass::WithinClass::StaticClass
		);
	}
	return Z_Registration_Info_UClass_##TClass.InnerSingleton;
}
```

##### Expanded MyClass.h

```cpp
class REFLECTIONTEST_API UMyClass : public UObject
{
private:
	static void StaticRegisterNativesUMyClass();
	friend struct Z_Construct_UClass_UMyClass_Statics;
private:
	TClass& operator=(TClass&&);
	TClass& operator=(const TClass&);
	TRequiredAPI static UClass* GetPrivateStaticClass();
public:
	/** Bitwise union of #EClassFlags pertaining to this class.*/
	static constexpr EClassFlags StaticClassFlags=EClassFlags(TStaticFlags);
	/** Typedef for the base class ({{ typedef-type }}) */
	typedef TSuperClass Super;
	/** Typedef for {{ typedef-type }}. */
	typedef TClass ThisClass;
	/** Returns a UClass object representing this class at runtime */
inline static UClass* StaticClass()
{
	return GetPrivateStaticClass();
}
/** Returns the package this class belongs in */
inline static const TCHAR* StaticPackage()
{
	return TPackage;
}
/** Returns the static cast flags for this class */
inline static EClassCastFlags StaticClassCastFlags()
{
	return TStaticCastFlags;
}
/** For internal use only; use StaticConstructObject() to create new objects. */
inline void* operator new(const size_t InSize, EInternal InInternalOnly, UObject* InOuter =
(UObject*)GetTransientPackage(), FName InName = NAME_None, EObjectFlags InSetFlags = RF_NoFlags)
{
	return StaticAllocateObject(StaticClass(), InOuter, InName, InSetFlags);
}
/** For internal use only; use StaticConstructObject() to create new objects. */
inline void* operator new( const size_t InSize, EInternal* InMem )
{
	return (void*)InMem;
}
/* Eliminate V1062 warning from PVS-Studio while keeping MSVC and Clang happy. */
inline void operator delete(void* InMem)
{
	::operator delete(InMem);
}
// Serializer Macro
friend FArchive &operator<<( FArchive& Ar, TClass*& Res )
{
	return Ar << (UObject*&)Res;
}
friend void operator<<(FStructuredArchive::FSlot InSlot, TClass*& Res)
{
	InSlot << (UObject*&)Res;
}
7/** Standard constructor, called after all reflected properties have been initialized */
NO_API UMyClass(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
static void __DefaultConstructor(const FObjectInitializer& X) { new((EInternal*)X.GetObj())TClass(X); }

private: /** Private move- and copy-constructors, should never be used */ NO_API UMyClass(UMyClass&&); NO_API
    UMyClass(const UMyClass&);
```

##### Expanded MyClass.Generated.cpp

```cpp
// Cross Module References CoreUObject
COREUOBJECT_API UClass* Z_Construct_UClass_UObject();
REFLECTIONTEST_API UClass* Z_Construct_UClass_UMyClass();
REFLECTIONTEST_API UClass* Z_Construct_UClass_UMyClass_NoRegister();
UPackage* Z_Construct_UPackage__Script_ReflectionTest();
void EmptyLinkFunctionForGeneratedCodeMyClass() {}
DEFINE_FUNCTION(UMyClass::execNativeFunc)
{
	P_FINISH;
	P_NATIVE_BEGIN;
	P_THIS->NativeFunc_Implementation();
	P_NATIVE_END;
}
DEFINE_FUNCTION(UMyClass::execCallableFunc)
{
	P_FINISH;
	P_NATIVE_BEGIN;
	P_THIS->CallableFunc();
	P_NATIVE_END;
}
static FName NAME_UMyClass_ImplementableFunc = FName(TEXT("ImplementableFunc"));
void UMyClass::ImplementableFunc()
{
	ProcessEvent(FindFunctionChecked(NAME_UMyClass_ImplementableFunc),NULL);
}
static FName NAME_UMyClass_NativeFunc = FName(TEXT("NativeFunc"));
void UMyClass::NativeFunc()
{
	ProcessEvent(FindFunctionChecked(NAME_UMyClass_NativeFunc),NULL);
}
void UMyClass::StaticRegisterNativesUMyClass()
{
	UClass* Class = UMyClass::StaticClass();
	static const FNameNativePtrPair Funcs[] = {
	{ "CallableFunc", &UMyClass::execCallableFunc },
	{ "NativeFunc", &UMyClass::execNativeFunc },
	};
	FNativeFunctionRegistrar::RegisterFunctions(Class, Funcs, UE_ARRAY_COUNT(Funcs));
}
// Define FuncParameters and Metadata Params
struct Z_Construct_UFunction_UMyClass_CallableFunc_Statics
{
	#if WITH_METADATA
	static const UECodeGen_Private::FMetaDataPairParam Function_MetaDataParams[];
	#endif
	static const UECodeGen_Private::FFunctionParams FuncParams;
};
// Static Defines for the Collection Phase
struct Z_Construct_UClass_UMyClass_Statics
{
	static UObject* (*const DependentSingletons[])();
	static const FClassFunctionLinkInfo FuncInfo[];
	#if WITH_METADATA
	static const UECodeGen_Private::FMetaDataPairParam Class_MetaDataParams[];
	#endif
	#if WITH_METADATA
	static const UECodeGen_Private::FMetaDataPairParam NewProp_Health_MetaData[];
	#endif
	static const UECodeGen_Private::FUnsizedIntPropertyParams NewProp_Health;
	static const UECodeGen_Private::FPropertyParamsBase* const PropPointers[];
	static const FCppClassTypeInfoStatic StaticCppClassTypeInfo;
	static const UECodeGen_Private::FClassParams ClassParams;
};

UClass* TClass::GetPrivateStaticClass(){
if (!Z_Registration_Info_UClass_##TClass.InnerSingleton)
{
	/* this could be handled with templates, but we want it external to avoid code bloat */
	GetPrivateStaticClassBody(
	StaticPackage(),
	(TCHAR*)TEXT(#TClass) + 1 + ((StaticClassFlags & CLASS_Deprecated) ? 11 : 0),
	Z_Registration_Info_UClass_##TClass.InnerSingleton,
	StaticRegisterNatives##TClass,
	sizeof(TClass),
	alignof(TClass),
	TClass::StaticClassFlags,
	TClass::StaticClassCastFlags(),
	TClass::StaticConfigName(),
	(UClass::ClassConstructorType)InternalConstructor<TClass>,
	(UClass::ClassVTableHelperCtorCallerType)InternalVTableHelperCtorCaller<TClass>,
	UOBJECT_CPPCLASS_STATICFUNCTIONS_FORCLASS(TClass),
	&TClass::Super::StaticClass,
	&TClass::WithinClass::StaticClass
	);
}
	return Z_Registration_Info_UClass_##TClass.InnerSingleton;
}

UClass* Z_Construct_UClass_UMyClass_NoRegister()
{
	return UMyClass::StaticClass();
}
UObject* (*const Z_Construct_UClass_UMyClass_Statics::DependentSingletons[])() = {
	(UObject* (*)())Z_Construct_UClass_UObject,
	(UObject* (*)())Z_Construct_UPackage__Script_ReflectionTest,
};
const FClassFunctionLinkInfo Z_Construct_UClass_UMyClass_Statics::FuncInfo[] = {
	{ &Z_Construct_UFunction_UMyClass_CallableFunc, "CallableFunc" }, // 2296517141
	{ &Z_Construct_UFunction_UMyClass_ImplementableFunc, "ImplementableFunc" }, // 1412324750
	{ &Z_Construct_UFunction_UMyClass_NativeFunc, "NativeFunc" }, // 1217410402
};

#if WITH_METADATA
const UECodeGen_Private::FMetaDataPairParam Z_Construct_UClass_UMyClass_Statics::Class_MetaDataParams[] = {
	{ "IncludePath", "MyClass.h" },
	{ "ModuleRelativePath", "MyClass.h" },
	{ "ObjectInitializerConstructorDeclared", "" },
};
#endif
#if WITH_METADATA
const UECodeGen_Private::FMetaDataPairParam Z_Construct_UClass_UMyClass_Statics::NewProp_Health_MetaData[] = {
	{ "ModuleRelativePath", "MyClass.h" },
};
#endif
const FClassRegisterCompiledInInfo
Z_CompiledInDeferFile_FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_Statics::ClassInfo[]
= {
	{ Z_Construct_UClass_UMyInterface, UMyInterface::StaticClass, TEXT("UMyInterface"),
	&Z_Registration_Info_UClass_UMyInterface, CONSTRUCT_RELOAD_VERSION_INFO(FClassReloadVersionInfo, sizeof(UMyInterface),
	4078863325U) },
	{ Z_Construct_UClass_UMyClass, UMyClass::StaticClass, TEXT("UMyClass"), &Z_Registration_Info_UClass_UMyClass,
	CONSTRUCT_RELOAD_VERSION_INFO(FClassReloadVersionInfo, sizeof(UMyClass), 138299640U) },
};
```

##### Reflection.Test.init.gen.cpp

During the `ConstructUClass` process, the call to `(*SingletonFunc)->Z_Registration_Info_UPackage__Script_ReflectionTest()` in the reflection code initiates the construction of the UPackage for the Reflection Test project.

```cpp
#include "UObject/GeneratedCppIncludes.h"
PRAGMA_DISABLE_DEPRECATION_WARNINGS
void EmptyLinkFunctionForGeneratedCodeReflectionTest_init() {}
static FPackageRegistrationInfo Z_Registration_Info_UPackage__Script_ReflectionTest;
FORCENOINLINE UPackage* Z_Construct_UPackage__Script_ReflectionTest()
{
	if (!Z_Registration_Info_UPackage__Script_ReflectionTest.OuterSingleton)
	{
		static const UECodeGen_Private::FPackageParams PackageParams = {
		"/Script/ReflectionTest",
		nullptr,
		0,
		PKG_CompiledIn | 0x00000000,
		0x68D52929,
		0x02EA1D27,
		METADATA_PARAMS(nullptr, 0)
		};
		UECodeGen_Private::ConstructUPackage(Z_Registration_Info_UPackage__Script_ReflectionTest.OuterSingleton,
		PackageParams);
	}
	return Z_Registration_Info_UPackage__Script_ReflectionTest.OuterSingleton;
}
static FRegisterCompiledInInfo
Z_CompiledInDeferPackage_UPackage__Script_ReflectionTest(Z_Construct_UPackage__Script_ReflectionTest,
TEXT("/Script/ReflectionTest"), Z_Registration_Info_UPackage__Script_ReflectionTest,
CONSTRUCT_RELOAD_VERSION_INFO(FPackageReloadVersionInfo, 0x68D52929, 0x02EA1D27));
PRAGMA_ENABLE_DEPRECATION_WARNINGS
```

`GetPrivateStaticClassBody` serves as the scaffolding for UClass. It retrieves the package for hot reload purposes, and `InitializePrivateStaticClass` is subsequently invoked. However, it's essential to note that the `OuterPrivate` member variable of UClass is set by the package from the game module. Importantly, it doesn't actually get created and set until `Object->Deferred` is called. `GetPrivateStaticClass` receives a reference to a function during this process.

#### UEnum Anatomy

```cpp
UENUM()  
enum EMyEnum : int  
{  
    E_Num1,  
    E_Num2,  
    E_Num3,  
};
```

##### EMyEnum.generated.h

```cpp
#define FOREACH_ENUM_EMYENUM(op) op(E_Num1) op(E_Num2) op(E_Num3)  
enum EMyEnum : int;  
template<> REFLECTIONTEST_API UEnum* StaticEnum<EMyEnum>();
```

##### EMyEnum.generated.cpp

```cpp
static FEnumRegistrationInfo Z_Registration_Info_UEnum_EMyEnum;
	static UEnum* EMyEnum_StaticEnum()
	{
		if (!Z_Registration_Info_UEnum_EMyEnum.OuterSingleton)
		{
			Z_Registration_Info_UEnum_EMyEnum.OuterSingleton = GetStaticEnum(Z_Construct_UEnum_ReflectionTest_EMyEnum,
			(UObject*)Z_Construct_UPackage__Script_ReflectionTest(), TEXT("EMyEnum"));
		}
		return Z_Registration_Info_UEnum_EMyEnum.OuterSingleton;
	}
	template<> REFLECTIONTEST_API UEnum* StaticEnum<EMyEnum>()
	{
		return EMyEnum_StaticEnum();
	}
	struct Z_Construct_UEnum_ReflectionTest_EMyEnum_Statics
	{
		static const UECodeGen_Private::FEnumeratorParam Enumerators[];
		#if WITH_METADATA
		static const UECodeGen_Private::FMetaDataPairParam Enum_MetaDataParams[];
		#endif
		static const UECodeGen_Private::FEnumParams EnumParams;
	};
	const UECodeGen_Private::FEnumeratorParam Z_Construct_UEnum_ReflectionTest_EMyEnum_Statics::Enumerators[] = {
		{ "E_Num1", (int64)E_Num1 },
		{ "E_Num2", (int64)E_Num2 },
		{ "E_Num3", (int64)E_Num3 },
	};
	#if WITH_METADATA
	const UECodeGen_Private::FMetaDataPairParam Z_Construct_UEnum_ReflectionTest_EMyEnum_Statics::Enum_MetaDataParams[] = {
		{ "E_Num1.Name", "E_Num1" },
		{ "E_Num2.Name", "E_Num2" },
		{ "E_Num3.Name", "E_Num3" },
		{ "ModuleRelativePath", "MyClass.h" },
	};
	#endif
	const UECodeGen_Private::FEnumParams Z_Construct_UEnum_ReflectionTest_EMyEnum_Statics::EnumParams = {
		(UObject*(*)())Z_Construct_UPackage__Script_ReflectionTest,
		nullptr,
		"EMyEnum",
		"EMyEnum",
		Z_Construct_UEnum_ReflectionTest_EMyEnum_Statics::Enumerators,
		UE_ARRAY_COUNT(Z_Construct_UEnum_ReflectionTest_EMyEnum_Statics::Enumerators),
		RF_Public|RF_Transient|RF_MarkAsNative,
		EEnumFlags::None,
		(uint8)UEnum::ECppForm::Regular,
		METADATA_PARAMS(Z_Construct_UEnum_ReflectionTest_EMyEnum_Statics::Enum_MetaDataParams,
		UE_ARRAY_COUNT(Z_Construct_UEnum_ReflectionTest_EMyEnum_Statics::Enum_MetaDataParams))
		
		};
UEnum* Z_Construct_UEnum_ReflectionTest_EMyEnum()
{
	if (!Z_Registration_Info_UEnum_EMyEnum.InnerSingleton)
	{
		UECodeGen_Private::ConstructUEnum(Z_Registration_Info_UEnum_EMyEnum.InnerSingleton,
		Z_Construct_UEnum_ReflectionTest_EMyEnum_Statics::EnumParams);
	}
	return Z_Registration_Info_UEnum_EMyEnum.InnerSingleton;
}
```

The structure for the MyEnum does not have more functions than Z_Construct_UEnum_ReflectionTest_EMyEnum(). Starting from the top we have

```cpp
static FEnumRegistrationInfo Z_Registration_Info_UEnum_EMyEnum;
```

During the collection process, the type information for MyEnum is populated within `FEnumRegistrationInfo`. This involves filling `Enumerators[]`, `FMetadatasPairParams[]`, and the primary data structure `FEnumParams`. `FEnumParams`, in particular, takes in the package function pointer, sets flags, and plays a central role in organizing the collected information.

Construction of the Enum Object will be
```cpp
REFLECTIONTEST_API UEnum* Z_Construct_UEnum_ReflectionTest_EMyEnum();
```

Which again calls UECodeGen_Private::ConstructUEnum

```cpp
void ConstructUEnum(UEnum*& OutEnum, const FEnumParams& Params)
{
	UObject* (*OuterFunc)() = Params.OuterFunc;
	UObject* Outer = OuterFunc ? OuterFunc() : nullptr;
	if (OutEnum)
	{
		return;
	}
	UEnum* NewEnum = new (EC_InternalUseOnlyConstructor, Outer, UTF8_TO_TCHAR(Params.NameUTF8), Params.ObjectFlags)
	UEnum(FObjectInitializer());
	OutEnum = NewEnum;
	
	TArray<TPair<FName, int64>> EnumNames;
	EnumNames.Reserve(Params.NumEnumerators);
	for (const FEnumeratorParam* Enumerator = Params.EnumeratorParams, *EnumeratorEnd = Enumerator +
	Params.NumEnumerators; Enumerator != EnumeratorEnd; ++Enumerator)
	{
		EnumNames.Emplace(UTF8_TO_TCHAR(Enumerator->NameUTF8), Enumerator->Value);
	}
	const bool bAddMaxKeyIfMissing = true;
	NewEnum->SetEnums(EnumNames, (UEnum::ECppForm)Params.CppForm, Params.EnumFlags, bAddMaxKeyIfMissing);
	NewEnum->CppType = UTF8_TO_TCHAR(Params.CppTypeUTF8);
	if (Params.DisplayNameFunc)
	{
		NewEnum->SetEnumDisplayNameFn(Params.DisplayNameFunc);
	}
	#if WITH_METADATA
		AddMetaData(NewEnum, Params.MetaDataArray, Params.NumMetaData);
	#endif
```

The address pointer of `UEnum` is passed, and function pointers are supplied to the static `EEnumParams` structure. The `TRegistrationInfo` is marked as using `FEnumRegistrationInfo`. This struct holds pointers to inner and outer singletons, where 'inner' refers to itself and 'outer' refers to the package. It's important to note that during the collection phase, the meanings of 'outer' and 'inner' differ.

The `RegisterCompiledInfo` is called from the constructor of `FRegisterCompiledInfo`, which passes parameters from static objects. C++ Perfect Forwarding is utilized here to navigate RValue and LValue issues when dealing with templates. This technique, available since C++11, is employed for very specific cases. In the engine source, various templated variations of the `RegisterCompiledInfo` function (for `UClass`, `UStruct`, `UEnum`, etc.) can be found, explaining the use of perfect forwarding for this purpose.

```cpp
struct FRegisterCompiledInInfo
{
    template <typename ... Args>
    FRegisterCompiledInInfo(Args&& ... args)
    {
    RegisterCompiledInInfo(std::forward<Args>(args)...);
    }
};


RegisterCompiledInInfo(class UEnum* (*InOuterRegister)(), const TCHAR* InPackageName, const TCHAR* InName,
FEnumRegistrationInfo& InInfo, const FEnumReloadVersionInfo& InVersionInfo)


class UEnum *GetStaticEnum(class UEnum *(*InRegister)(), UObject* EnumOuter, const TCHAR* EnumName)
{
    UEnum *Result = (*InRegister)();
    NotifyRegistrationEvent(*EnumOuter->GetOutermost()->GetName(), EnumName, ENotifyRegistrationType::NRT_Enum,
    ENotifyRegistrationPhase::NRP_Finished, nullptr, false, Result);
    return Result;
}
```

The `GetStaticEnum` function is responsible for creating the skeleton for MyEnum. This function reference is stored in `EnumInfo[]`, and this information is further passed to a static `FRegisterCompiledInInfo` structure type.

#### UStruct Anatomy

#####  FMyStruct.generated.h

```cpp
#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_59_GENERATED_BODY 
friend struct Z_Construct_UScriptStruct_FMyStruct_Statics; 
static class UScriptStruct* StaticStruct();
template<> REFLECTIONTEST_API UScriptStruct* StaticStruct<struct FMyStruct>();
```

Similarly, `GENERATED_USTRUCT_BODY()` will be replaced by our macro definition. This function is only defined for `StaticStruct` internally since `FMyStruct` does not inherit from UObject.

##### FMyStruct.generated.cpp

```cpp
static FStructRegistrationInfo Z_Registration_Info_UScriptStruct_MyStruct;
class UScriptStruct* FMyStruct::StaticStruct()
{
	if (!Z_Registration_Info_UScriptStruct_MyStruct.OuterSingleton)
	{
		Z_Registration_Info_UScriptStruct_MyStruct.OuterSingleton = GetStaticStruct(Z_Construct_UScriptStruct_FMyStruct,
		(UObject*)Z_Construct_UPackage__Script_ReflectionTest(), TEXT("MyStruct"));
	}
	return Z_Registration_Info_UScriptStruct_MyStruct.OuterSingleton;
}
template<> REFLECTIONTEST_API UScriptStruct* StaticStruct<FMyStruct>()
{
	return FMyStruct::StaticStruct();
}
struct Z_Construct_UScriptStruct_FMyStruct_Statics
{
	#if WITH_METADATA
	static const UECodeGen_Private::FMetaDataPairParam Struct_MetaDataParams[];
	#endif
	static void* NewStructOps();
	#if WITH_METADATA
	static const UECodeGen_Private::FMetaDataPairParam NewProp_Score_MetaData[];
	#endif
	static const UECodeGen_Private::FFloatPropertyParams NewProp_Score;
	static const UECodeGen_Private::FPropertyParamsBase* const PropPointers[];
	static const UECodeGen_Private::FStructParams ReturnStructParams;
};
#if WITH_METADATA
const UECodeGen_Private::FMetaDataPairParam Z_Construct_UScriptStruct_FMyStruct_Statics::Struct_MetaDataParams[] = {
	{ "ModuleRelativePath", "MyClass.h" },
};
#endif
void* Z_Construct_UScriptStruct_FMyStruct_Statics::NewStructOps()
{
	return (UScriptStruct::ICppStructOps*)new UScriptStruct::TCppStructOps<FMyStruct>();
}
#if WITH_METADATA
const UECodeGen_Private::FMetaDataPairParam Z_Construct_UScriptStruct_FMyStruct_Statics::NewProp_Score_MetaData[] = {
	{ "ModuleRelativePath", "MyClass.h" },
};
#endif
const UECodeGen_Private::FFloatPropertyParams Z_Construct_UScriptStruct_FMyStruct_Statics::NewProp_Score = { "Score",
nullptr, (EPropertyFlags)0x0010000000000000, UECodeGen_Private::EPropertyGenFlags::Float,
RF_Public|RF_Transient|RF_MarkAsNative, 1, nullptr, nullptr, STRUCT_OFFSET(FMyStruct, Score),
METADATA_PARAMS(Z_Construct_UScriptStruct_FMyStruct_Statics::NewProp_Score_MetaData,
UE_ARRAY_COUNT(Z_Construct_UScriptStruct_FMyStruct_Statics::NewProp_Score_MetaData)) };

const UECodeGen_Private::FPropertyParamsBase* const Z_Construct_UScriptStruct_FMyStruct_Statics::PropPointers[] = {
	
	(const UECodeGen_Private::FPropertyParamsBase*)&Z_Construct_UScriptStruct_FMyStruct_Statics::NewProp_Score,
};

const UECodeGen_Private::FStructParams Z_Construct_UScriptStruct_FMyStruct_Statics::ReturnStructParams = {
	(UObject* (*)())Z_Construct_UPackage__Script_ReflectionTest,
	nullptr,
	&NewStructOps,
	"MyStruct",
	sizeof(FMyStruct),
	alignof(FMyStruct),
	Z_Construct_UScriptStruct_FMyStruct_Statics::PropPointers,
	UE_ARRAY_COUNT(Z_Construct_UScriptStruct_FMyStruct_Statics::PropPointers),
	RF_Public|RF_Transient|RF_MarkAsNative,
	EStructFlags(0x00000201),
	METADATA_PARAMS(Z_Construct_UScriptStruct_FMyStruct_Statics::Struct_MetaDataParams,
	UE_ARRAY_COUNT(Z_Construct_UScriptStruct_FMyStruct_Statics::Struct_MetaDataParams))
};

UScriptStruct* Z_Construct_UScriptStruct_FMyStruct()
{
	if (!Z_Registration_Info_UScriptStruct_MyStruct.InnerSingleton)
	{
		UECodeGen_Private::ConstructUScriptStruct(Z_Registration_Info_UScriptStruct_MyStruct.InnerSingleton,
		Z_Construct_UScriptStruct_FMyStruct_Statics::ReturnStructParams);
	}
	return Z_Registration_Info_UScriptStruct_MyStruct.InnerSingleton;
}
```

The construction path for `FMyStruct` is similar to that of a `UClass`; there are no additional elements inside `FMyStruct::StaticStruct` compared to `Z_Construct_UScriptStruct_FMyStruct`. The notable difference lies in the registration process. When registering a `UStruct` inside `RegisterCompiledInInfo`, a call to `DeferCppStructOps` is made. This call registers C++ information (size, memory, alignment, etc.) for the struct as soon as the program starts. This action is equivalent to the `AutoInitialize` function's purpose inserted by the `IMPLEMENT_CLASS` macro for `UClasses`.
#### UInterface Anatomy

In Unreal Engine, a `UInterface` represents a pure interface containing only methods. This is considered purer than an abstract class in traditional C++ when the elements inside are marked with their corresponding macros. Unreal Engine relies on these macros to manage fields and functions since it doesn't handle true pure C++ fields and functions directly. The use of `UInterface` allows for a clean separation of interface definitions within the Unreal Engine framework.

```cpp
UINTERFACE(BlueprintType)
class REFLECTIONTEST_API UMyInterface : public UInterface
{
    GENERATED_UINTERFACE_BODY()
};
class IMyInterface
{
    GENERATED_IINTERFACE_BODY()
public:
    UFUNCTION(BlueprintImplementableEvent)
    void BPFunc() const;
};
```

Both `GENERATED_UINTERFACE_BODY` and `GENERATED_IINTERFACE_BODY` can be replaced with `GENERATED_BODY` to provide a default `UMyInterface(const FObjectInitializer& ObjectInitializer)` constructor implementation. However, when replacing `GENERATED_IINTERFACE_BODY`, the effect remains the same. This is because interfaces do not require a constructor, making both macros interchangeable in this context.
##### UMyInterface.generated.h

```cpp
#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_STANDARD_CONSTRUCTORS
	/** Standard constructor, called after all reflected properties have been initialized */
	NO_API UMyInterface(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
	DEFINE_ABSTRACT_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(UMyInterface)
	DECLARE_VTABLE_PTR_HELPER_CTOR(NO_API, UMyInterface);
	DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER(UMyInterface);
private:
	/** Private move- and copy-constructors, should never be used */
	NO_API UMyInterface(UMyInterface&&);
	NO_API UMyInterface(const UMyInterface&);
public:
	NO_API virtual ~UMyInterface();
#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_ENHANCED_CONSTRUCTORS
	/** Standard constructor, called after all reflected properties have been initialized */
	NO_API UMyInterface(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
private:
	/** Private move- and copy-constructors, should never be used */
	NO_API UMyInterface(UMyInterface&&);
	NO_API UMyInterface(const UMyInterface&);
public:
	DECLARE_VTABLE_PTR_HELPER_CTOR(NO_API, UMyInterface);
	DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER(UMyInterface);
	DEFINE_ABSTRACT_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(UMyInterface)
	NO_API virtual ~UMyInterface();
#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_GENERATED_UINTERFACE_BODY()
private:
	static void StaticRegisterNativesUMyInterface();
	friend struct Z_Construct_UClass_UMyInterface_Statics;
public:
	DECLARE_CLASS(UMyInterface, UInterface, COMPILED_IN_FLAGS(CLASS_Abstract | CLASS_Interface), CASTCLASS_None,
	TEXT("/Script/ReflectionTest"), NO_API)
	DECLARE_SERIALIZER(UMyInterface)

#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_GENERATED_BODY_LEGACY
	PRAGMA_DISABLE_DEPRECATION_WARNINGS
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_GENERATED_UINTERFACE_BODY()
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_STANDARD_CONSTRUCTORS
	PRAGMA_ENABLE_DEPRECATION_WARNINGS
	#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_GENERATED_BODY
	PRAGMA_DISABLE_DEPRECATION_WARNINGS
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_GENERATED_UINTERFACE_BODY()
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_ENHANCED_CONSTRUCTORS
private:
	PRAGMA_ENABLE_DEPRECATION_WARNINGS
	#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_INCLASS_IINTERFACE_NO_PURE_DECLS
protected:
virtual ~IMyInterface() {}
public:
	typedef UMyInterface UClassType;
	typedef IMyInterface ThisClass;
	static void Execute_BPFunc(const UObject* O);
	virtual UObject* _getUObject() const { return nullptr; }
	#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_INCLASS_IINTERFACE
protected:
	virtual ~IMyInterface() {}
public:
	typedef UMyInterface UClassType;
	typedef IMyInterface ThisClass;
static void Execute_BPFunc(const UObject* O);
	virtual UObject* _getUObject() const { return nullptr; }
	#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_13_PROLOG
	#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_21_GENERATED_BODY_LEGACY
PRAGMA_DISABLE_DEPRECATION_WARNINGS
public:
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_SPARSE_DATA
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_RPC_WRAPPERS
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_ACCESSORS
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_CALLBACK_WRAPPERS
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_INCLASS_IINTERFACE
public:
PRAGMA_ENABLE_DEPRECATION_WARNINGS
	#define FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_21_GENERATED_BODY
PRAGMA_DISABLE_DEPRECATION_WARNINGS
public:
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_SPARSE_DATA
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_RPC_WRAPPERS_NO_PURE_DECLS
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_ACCESSORS
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_CALLBACK_WRAPPERS
	FID_Users_StaticJPL_Documents_Unreal_Projects_ReflectionTest_Source_ReflectionTest_MyClass_h_16_INCLASS_IINTERFACE_NO_PURE_DECLS
private:
PRAGMA_ENABLE_DEPRECATION_WARNINGS
template<> REFLECTIONTEST_API UClass* StaticClass<class UMyInterface>();
```

The definition of an interface requires the use of two classes, which slightly complicates the generated information by introducing `UInterface`. Despite this, the class `UMyInterface` only inherits from `IMyInterface`, serving as a carrier of the interface type to distinguish and differentiate interfaces. According to Engine comments, there are hacks where `UClass` is created with `FImplementInterface`, holding information about all interfaces that the class implements. Additionally, it includes a pointer property that uses a hacky offset on the `VTable`. If the interface is not native, the property is null.

```cpp
class IMyInterface
{
protected:
    virtual ~IMyInterface() {}
public:
    typedef UMyInterface UClassType;
    typedef IMyInterface ThisClass;
    static void Execute_BPFunc(const UObject* O);
public:
    UFUNCTION(BlueprintImplementableEvent)
    void BPFunc() const;
};
```

Further up in the code, the generation of `UMyInterface` includes adherence to the rules of construction because `UMyInterface` inherits from `UObject` and is part of the `UObject` System. Similar to `UClass` generated code, `UInterface` code generation involves setting flags such as COMPILED_IN_FLAGS(CLASS_Abstract | CLASS_Interface).

##### UMyInterface.generated.cpp

```cpp
void EmptyLinkFunctionForGeneratedCodeMyClass() {}
...
REFLECTIONTEST_API UClass* Z_Construct_UClass_UMyInterface();
REFLECTIONTEST_API UClass* Z_Construct_UClass_UMyInterface_NoRegister();
...
void IMyInterface::BPFunc() const // To pass compilation and include error checking
{
	check(0 && "Do not directly call Event functions in Interfaces. Call Execute_BPFunc instead.");
}
void UMyInterface::StaticRegisterNativesUMyInterface()
{
}
struct Z_Construct_UFunction_UMyInterface_BPFunc_Statics
{
	#if WITH_METADATA
	static const UECodeGen_Private::FMetaDataPairParam Function_MetaDataParams[];
	#endif
	static const UECodeGen_Private::FFunctionParams FuncParams;
};
#if WITH_METADATA
const UECodeGen_Private::FMetaDataPairParam 
Z_Construct_UFunction_UMyInterface_BPFunc_Statics::Function_MetaDataParams[] =
{
	{ "ModuleRelativePath", "MyClass.h" },
};
#endif
const UECodeGen_Private::FFunctionParams
 Z_Construct_UFunction_UMyInterface_BPFunc_Statics::FuncParams = { (UObject*(*)
	())Z_Construct_UClass_UMyInterface, nullptr, "BPFunc", nullptr, nullptr, 0, nullptr, 0,
	RF_Public|RF_Transient|RF_MarkAsNative, (EFunctionFlags)0x48020800, 0, 0,
	METADATA_PARAMS(Z_Construct_UFunction_UMyInterface_BPFunc_Statics::Function_MetaDataParams,
	UE_ARRAY_COUNT(Z_Construct_UFunction_UMyInterface_BPFunc_Statics::Function_MetaDataParams)) };

UFunction* Z_Construct_UFunction_UMyInterface_BPFunc()
{
	static UFunction* ReturnFunction = nullptr;
	if (!ReturnFunction)
	{
		UECodeGen_Private::ConstructUFunction(&ReturnFunction,
		Z_Construct_UFunction_UMyInterface_BPFunc_Statics::FuncParams);
	}
	return ReturnFunction;
}
	// Same like UClass
	IMPLEMENT_CLASS_NO_AUTO_REGISTRATION(UMyInterface);
	UClass* Z_Construct_UClass_UMyInterface_NoRegister()
	{
	return UMyInterface::StaticClass();
	}
struct Z_Construct_UClass_UMyInterface_Statics
{
	static UObject* (*const DependentSingletons[])();
	static const FClassFunctionLinkInfo FuncInfo[];
	#if WITH_METADATA
	static const UECodeGen_Private::FMetaDataPairParam Class_MetaDataParams[];
	#endif
	static const FCppClassTypeInfoStatic StaticCppClassTypeInfo;
	static const UECodeGen_Private::FClassParams ClassParams;
};
UObject* (*const Z_Construct_UClass_UMyInterface_Statics::DependentSingletons[])() = {
	(UObject* (*)())Z_Construct_UClass_UInterface,
	(UObject* (*)())Z_Construct_UPackage__Script_ReflectionTest,
};
const FClassFunctionLinkInfo Z_Construct_UClass_UMyInterface_Statics::FuncInfo[] = {
	{ &Z_Construct_UFunction_UMyInterface_BPFunc, "BPFunc" }, // 2802701767
};
#if WITH_METADATA
const UECodeGen_Private::FMetaDataPairParam Z_Construct_UClass_UMyInterface_Statics::Class_MetaDataParams[] 
= {
	{ "BlueprintType", "true" },
	{ "ModuleRelativePath", "MyClass.h" },
};
#endif
const FCppClassTypeInfoStatic Z_Construct_UClass_UMyInterface_Statics::StaticCppClassTypeInfo = {
	TCppClassTypeTraits<IMyInterface>::IsAbstract,
};
const UECodeGen_Private::FClassParams Z_Construct_UClass_UMyInterface_Statics::ClassParams = {
	&UMyInterface::StaticClass,
	nullptr,
	&StaticCppClassTypeInfo,
	DependentSingletons,
	FuncInfo,
	nullptr,
	nullptr,
	UE_ARRAY_COUNT(DependentSingletons),
	UE_ARRAY_COUNT(FuncInfo),
	0,
	0,
	0x001040A1u,
	METADATA_PARAMS(Z_Construct_UClass_UMyInterface_Statics::Class_MetaDataParams,
	UE_ARRAY_COUNT(Z_Construct_UClass_UMyInterface_Statics::Class_MetaDataParams))
};

UClass* Z_Construct_UClass_UMyInterface()
{
	if (!Z_Registration_Info_UClass_UMyInterface.OuterSingleton)
	{
	UECodeGen_Private::ConstructUClass(Z_Registration_Info_UClass_UMyInterface.OuterSingleton,
	Z_Construct_UClass_UMyInterface_Statics::ClassParams);
	}
	return Z_Registration_Info_UClass_UMyInterface.OuterSingleton;
}
template<> REFLECTIONTEST_API UClass* StaticClass<UMyInterface>()
{
	return UMyInterface::StaticClass();
}
DEFINE_VTABLE_PTR_HELPER_CTOR(UMyInterface);
UMyInterface::~UMyInterface() {}
static FName NAME_UMyInterface_BPFunc = FName(TEXT("BPFunc")); // Definition of the name
void IMyInterface::Execute_BPFunc(const UObject* O)
{
	check(O != NULL);
	check(O->GetClass()->ImplementsInterface(UMyInterface::StaticClass()));
	UFunction* const Func = O->FindFunction(NAME_UMyInterface_BPFunc);
	if (Func)
	{
	const_cast<UObject*>(O)->ProcessEvent(Func, NULL);
	}
}
...
Delayed Registrations at the bottom of CPP file
```

`UMyInterface` is akin to the structure of `UClass`; however, it involves additional processing for the construction, definition, and operations related to adding functions for the Interface class types.

**Fields and Functions generation UClass**

I’ve added a couple fields and functions inside UMyClass but I haven’t covered the generation of this code until now. In review I've added.

```cpp
UPROPERTY()
int Health;
UFUNCTION(BlueprintCallable, Category = "Hello")
void CallableFunc();
UFUNCTION(BlueprintNativeEvent, Category = "Hello")
void NativeFunc();
UFUNCTION(BlueprintImplementableEvent, Category = "Hello")
void ImplementableFunc();
```

**UMyClass.generated.h**

```cpp
#define FID_FilePath_ReflectionTest_Source_ReflectionTest_MyClass_h_30_RPC_WRAPPERS
    virtual void NativeFunc_Implementation();
    DECLARE_FUNCTION(execNativeFunc);
    DECLARE_FUNCTION(execCallableFunc);
#define FID_FilePath_Source_ReflectionTest_MyClass_h_30_RPC_WRAPPERS_NO_PURE_DECLS
```

**UMyClass.generate.cpp**

```cpp
DEFINE_FUNCTION(UMyClass::execNativeFunc)
{
    P_FINISH;
    P_NATIVE_BEGIN;
    P_THIS->NativeFunc_Implementation();
    P_NATIVE_END;
}

DEFINE_FUNCTION(UMyClass::execCallableFunc)
{
    P_FINISH;
    P_NATIVE_BEGIN;
    P_THIS->CallableFunc();
    P_NATIVE_END;
}
```

`ImplementableFunc` is implemented in C++, indicating that UE will automatically generate the function body for ImplementableFunc in C++. This eliminates the need to define the function body explicitly here. In contrast, `NativeFunc` and `CallableFunc` are defined in the blueprint, necessitating the exposure of calls to these functions to the blueprint. Therefore, UE generates a specially prefixed function with exec to facilitate these calls within the blueprint virtual machine.

**Example of expanding the ExecCallableFunc**

```cpp
void execCallableFunc( FFrame& Stack, void*const Z_Param__Result ) 
// Function interface used by the blueprint virtual machine
{
    Stack.Code += !!Stack.Code; /* increment the code ptr unless it is null */
    {
    FBlueprintEventTimer::FScopedNativeTimer ScopedNativeCallTimer; // Blueprint timing statistics
    this->CallableFunc(); // Invoke our own implementation
    }
}
```

Different parameters will be added depending on the function signature. The above functions are all defined inside UMyClass.

**ImplementableFunc()**

```cpp
static FName NAME_UMyClass_ImplementableFunc = FName(TEXT("ImplementableFunc"));
void UMyClass::ImplementableFunc()
{
	// Implementation on the C++ side
	ProcessEvent(FindFunctionChecked(NAME_UMyClass_ImplementableFunc),NULL);
}
// Define the Static Collection Data structures for construction and registration
struct Z_Construct_UFunction_UMyClass_ImplementableFunc_Statics
{
	#if WITH_METADATA
	static const UECodeGen_Private::FMetaDataPairParam Function_MetaDataParams[];
	#endif
	static const UECodeGen_Private::FFunctionParams FuncParams;
};
#if WITH_METADATA
const UECodeGen_Private::FMetaDataPairParam
Z_Construct_UFunction_UMyClass_ImplementableFunc_Statics::Function_MetaDataParams[] = {
	{ "Category", "Hello" },
	{ "ModuleRelativePath", "MyClass.h" },
};
#endif
// Fill the Function parameters and data to pass to construct UECodeGen_Private
const UECodeGen_Private::FFunctionParams 
Z_Construct_UFunction_UMyClass_ImplementableFunc_Statics::FuncParams =
{ (UObject*(*)())Z_Construct_UClass_UMyClass, nullptr, 
"ImplementableFunc", nullptr, nullptr, 0, nullptr, 0,
RF_Public|RF_Transient|RF_MarkAsNative, 
(EFunctionFlags)0x08020800, 0, 0,
METADATA_PARAMS(Z_Construct_UFunction_UMyClass_ImplementableFunc_Statics::Function_MetaDataParams,
UE_ARRAY_COUNT(Z_Construct_UFunction_UMyClass_ImplementableFunc_Statics::Function_MetaDataParams))};


// Calling Construction UFunction to build the UFunction Object
UFunction* Z_Construct_UFunction_UMyClass_ImplementableFunc()
{
    static UFunction* ReturnFunction = nullptr;
    if (!ReturnFunction)
    {
        UECodeGen_Private::ConstructUFunction(&ReturnFunction,
        Z_Construct_UFunction_UMyClass_ImplementableFunc_Statics::FuncParams);
    }
    return ReturnFunction;
}
```

**CallableFunc()**

```cpp
struct Z_Construct_UFunction_UMyClass_CallableFunc_Statics
{
	#if WITH_METADATA
	static const UECodeGen_Private::FMetaDataPairParam Function_MetaDataParams[];
	#endif
	static const UECodeGen_Private::FFunctionParams FuncParams;
};
#if WITH_METADATA
const UECodeGen_Private::FMetaDataPairParam 
Z_Construct_UFunction_UMyClass_CallableFunc_Statics::Function_MetaDataParams[] = {
	{ "Category", "Hello" },
	{ "ModuleRelativePath", "MyClass.h" },
};
#endif
const UECodeGen_Private::FFunctionParams 
Z_Construct_UFunction_UMyClass_CallableFunc_Statics::FuncParams = { (UObject*(*)
())Z_Construct_UClass_UMyClass, nullptr, "CallableFunc", nullptr, nullptr, 0, nullptr, 0,
RF_Public|RF_Transient|RF_MarkAsNative, (EFunctionFlags)0x04020401, 0, 0,
METADATA_PARAMS(Z_Construct_UFunction_UMyClass_CallableFunc_Statics::Function_MetaDataParams,
UE_ARRAY_COUNT(Z_Construct_UFunction_UMyClass_CallableFunc_Statics::Function_MetaDataParams)) };

UFunction* Z_Construct_UFunction_UMyClass_CallableFunc()
{
	static UFunction* ReturnFunction = nullptr;
	if (!ReturnFunction)
	{
		UECodeGen_Private::ConstructUFunction(&ReturnFunction,
		Z_Construct_UFunction_UMyClass_CallableFunc_Statics::FuncParams);
	}
	return ReturnFunction;
}
```

**NativeFunc()**

```cpp
struct Z_Construct_UFunction_UMyClass_NativeFunc_Statics
{
	#if WITH_METADATA
	static const UECodeGen_Private::FMetaDataPairParam Function_MetaDataParams[];
	#endif
	static const UECodeGen_Private::FFunctionParams FuncParams;
};
#if WITH_METADATA
const UECodeGen_Private::FMetaDataPairParam 
Z_Construct_UFunction_UMyClass_NativeFunc_Statics::Function_MetaDataParams[] =
{
	{ "Category", "Hello" },
	{ "ModuleRelativePath", "MyClass.h" },
};
#endif
const UECodeGen_Private::FFunctionParams Z_Construct_UFunction_UMyClass_NativeFunc_Statics::FuncParams = { (UObject*(*)
())Z_Construct_UClass_UMyClass, nullptr, "NativeFunc", nullptr, nullptr, 0, nullptr, 0,
RF_Public|RF_Transient|RF_MarkAsNative, (EFunctionFlags)0x08020C00, 0, 0,
METADATA_PARAMS(Z_Construct_UFunction_UMyClass_NativeFunc_Statics::Function_MetaDataParams,
UE_ARRAY_COUNT(Z_Construct_UFunction_UMyClass_NativeFunc_Statics::Function_MetaDataParams)) };
UFunction* Z_Construct_UFunction_UMyClass_NativeFunc()
{
	static UFunction* ReturnFunction = nullptr;
	if (!ReturnFunction)
	{
		UECodeGen_Private::ConstructUFunction(&ReturnFunction,
		Z_Construct_UFunction_UMyClass_NativeFunc_Statics::FuncParams);
	}
	return ReturnFunction;
}
```

Following the same pattern, function declarations are gathered along with static data structures, and then the respective `UECodeGen_PrivateConstruct` namespace function is called to handle the creation of the object. Remember, these functions are defined within `UMyClass` and are collected during this process.

```cpp
const FClassFunctionLinkInfo Z_Construct_UClass_UMyClass_Statics::FuncInfo[] = {
    { &Z_Construct_UFunction_UMyClass_CallableFunc, "CallableFunc" }, // 2296517141
    { &Z_Construct_UFunction_UMyClass_ImplementableFunc, "ImplementableFunc" }, // 1412324750
    { &Z_Construct_UFunction_UMyClass_NativeFunc, "NativeFunc" }, // 1217410402
};
```

In the C++ implementation of CallableFunc, the blueprint simply calls this method, and the generated code only produces the corresponding UFunction pointer object. However, for NativeFunc and ImplementableFunc, there is no need to write their implementations in C++. To facilitate compilation and enable direct calls from the C++ side, it becomes necessary to generate a default implementation during code generation.

```cpp
void UMyClass::StaticRegisterNativesUMyClass()
{
    UClass* Class = UMyClass::StaticClass();
    static const FNameNativePtrPair Funcs[] = {
        { "CallableFunc", &UMyClass::execCallableFunc },
        { "NativeFunc", &UMyClass::execNativeFunc },
};
FNativeFunctionRegistrar::RegisterFunctions(Class, Funcs, UE_ARRAY_COUNT(Funcs));
```

Snippet of the ConstructUFunction

```cpp
FORCEINLINE void ConstructUFunctionInternal(UFunction*& OutFunction, 
const FFunctionParams& Params,
UFunction** SingletonPtr)
{
	UObject* (*OuterFunc)() = Params.OuterFunc;
	UFunction* (*SuperFunc)() = Params.SuperFunc;
	UObject* Outer = OuterFunc ? OuterFunc() : nullptr;
	UFunction* Super = SuperFunc ? SuperFunc() : nullptr;
	if (OutFunction)
	{
		return;
	}
	FName FuncName(UTF8_TO_TCHAR(Params.NameUTF8));
	#if WITH_LIVE_CODING
	// When a package is patched, it might reference a function in a class. When this happens, the existing UFunction
	// object gets reused but the UField's Next pointer gets nulled out. This ends up terminating the function list
	// for the class. To work around this issue, cache the next pointer and then restore it after the new instance
	// is created. Only do this if we reuse the current instance.
	UField* PrevFunctionNextField = nullptr;
	UFunction* PrevFunction = nullptr;
	if (UObject* PrevObject = StaticFindObjectFastInternal( /*Class=*/ nullptr, Outer, FuncName, true))
	{
		PrevFunction = Cast<UFunction>(PrevObject);
		if (PrevFunction != nullptr)
		{
			PrevFunctionNextField = PrevFunction->Next;
		}
	}
	#endif
	UFunction* NewFunction;
	if (Params.FunctionFlags & FUNC_Delegate)
	{
		if (Params.OwningClassName == nullptr)
		{
			NewFunction = new (EC_InternalUseOnlyConstructor, Outer, FuncName, Params.ObjectFlags) UDelegateFunction(
			FObjectInitializer(),
			Super,
			Params.FunctionFlags,
			Params.StructureSize
			);
		}
		else
		{
			USparseDelegateFunction* NewSparseFunction = new (EC_InternalUseOnlyConstructor, Outer, FuncName,
			Params.ObjectFlags) USparseDelegateFunction(
			FObjectInitializer(),
			Super,
			Params.FunctionFlags,
			Params.StructureSize
			);
			NewSparseFunction->OwningClassName = FName(Params.OwningClassName);
			NewSparseFunction->DelegateName = FName(Params.DelegateName);
			NewFunction = NewSparseFunction;
		}
	}
	else
	{
		NewFunction = new (EC_InternalUseOnlyConstructor, Outer, FuncName, Params.ObjectFlags) UFunction(
		FObjectInitializer(),
		Super,
		Params.FunctionFlags,
		Params.StructureSize
		);
	}
	OutFunction = NewFunction;
	#if WITH_LIVE_CODING
	NewFunction->SingletonPtr = SingletonPtr;
	if (NewFunction == PrevFunction)
	{
		NewFunction->Next = PrevFunctionNextField;
	}
	#endif
	#if WITH_METADATA
	AddMetaData(NewFunction, Params.MetaDataArray, Params.NumMetaData);
	#endif
	NewFunction->RPCId = Params.RPCId;
	NewFunction->RPCResponseId = Params.RPCResponseId;
	ConstructFProperties(NewFunction, Params.PropertyArray, Params.NumProperties);
	NewFunction->Bind();
	NewFunction->StaticLink();
}
```

To construct the skeleton of the `UFunction*`, the initial step involves UMyClass constructing itself and subsequently calling `CreateLinkAddChildFunctionToMap`.

```cpp
void UClass::CreateLinkAndAddChildFunctionsToMap
(const FClassFunctionLinkInfo* Functions, uint32 NumFunctions)
{
    for (; NumFunctions; --NumFunctions, ++Functions)
    {
       const char* FuncNameUTF8 = Functions->FuncNameUTF8;
       UFunction* Func = Functions->CreateFuncPtr();
       Func->Next = Children;
       Children = Func;
       AddFunctionToFunctionMap(Func, FName(UTF8_TO_TCHAR(FuncNameUTF8)));
    }
}
```

The `TMap` stores a `FName` for the function associated with a `UFunction` pointer. The `UObject` pointer holds the reference to the class it belongs to, and the `Super` function refers to the `SuperStruct` for the `UFunction`, acting as a superclass or interface. The engine caches parent functions for child classes as a runtime optimization. This caching mechanism is utilized in `FindFunctionByName` for Blueprint function usage.
#### Summary

In summary, the generated code structure encompasses functions and static data structures used by various types such as `UEnum`, `UMyStruct`, `UMyInterface`, `UFunction`, and others. These are defined and added as properties and methods within `UMyClass`. For Enums, names and values are recorded; for Structs, names and byte offsets of each property are recorded. Function pointers are defined for each function or wrapper function, including default constructors.
