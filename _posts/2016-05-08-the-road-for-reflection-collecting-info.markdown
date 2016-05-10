---
layout: post
title: Collecting information
modified:
categories: C++
excerpt:
tags: [C++]
image:
  feature:
date: 2016-05-08T14:51:34+02:00
comments: true
---

[Last time]() we saw how to implement a `Boost.Any` kind of `Object`,
a class that type erases its value so we can assign to it any value we
want:

``` cpp
cpp::dynamic_reflection::Object any = 1; // type int, value 1
any = "hello, world"s; // type std::string, value "hello, world"
```

We've also seen a couple of tools that help to manipulate objects, such as
invoking functions:

``` cpp
void foo(const std::string& str, bool boolean);

std::vector<Object> params = pack_to_vector("hello, world"s, 1);
vector_call(foo, params); // invokes foo("hello, world", 1);
```

*If your were following this series, you may have noticed I've changed the
name of the class from `cpp::MetaObject` to
`cpp::dynamic_reflection::Object`. During this weeks I have been doing
a full refactoring of the reflection library, splitting it into two
independent sub-libraries: One for static reflection, and one for dynamic.
More on this bellow.*

With the tools above you're able to write the basics for a dynamic
reflection runtime. For example, to invoke a method of a class
dynamically, just register its pointer and later call the method using the
object manipulation tools:

``` cpp
class Method
{
public:
    template<typename MethodType>
    Method(MethodType method) :
        _method{ new MethodPointer<MethodType>{method} }
    {}

    template<typename Class, typename... Args>
    Object operator()(Class& object, Args&&... args)
    {
        return _method->invoke(Object{object},
        pack_to_vector(std::forward<Args>(args)...));
    }

private:
    class MethodInterface
    {
    public:
        Object invoke(Object&& object, const std::vector<Object>& args) = 0;
    };

    std::unique_ptr<MethodInterface> _method;

    template<typename MethodType>
    class MethodPointer;

    template<typename R, typename C, typename... Args>
    class MethodPointer<R (Class::*)(Args...)> : public MethodInterface
    {
    public:
        MethodPointer(R(Class::* pointer)(Args...)) :
            _pointer{pointer}
        {}

        Object invoke(Object&& object, const std::vector<Object>& args)
        override
        {
            // invoke method ptr with object and args
            return vector_call(_pointer, object.get<C>(), args);
        }

    private:
        R (Class::*_pointer)(Args...);
    };
};
```

*Don't worry if you don't get the code above, we will talk about it in following posts*.

``` cpp

class MyClass
{
public:
    int f(int a);
    std::string g(const std::vector<std::string>& strings);
};

std::unordered_map<std::string, Method> myClassMethods = {
    {"f", &MyClass::f},
    {"g", &MyClass::g}
};

// Dynamically invoke f method:

MyClass myObject;
int result = myClassMethods["f"](myObject, 42);
```

Even if you find the code above to be black magic or something (As I said,
don't worry, I will explain dynamic reflection runtime in folllowing
posts) there's something you can notice even not completely understanding
what `Method` does: **Class registration is a manual, repetitive, boring,
and error prone process**.

In real world codebases such a manual process is not an option: We manage
hundreds of classes, with many methods each (*I hope you don't have
a class with more than 15 methods as part of its public API, that's no
longer a class but a hard to maintain monster. Ptsss By Tech serial port
`Controller` class...*). Also we are not just insterested in methods, we
could want programatic access to class fields, constructors, base classes,
etc. There's a lot of information to collect and register.

In this post I will introduce you to the **DRLParser** (Diego Rodriguez
Losada Parser, ah sorry, I meant "Dynamic Reflection Library Parser"),
a libclang based python tool I wrote to help with this pain.






































































































































































































































































































































































































































































































































































