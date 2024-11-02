### Preface

This document outlines the Gameplay Architecture for Unreal Engine. The document attempts to peal the layers of the gameplay systems in order to understand where gameplay logic should be written.

Why is this useful? 

I hope this docuement provides additional insights for new or existing gameplay programmers. The idea is to eleviate the mental burden from discretly mapping Unreals gameplay architecture and instead view the gameplay architecture from a software systems perspective. Having a systems mental map should help determine how to design or implement gameplay systems for the editor or during gameplay.

#### Requirements

- A Beginer/Intermediate level of C++ is necessary to comprehend some of the concepts presented in this document.
- I strongly recommend downloading the engine source code. This is crucial for inserting breakpoints to see the call stack for things like the Actor Life Cycle, Editor Startup, MapLoading Etc..
