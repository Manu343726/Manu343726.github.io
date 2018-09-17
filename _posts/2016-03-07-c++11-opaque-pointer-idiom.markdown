---
layout: post
title: C++11 Opaque Pointer Idiom
modified:
categories: 
excerpt:
tags: []
image:
  feature:
date: 2016-03-07T09:55:00+01:00
---

*This was originally a wiki entry I was writing for the [By Tech](http://www.by.com.es/) internal development wiki this morning, but I thought it's the kind of simple and useful tip that deserves a blog post...*

## Opaque Pointer (PIMPL) Idiom

[The opaque pointer idiom](https://en.wikipedia.org/wiki/Opaque_pointer) provides a way to isolate the implementation of a class from its interface, by forwarding interface calls to a private type (The *implementation*) that's declared as part of class definition:

    class MyClass
    {
        ... // Interface
    private:
        class Implementation; // Forward-declared, declared in the definition of MyClass.
        Implementation* _impl;
    };

This provides multiple advantages:

 - **Implementation dependencies don't pollute class declaration**: The class declaration is implementation-agnostic, so implementation dependencies are completely private.
 - **Reduce compile times**: Since the implementation details and dependencies are out of the class declaration, an implementation change doesn't involve full dependents compilation. As long as the interface remains untouched, clients only have to re-link against the new class version, instead of rebuilding their app.

## C++11 opaque pointer

As with C++11 one may be attempted to write *memory-safe* PIMPLs by putting the implementation pointers in smart pointers (usually `std::unique_ptr`):

    class MyClass
    {
        ... // Interface
    private:
        class Implementation; // Forward-declared, declared in the definition of MyClass.
        std::unique_ptr<Implementation> _impl;
    };

Sadly is not that simple since standard smart pointers impose some requirements to the type being wrapped:

*`I` means an incomplete type is enough, `C` means a complete type is needed*

    Complete type requirements for unique_ptr and shared_ptr
    
                                unique_ptr       shared_ptr
    +------------------------+---------------+---------------+
    |          P()           |      I        |      I        |
    |  default constructor   |               |               |
    +------------------------+---------------+---------------+
    |      P(const P&)       |     N/A       |      I        |
    |    copy constructor    |               |               |
    +------------------------+---------------+---------------+
    |         P(P&&)         |      I        |      I        |
    |    move constructor    |               |               |
    +------------------------+---------------+---------------+
    |         ~P()           |      C        |      I        |
    |       destructor       |               |               |
    +------------------------+---------------+---------------+
    |         P(A*)          |      I        |      C        |
    +------------------------+---------------+---------------+
    |  operator=(const P&)   |     N/A       |      I        |
    |    copy assignment     |               |               |
    +------------------------+---------------+---------------+
    |    operator=(P&&)      |      C        |      I        |
    |    move assignment     |               |               |
    +------------------------+---------------+---------------+
    |        reset()         |      C        |      I        |
    +------------------------+---------------+---------------+
    |       reset(A*)        |      C        |      C        |
    +------------------------+---------------+---------------+
    
*From ["Is std::unique_ptr<T> required to know the full definition of T?"](http://stackoverflow.com/questions/6012157/is-stdunique-ptrt-required-to-know-the-full-definition-of-t)*
    
As you can see, `std::unique_ptr` requires a complete type to define its destructor. The reasoning behind this is that the dtor calls a deleter that at the end calls `delete`, which needs a complete type to call that type dtor prior to deallocation.

However there's a workaround to make the code above work: A class destructor definition is illformed as long as any of its memebers destructor's is illformed. So when the compiler tries to define a class destructor all members dtors should be available. The code above fails to compile not because `Implementation` doesn't have a destructor, but **because the compiler cannot see it when trying to define the class default constructor from the translation unit including the declaration of the class.** 

The workaround consists in defining the class dtor in the definition (`.cpp`) file instead of the declaration, regardless it's `= default` or not. This way the compiler will try to define the class dtor in a translation unit **where the `Implementation` type, and its dtor, are defined and available**: 

`myclass.h`

    class MyClass
    {
        ... // Interface
        
        ~MyClass(); // define in .cpp
    private:
        class Implementation; // Forward-declared, declared in the definition of MyClass.
        std::unique_ptr<Implementation> _impl;
    };
    
`myclass.cpp`

    #include "myclass.h"
    
    // Declare and define implementation
    class MyClass::Implementation
    {
        ...
    };
    
    // Define default class dtor
    MyClass::~MyClass() = default;
