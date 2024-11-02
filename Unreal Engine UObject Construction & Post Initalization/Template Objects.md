### Template Objects

Before delving deeper into `FObjectInitializer`, it's important to introduce the prototype design pattern. Understanding this pattern will provide context for Unreal Engine's extensive usage of template objects and clarify when objects are instantiated and initialized from these templates.

**Prototype Pattern**

*"The Prototype Pattern in programming is a design approach where objects are copied or cloned rather than created a new. It involves defining a prototype interface or abstract class with a method for cloning, then implementing concrete classes that adhere to this interface. By cloning existing objects, the pattern promotes efficiency, especially when creating instances is resource-intensive. It also fosters flexibility, allowing for dynamic modifications to class structures at runtime. This pattern is particularly useful in scenarios where creating new objects is costly or where object creation needs to be dynamic and customizable."*

Unreal Engine's prototype-like pattern is closely tied to its reflection system. `UClass`, which holds type information at runtime, associates a template object, typically known as a Class Default Object (CDO), with itself. The CDO serves as the object that instances the default state for any new objects of that class.

![[Unreal Engine UObject Construction & Post Initalization/Diagrams/TemplateObjects.svg]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/bb5af558180d9154bb3f16f7a463af4dd2761ca3/Unreal%20Engine%20UObject%20Construction%20%26%20Post%20Initalization/Diagrams/TemplateObjects.svg)
#### Template CDO and Archetypes Flags

A Template Object can manifest in several forms:

1. **Archetype object** (RF_ArchetypeObject).
2. **CDO object** (RF_ClassDefaultObject).
3. **Object instance** with or without RF_ArchetypeObject or RF_ClassDefaultObject flags.

Expressed differently:

- A Template object can serve as both an Archetype and a CDO.
- A Template object can solely function as a CDO.
- A Template object can be an instantiated object, which may also act as an archetype or a CDO.

Another crucial aspect of `NewObject` involves the use of a provided template object:

- `UClass* Class`: Specifies the class of the object to create. The return value of `Class->GetDefaultObject()` is the CDO.
- `UObject* Template`: If specified, the property values from this object will be copied to the new object, and the new object's ObjectArchetype value will be set to this object. If nullptr, the class default object (CDO) is used instead.
- `EObjectFlags InFlags`: Specifies the ObjectFlags to assign to the new object.

The key takeaway is: If a Template Object is specified, it takes precedence over the CDO for copying default property values to newly instantiated objects.

```cpp
FObjectInitializer::~FObjectInitializer()
{
	//...
	UObject* Defaults = ObjectArchetype ? ObjectArchetype : BaseClass->GetDefaultObject(false);
	InitProperties(Obj, BaseClass, Defaults, bCopyTransientsFromClassDefaults);
	//...
}
```

Archetype objects are initialized from `uassets`, which requires understanding the serialization system, specifically `FArchive`. I plan to add a section on this topic later on. An illustrative example of archetype usage in the engine source code involves loading a UObject from disk using the function `CreateExport`.

```cpp
UObject* FLinkerLoad::CreateExport( int32 Index )
{
	FObjectExport& Export = ExportMap[ Index ];
	...
	// RF_ArchetypeObject and other flags
	EObjectFlags ObjectLoadFlags = Export.ObjectFlags;
	...
	Export.Object = StaticConstructObject_Internal
	(
		LoadClass,
		ThisParent,
		NewName,
		ObjectLoadFlags,
		EInternalObjectFlags::None,
		Template
	);
}
```
