# Summary

If you have diligently navigated through the engine source code, the UML layout provided should elucidate the structure of the Type System.

![[Unreal Engine Reflection System/Reflection Diagrams/TypeSystemLayout.png]](https://github.com/staticJPL/Unreal-Engine-Documentation/blob/0faba8471b6c254202b9e65342ad9fd99ac38c72/Unreal%20Engine%20Reflection%20System/Reflection%20Diagrams/TypeSystemLayout.png)

#### Recap

  
In review, the high-level process of the Unreal Engine reflection system has been dissected into distinct sections: "Generation," "Collection," and "Registration." The code generation occurs through the Unreal Build Tool, parsing predefined macros that encapsulate native types for type reflection. Once the types are tokenized, special .gen files (cpp and header files) are generated, containing definitions of data structures that hold type information, function pointers for runtime construction, and operators for memory allocation of UObjects.

The Collection phase statically gathers type information into a global deferred singleton registry through Dynamic Library Linking of Engine modules. The modular design facilitates the orderly linking and processing of each module, each capable of containing reflection information.

In Editor mode, Registration initiates with the Engine startup and loading of the Core Module first. This process sets up the core types (UClass, UStruct, UEnum, and UInterface) required to handle subsequent types loaded from other modules. Finally, at the conclusion of Registration, Bind linking is employed to organize and optimize the structure, providing additional performance gains and convenience.

**"UClass is a UObject, and UObject is UClass"**

The fundamental concept to grasp is that UClass* serves as the primary intersection connecting various elements, while UObject stands as the low-level representation for all actions within Unreal Engine. A profound understanding of this relationship traces back to the foundational aspects of the Garbage Collection process and the proper allocation or destruction of UObjects. This comprehension extends its influence into serialization, Actor and component architecture, gameplay systems, networking, and various other aspects. Emphasizing the life cycle of UClass is beneficial:

1. **Memory Structure Definition:** Allocate a segment of memory and invoke the UClass Constructor.
2. **Registration:** Assign a name and integrate it into the object system through Deferred Registry.
3. **Object Construction:** Populate the object with details concerning properties, functions, interfaces, and metadata outlined in the .gen files.
4. **Binding:** Similar to the binding of the One Ring, defines the addresses of attributes, sorts them, and optimizes the overall structure for performance, convenience, and access.
5. **Token Stream Construction:** Specify how UClass references other objects. Construct a reference tree, serving as the core of the garbage collection process. Additionally, this reference tree aids in the analysis of other objects referenced by the given object.
