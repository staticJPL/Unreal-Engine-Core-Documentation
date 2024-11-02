### Preface

This document outlines the architecture of constructing UObject's. It also details the Post Initialization process used for sub-objects marked with Reflection macros contained inside UClass.

The concepts are consolidated as reference for my future self. It includes my own personal research including segments of articles I've pieced together to ensure no gaps in knowledge are missed when navigating other systems that are involved.

Why did I decide to do this? 

Well once again, during my time debugging, and implementing gameplay systems with Unreal Engine I encountered many black box scenarios where I was pulled into the deep end of the source code. This usually was a result of programming something incorrectly or hacking around the intended design of the Engine. One example nested sub-object components.

The end goal is to give C++ programmers some additional visibility when navigating the guard rails of the Engine. Especially for something a simple as constructing or instantiating a object. With so many layers of abstraction in Unreal it can make programmer want to bury their head into the sand until they suffocate from their own codebase. Which intern creates a need to understand what's actually happening behind the scenes. 
#### Requirements

- An intermediate level of C++ is necessary to comprehend the concepts presented in this document. When I began, I possessed just enough C++ knowledge but found it beneficial to review some lower-level aspects along the way to grasp certain tricks employed with the language. As you progress through the reading process, you may discover and learn more.
- I strongly recommend downloading the engine source code. This is crucial for inserting if statements in scenarios where you can set breakpoints for a specific step in the construction process.