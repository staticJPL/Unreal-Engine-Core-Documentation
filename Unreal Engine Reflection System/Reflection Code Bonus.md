# Reflection Bonus

What would be the point of learning all the knowledge behind Unreal Engine's reflection system without practical applications? This section will contain useful examples of its usage that you may use or encounter examples of in the engine source code.

##### Getting Objects by Type

```cpp
TArray<UObject*> UClassTypeArray;
TArray<UObject*> UEnumTypeArray;
TArray<UObject*> UScriptStructArray;
TArray<UObject*> UStaticStruct;
// This Loads all UClass types into the array including Interface
GetObjectsOfClass(UClass::StaticClass(), UClassTypeArray);
// Load all UEnum objects
GetObjectsOfClass(UEnum::StaticClass(), UEnumTypeArray);
// Load UScriptStructs
GetObjectsOfClass(UScriptStruct::StaticClass(), UScriptStructArray);   
// Will load UScript Struct and UStructs
GetObjectsOfClass(UStaticStruct::StaticClass(), UStaticStruct);
```

This will grab all the `UClass`, `UEnum` and `UStruct` types and load them into their associated array.
##### FindObject

```cpp
/**
 * Find an optional object.
 * @see StaticFindObject()
 */
template< class T > 
inline T* FindObject( UObject* Outer, const TCHAR* Name, bool ExactClass=false )
{
	return (T*)StaticFindObject( T::StaticClass(), Outer, Name, ExactClass );
}

// Example Code Begin Play Actor
UClass* ClassObj = FindObject<UClass>(ANY_PACKAGE,TEXT("MyClass"),false);
```

`FindObject` takes in three parameters the two important ones are the `Outer` and the `FName` of the object. `ANY_PACKAGE` is deprecated and in the future you will need to pass valid package object for the object you're referencing with `FName`. 

##### TFieldIterator

```cpp
const UScriptStruct* MyStruct = FMyStruct::StaticStruct();
for(TFieldIterator<FProperty> It(MyStruct); It; ++It)
{
	FProperty* Prop = *It;
}
```

In the above example, if you remember I declared a `UStruct` type called "FMyStruct". In the code snippet I get a reference by calling `StaticStruct`. Next I traverse inner properties using the `TFieldIterator` . You can also do this for a UClass also.

```cpp
const UClass* MyClass = UMyClass::StaticClass();
for(TFieldIterator<UFunction> It(MyClass); It; ++It)
{
	UFunction* Func = *It;
	for(TFieldIterator<FProperty> Jt(Func); Jt; ++Jt)
	{
		// Function Paramters
		FProperty* parameter = *Jt;
		if(Jt->PropertyFlags & CPF_ReturnParm)
		{
			// Returns parameters
		}
	}
}	
```

Traversing UMyClass to find all the UFunction types declared inside. Once you get the UFunction types you can take another step further and iterate through UFunction properties.

Traversing interfaces associated with a UClass Object that has interfaces

```cpp
// Recall Interace is a UClass type
const UClass* MyInterfaceObj = UMyInterface::StaticClass();
for(const FImplementedInterface& i: MyInterfaceObj->Interfaces)
{
	UClass* interfaceClass = i.Class;
}
```

**Traversing UEnum** 

```cpp
UEnum* MyEnum = FindObject<UEnum>(nullptr,TEXT("/Script/ReflectionTest.EMyEnum"),false);
for(int i = 0; i < MyEnum->NumEnums(); ++i)        
{                                                  
	FName enumName = MyEnum->GetNameByIndex(i);    
	int value = MyEnum->GetValueByIndex(i);        
}                                                  
```

**Traversing UMetaData**

```cpp
UMetaData* metaData = MyClass->GetOutermost()->GetMetaData();        
TMap<FName,FString>* keyValues = metaData->GetMapForObject(MyClass);

if(keyValues != nullptr && keyValues->Num() > 0)                     
{                                                                    
	// for loop values                                               
	for(const auto& i : *keyValues)                                  
	{                                                                
		FName key = i.Key;                                           
		FString value = i.Value;                                     
	}                                                                
}                                                                   
```

Inside UMyClass I have an `int health` property so calling FindPropertyByName is simple and useful.

```cpp
UClass* MyClass = UMyClass::StaticClass();
FProperty* Health = MyClass->FindPropertyByName(FName(TEXT("Health")));
```

##### Traversing the Inheritance Chain

```cpp
void GetDerivedClasses(const UClass* ClassToLookFor, TArray<UClass*>& Results, bool bRecursive)
{
	FUObjectHashTables& ThreadHash = FUObjectHashTables::Get();
	FHashTableLock HashLock(ThreadHash);

	if (bRecursive)
	{
		RecursivelyPopulateDerivedClasses(ThreadHash, ClassToLookFor, Results);
	}
	else
	{
		TSet<UClass*>* DerivedClasses = ThreadHash.ClassToChildListMap.Find(ClassToLookFor);
		if ( DerivedClasses )
		{
			Results.Append( DerivedClasses->Array() );
		}
	}
}
```

The derived classes function is a helper function that can allow you to obtain all subclasses under a class.

```cpp
TArray<FString> classNames;
classNames.Add(MyClass->GetName());

UStruct* superClass = MyClass->GetSuperStruct();

while(superClass)
{
	classNames.Add(superClass->GetName());
	superClass = superClass->GetSuperStruct();
	
}
```

SuperStructs chain up the inheritance hierarchy. Thus, we can use a while loop on the superClass until termination with a null pointer, indicating that we've reached the base class in the hierarchy.

**Finding Subclass Implemented Interfaces**

```cpp
TArray<UObject*> result;
GetObjectOfClass(UClass::StaticClass(),result);

TArray<UClass*> classes;
for(UObject* obj : result)
{
	UClass* classObj = Cast<UClass>(obj);
	if(classObj->ImplementsInterface(interfaceClass)){
		classes.Add(classObj);
	}
}
```

**Getting and Setting Attribute Values**

From the engine source code there are helper functions that are overloaded with void* and UObject*

```cpp
template<typename ValueType>  
FORCEINLINE ValueType* ContainerPtrToValuePtr(UObject* ContainerPtr, int32 ArrayIndex = 0) const  
{  
    return (ValueType*)ContainerUObjectPtrToValuePtrInternal(ContainerPtr, ArrayIndex);  
}  
template<typename ValueType>  
FORCEINLINE ValueType* ContainerPtrToValuePtr(void* ContainerPtr, int32 ArrayIndex = 0) const  
{  
    return (ValueType*)ContainerVoidPtrToValuePtrInternal(ContainerPtr, ArrayIndex);  
}
```

```cpp
template<typename ValueType>
FORCEINLINE ValueType const* ContainerPtrToValuePtr(UObject const* ContainerPtr, int32 ArrayIndex = 0) const
{
	return ContainerPtrToValuePtr<ValueType>((UObject*)ContainerPtr, ArrayIndex);
}
template<typename ValueType>
FORCEINLINE ValueType const* ContainerPtrToValuePtr(void const* ContainerPtr, int32 ArrayIndex = 0) const
{
	return ContainerPtrToValuePtr<ValueType>((void*)ContainerPtr, ArrayIndex);
}
```

`FProperty` inside UnrealTypes.h includes several additional helper functions. The question arises: what does Unreal mean by "Container"? You've encountered them before in the "Collection" section when we explored derived types of `FProperty` (such as `FNumericProperty`, `FFloatProperty`, `FDoubleProperty`, etc.).

```cpp
class UMyObject
{
GENERATED_BODY()
private:
int32 x = 0;
int32 y = 0;
}
int32 GimmeY(UMyObject* Obj)
{
return *(int32*)(Obj + 4); // This is the offset in Action
}
FProperty* RelfectionOfY;

// This gets us the pointer to the type 
ReflectionOfY->ContainerPtrToValuePtr<int32>(Obj);
```
  
You can obtain the `Offset_Internal` for the container type, as mentioned before. It was briefly noted that "FField gets an Int32 Offset when `Linker()` is called. Otherwise, it receives the offset from UHT `offset_of(type, member)`. This memory offset is set during `STRUCT_OFFSET()`.

```cpp
FORCEINLINE void* ContainerVoidPtrToValuePtrInternal(void* ContainerPtr, int32 ArrayIndex) const
{
	check(ArrayIndex < ArrayDim);
	check(ContainerPtr);

	if (0)
	{
		// in the future, these checks will be tested if the property is NOT relative to a UClass
		check(!Cast<UClass>(GetOuter())); // Check we are _not_ calling this on a direct child property of a UClass, you should pass in a UObject* in that case
	}

	return (uint8*)ContainerPtr + Offset_Internal + ElementSize * ArrayIndex;
}

FORCEINLINE void* ContainerUObjectPtrToValuePtrInternal(UObject* ContainerPtr, int32 ArrayIndex) const
{
	check(ArrayIndex < ArrayDim);
	check(ContainerPtr);

	// in the future, these checks will be tested if the property is supposed be from a UClass
	// need something for networking, since those are NOT live uobjects, just memory blocks
	check(((UObject*)ContainerPtr)->IsValidLowLevel()); // Check its a valid UObject that was passed in
	check(((UObject*)ContainerPtr)->GetClass() != NULL);
	check(GetOuter()->IsA(UClass::StaticClass())); // Check that the outer of this property is a UClass (not another property)

	// Check that the object we are accessing is of the class that contains this property
	checkf(((UObject*)ContainerPtr)->IsA((UClass*)GetOuter()), TEXT("'%s' is of class '%s' however property '%s' belongs to class '%s'")
		, *((UObject*)ContainerPtr)->GetName()
		, *((UObject*)ContainerPtr)->GetClass()->GetName()
		, *GetName()
		, *((UClass*)GetOuter())->GetName());

	if (0)
	{
		// in the future, these checks will be tested if the property is NOT relative to a UClass
		check(!GetOuter()->IsA(UClass::StaticClass())); // Check we are _not_ calling this on a direct child property of a UClass, you should pass in a UObject* in that case
	}

	return (uint8*)ContainerPtr + Offset_Internal + ElementSize * ArrayIndex;
}
```

`ElementSize` is the memory size allocated and you obtain where the value is stored in the array with a `ArrayIndex`. 

```cpp
// Attributes can contain subobjects
UObject* subObject = objectProperty->GetObjectPropertyValue_InContainer(object);
```

**Details Panel Properties (Editor)**

In some scenario you might want to deal with some properties in a details panel if you're building a tool for the editor.

```cpp
/** 
 * export this structure 
 * @return true if the copy was exported, otherwise it will fall back to FStructProperty::ExportTextItem
 */
virtual bool ExportTextItem(FString& ValueStr, const void* PropertyValue, const void* DefaultValue, class UObject* Parent, int32 PortFlags, class UObject* ExportRootScope) = 0;
```

```cpp
/** 
 * import this structure 
 * @return true if the copy was imported, otherwise it will fall back to FStructProperty::ImportText
 */
virtual bool ImportTextItem(const TCHAR*& Buffer, void* Data, int32 PortFlags, class UObject* OwnerObject, FOutputDevice* ErrorText) = 0;
```

These methods are utilized in various scenarios, primarily for copying and pasting content. Unreal takes this clipboard information, serializes it into strings, and then passes it accordingly.

```cpp
// Example Export Text Item 
FString ExportedPropertyString
someprop->ExportTextItem(ExportedPropertyString, someprop->ContainerPtrToValuePtr<void*>(object),nullptr,(UObject*)Object,PPF_None);
```

```cpp
// Example Import Text Item
FString valueStr;
prop->ImportText(*valueStr, prop->ContainerPtrToValuePtr<void*>(obj), PPF_None, obj);
```

##### Calling Functions on Reflected Types

The advantages of having a reflection system include the ability to invoke functions within the reflected type. If you possess an instance of UFunction within an object, there are methodologies through which we can invoke it using reflection.

There is an example in PyUtil.cpp `bool InvokeFunctionCall` and Dazhao's example looks similar. 

Two function signatures 

```cpp
// PyUtil.cpp
bool InvokeFunctionCall(UObject* InObj, const UFunction* InFunc, void* InBaseParamsAddr, const TCHAR* InErrorCtxt)
```

```cpp
// Dazhao example
int32 InvokeFunction(UObject* obj, FName functionName,float param1)
```

There are several ways to achieve this. To implement a custom invoke function, the initial step involves obtaining a pointer to the UObject, followed by acquiring a pointer to a UFunction. Subsequently, you would invoke `UObject::ProcessEvent()`. `ProcessEvent` is a convenient function in Unreal Engine that internally manages Blueprint VM issues. Additionally, there are alternative low-level methods such as `CallFunction` and `Invoke`.

Dazhao's example is valuable as it elucidates the construction of a parameter struct required to pass to `ProcessEvent`. This addresses a common query raised by developers.

```cpp
int32 UMyClass::Func(float param1); 

UFUNCTION(BlueprintCallable)
int32 InvokeFunction(UObject* obj, FName functionName,float param1)
{
	// Struct to wrap parameter return values for memory offset reasons.
	// Process Event Requires UStruct of Params
    struct MyClass_Func_Parms   
    {
        float param1;
        int32 ReturnValue;
    };
    UFunction* func = obj->FindFunctionChecked(functionName);
    MyClass_Func_Parms params;
    params.param1=param1;
    obj->ProcessEvent(func, &params);
    return params.ReturnValue;
}

int r=InvokeFunction(obj,"Func",123.f);
```

I experimented with the `InvokeFunctions` to make calls to some test reflection functions through a derived `TestActor` during BeginPlay().  

I test two different scenarios.

1. TestActor calls to it's own function at runtime with `FindFunction` and `ProcessEvent`.
2. TestActor calls to another UClass object `UMyClass`. 

The functions called are defined both in TestActor and UMyClass as a health function.

```cpp
// Header
UFUNCTION()  
int ClassHealthFunc(int inHealth);

// Cpp
int UMyClass::ClassHealthFunc(int inHealth)  
{  
    int Health = 0;
    Health = inHealth;  
    UE_LOG(LogTemp,Warning,TEXT("MyClass Health Func Called: %i"),Health);  
  
    return Health;  
}
```

```cpp
// Header
UFUNCTION()  
int HealthFunc(int inHealth);

// Cpp
int ATestActor::HealthFunc(int inHealth)  
{  
    int Health = 0;  
    Health = inHealth
    UE_LOG(LogTemp,Warning,TEXT("AActor Health Func Called: %i"),Health);  
    return Health;  
}
```

Since the health function takes an `int` parameter and returns an `int`, the struct's memory layout should align with the order of the function's parameters.

```cpp
void ATestActor::InvokeCallableFunc(UObject* obj,FName inName,int param,UFunction* inFunc)  
{  
    struct FuncParams  
    {  
       // Order matters here as it was Linked in memory  
       int funcParamHealth;  
       int ReturnValue;  
    };
    
    FuncParams HealthFunParams;  
    HealthFunParams.funcParamHealth = param;  
    HealthFunParams.ReturnValue = 0;  
  
    UFunction* objFunc = nullptr;  
  
    if(obj != nullptr)  
    {  
		objFunc = obj->FindFunction(inName);  
		obj->ProcessEvent(objFunc,&HealthFunParams);  
	    UE_LOG(LogTemp,Warning,TEXT("Actor Health Func Called"));  
    }
    
    if(inFunc != nullptr)  
    { 
		FFrame Frame(nullptr,inFunc,&HealthFunParams,nullptr,inFunc->ChildProperties);
		inFunc->CallFunction(Frame,&HealthFunParams + inFunc->ReturnValueOffset,inFunc);  
		UE_LOG(LogTemp,Warning,TEXT("Call Func Obj Called"));  
    }
}
```

```cpp
void ATestActor::BeginPlay()  
{  
	Super::BeginPlay();
	
	// Load the UMyClass Object safely to ensure we don't get an editor crash 
	auto _MyClass = TSoftObjectPtr<UClass>(FSoftClassPath(TEXT("Class'/Script/TestReflect.MyClass'"))).LoadSynchronous();   
	// Call Test Actors Internal Functon 
	InvokeCallableFunc(this,"HealthFunc",120,nullptr);  
	
	if(_MyClass)  
	{  
		UStruct* ClassStruct = _MyClass->GetOwnerStruct();  
		UFunction* MyClassFunc = nullptr;
			
		for(TFieldIterator<UFunction> i(ClassStruct); i; ++i)  
		{  
			UFunction* func = *i;  
			if(func->GetFName() == TEXT("ClassHealthFunc"))
			{             
				  MyClassFunc = func;         
			}  
		}
		
		if(MyClassFunc){          
			InvokeCallableFunc(nullptr,"",250,MyClassFunc);  
		}    
	}
}
```

In the first scenario, I called `FindFunction` with the `FName` associated with its internal `FuncMap` of TestActor. This returns a pointer to the UFunction if it exists in the FuncMap. After obtaining the UFunction pointer, you can pass the parameter struct along with the function pointer to `ProcessEvent` and invoke it.

In the second scenario, I initially attempted to call UMyClass's Health Function from TestActor by obtaining a reference through `UMyClass:StaticClass`. With the UClass* obtained, I called `FindFunction`. I was surprised to find that I was getting a `nullptr` from the FuncMap!

This is because I derived UMyClass from UObject, so `FindFunction` calls `GetClass()` on UMyClass, which returns the Class Native type. The class native type will not have my function in the `FuncMap`.

Another issue I came across was that during runtime, if I obtained a reference to the UMyClass object and ended the game session in the editor, it would crash due to the GC looking for the index tied to a missing object reference. To address this, I decided to get a soft reference to the object by doing:

```c
auto _MyClass = TSoftObjectPtr<UClass>(FSoftClassPath(TEXT("Class'/Script/TestReflect.MyClass'"))).LoadSynchronous()
```

Since the issue with `FindFunction` searching the native class `FuncMap`, I instead traverse the properties until I get a reference to the UFunction* type with the same `FName`.

To invoke UMyClass's Health function, I set up a call with `CallFunction`. CallFunction is a lower-level method that requires an `FFrame`. `FFrame` allocates a block of memory (like an assembly-level stack frame) including any `ChildProperties` and the offset to the return value in the struct parameters.

```cpp

FFrame frame(nullptr, func, &params, nullptr, func->Children);
obj->CallFunction(frame, &params + func->ReturnValueOffset, func);

FFrame frame(nullptr, func, &params, nullptr, func->Children);
func->Invoke(obj, frame, &params + func->ReturnValueOffset);

```

These function calls allow you to invoke static functions. Knowing that functions and events in blueprints will be compiled into a generated UFunction object, these methods can directly call member functions and custom events in blueprints.

Dazhao took another step further and defined a wonderful invoke function that takes a tuple without the need to build a parameter struct and a fixed function prototype.

```cpp
template<typename... TReturns, typename... TArgs>
void InvokeFunction(UClass* objClass, UObject* obj, UFunction* func, TTuple<TReturns...>& outParams, TArgs&&... args)
{
    objClass = obj != nullptr ? obj->GetClass() : objClass;
    UObject* context = obj != nullptr ? obj : objClass;
    uint8* outPramsBuffer = (uint8*)&outParams;

    if (func->HasAnyFunctionFlags(FUNC_Native)) //quick path for c++ functions
    {
        TTuple<TArgs..., TReturns...> params(Forward<TArgs>(args)..., TReturns()...);
        context->ProcessEvent(func, &params);
        //copy back out params
        for (TFieldIterator<UProperty> i(func); i; ++i)
        {
            UProperty* prop = *i;
            if (prop->PropertyFlags & CPF_OutParm)
            {
                void* propBuffer = prop->ContainerPtrToValuePtr<void*>(&params);
                prop->CopyCompleteValue(outPramsBuffer, propBuffer);
                outPramsBuffer += prop->GetSize();
            }
        }
        return;
    }

    TTuple<TArgs...> inParams(Forward<TArgs>(args)...);
    void* funcPramsBuffer = (uint8*)FMemory_Alloca(func->ParmsSize);
    uint8* inPramsBuffer = (uint8*)&inParams;

    for (TFieldIterator<UProperty> i(func); i; ++i)
    {
        UProperty* prop = *i;
        if (prop->GetFName().ToString().StartsWith("__"))
        {
            //ignore private param like __WolrdContext of function in blueprint funcion library
            continue;
        }
        void* propBuffer = prop->ContainerPtrToValuePtr<void*>(funcPramsBuffer);
        if (prop->PropertyFlags & CPF_OutParm)
        {
            prop->CopyCompleteValue(propBuffer, outPramsBuffer);
            outPramsBuffer += prop->GetSize();
        }
        else if (prop->PropertyFlags & CPF_Parm)
        {
            prop->CopyCompleteValue(propBuffer, inPramsBuffer);
            inPramsBuffer += prop->GetSize();
        }
    }

    context->ProcessEvent(func, funcPramsBuffer);   //call function
    outPramsBuffer = (uint8*)&outParams;    //reset to begin

    //copy back out params
    for (TFieldIterator<UProperty> i(func); i; ++i)
    {
        UProperty* prop = *i;
        if (prop->PropertyFlags & CPF_OutParm)
        {
            void* propBuffer = prop->ContainerPtrToValuePtr<void*>(funcPramsBuffer);
            prop->CopyCompleteValue(outPramsBuffer, propBuffer);
            outPramsBuffer += prop->GetSize();
        }
    }
}
```

His function handles these scenarios.

1. Templated class with `TTuple`: This is used to hold multiple parameters, eliminating the need to constantly define a parameter struct. It leverages multiple return values. To create a `TTuple`, you need to define the parameters in the template and then use an `rvalue` reference. Passing the rvalue reference can be done with `MoveTemp` .
    
2. Handling situations where you can call member functions and static functions defined in C++. It supports calling member functions, events, and functions in a blueprint library. Blueprint functions may have more input/output values than in C++, so pushing parameters to the blueprint VM requires extra steps compared to C++. This involves allocating a piece of memory on the stack as a memory parameter when the function is invoked. This memory must be initialized from the function parameter and set before the reflection call. The value needs to be copied back to the return parameter after the reflection call.
    
3. For calling a static function where UObject is not needed, a nullptr can be passed, and the corresponding UClass can be used as a context calling object.
    
4. In scenarios with no parameters, a variable parameter template partial specialization and overloading can be used to define variations.
    

He further mentions that a call to a blueprint library just provides the function name and parameters. Therefore, you can create a wrapper like so.

```cpp

template<typename... TReturns, typename... TArgs>
static void InvokeFunctionByName(FName functionName,TTuple<TReturns...>& outParams,TArgs&&... args)
{
    UFunction* func = (UFunction*)StaticFindObjectFast(UFunction::StaticClass(), nullptr, functionName, false, true, RF_Transient); //exclude SKEL_XXX_C:Func
    InvokeFunction<TReturns...>(func->GetOuterUClass(), nullptr, func, outParams,Forward<TArgs>(args)...);
}
```

Dazhao adds that you cannot simply use `FindObject` for searching. This is because Blueprint generates two classes after compilation and before cooking (BPMyClass_C and SKEL_BPMyClass_C). Both of these have a UFunction pointer object with the same name. This can be filtered out by checking the ObjectFlags for RF_Transient to identify the correct UFunction object.
##### Runtime Modification Possibilities

In Unreal Engine, you've witnessed the construction process of various object types. This discussion aims to recognize that, with the reflection system, there exists the capability to dynamically alter, modify, and even register a type during runtime in the engine, after the regular type system registration process is executed. In C#, this is analogous to the emit method, where you can dynamically emit transient and persisting assemblies at runtime. Dazhao provides examples of what can be achieved with this capability:

1. Modifying MetaData information of an FField to change how the data is displayed in the editor. This is useful for building tools or plugins that you want to expose in the editor.
2. Dynamically modifying flags. There are several flags such as FPropertyFlags, StructFlags, and ClassFlags, and modifying these at runtime can alter behavior in the editor.
3. For UEnum, dynamically adding or removing fields.
4. Dynamically adding attributes to a Struct or Class.
5. Creating structures within a blueprint.
6. Reserving memory with FProperty Offset.
7. Dynamically exposing functions to Blueprints.
8. Dynamically registering a new structure and/or constructing a UScriptStruct.
#### Summary

Deeper knowledge of the reflection type system is crucial for an engine programmer seeking to explore the inner workings of Unreal Engine. Additionally, as a game programmer, understanding this system becomes essential when developing tools that open new opportunities for enhanced gameplay systems. Regardless of the role, there are significant rewards for a programmer equipped with a comprehensive understanding of this framework. This knowledge serves as the foundation for crucial aspects such as UObject garbage collection, serialization, and seamless integration with the blueprint system.