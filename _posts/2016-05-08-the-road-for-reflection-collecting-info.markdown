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

*If your were following this series, you might have noticed I've changed
the name of the class from `cpp::MetaObject` to
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

    // Specialization for non-const member functions
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

    // Specialization for const member functions ...
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


## Reflection metadata

First of all, before even considering writing an automation tool, we need
to decide what data we will need from our C++ classes, which of course
depends on your needs. If you want object serialization, access to class
fields only is fine:

``` json
{
  "fields": {
    "field": {
      "type": "int",
      "value": "42"
    },
    "objectOfInnerClass": {
      "fields": {
        "innerMember": {
          "type": "int",
          "value": "0"
        }
      },
      "type": "MyClass::InnerClass"
    },
    "otherField": {
      "type": "bool",
      "value": "0"
    },
    "strField": {
      "type": "std::__cxx11::basic_string<char>",
      "value": "hello"
    }
  },
  "type": "MyClass"
}
```

*That's JSON output of a `MyClass` instance, an example from a tiny
serialization lib built on top of the dynamic reflection engine. Check the
repo for `siplasplas/serialization` library if you like.*

Another use case is automatic registration of bindings, where you may need
the set of public methods of a class:

``` cpp
#include <chaiscript/chaiscript.hpp>
#include <chaiscript/chaiscript_stdlib.hpp>
#include <siplasplas/utility/fusion.hpp>
#include "myclass.hpp"

int main()
{
    chaiscript::ChaiScript chai{chaiscript::Std_Lib::library()};

    cpp::foreach<cpp::static_reflection::Class<MyClass>::Methods>([&](auto method)
    {
        using Method = cpp::meta::type_t<decltype(method)>;

        chai.add(chaiscript::fun(Method::get()), Method::spelling());
    });
}
```

*Again, a real example you can check in the repo. Sane users would use
Boost.Hana instead of my `cpp::foreach()` template.*

To keep things simple, suppose you want reflection of enums (Both
constants values and their names) and class methods.

*"To keep things simple". If this sounds like I'm an expert on reflection,
or even like if I know what I'm talking about, don't be confused: The
former is just the first thing I played with some months ago (i.e. the one
I'm most confident with) and the latter it's just what I've committed
less than 72 hours ago. Great post, isn't?*

## Metadata representation

So far we've seen what metadata we need, but not how how to record and
access it.

Since most (all) of this information is **static**, the simplest way
I found to access metadata is through templates:

``` cpp
namespace cpp::static_reflection
{

template<typename ClassType>
class Class
{
   ...
};

}
```

The idea is simple: To access metadata of a class type `T`, use the
`static_reflection::Class<T>` class. Any entity we reflect will be
accessible using this API convention. Following with methods:

``` cpp
namespace cpp::static_reflection
{

template<typename MethodType, MethodType method>
class Method
{
    ...
};

}
```

where `Method` instances have the metadata for the specified `method`:

``` cpp
class MyClass
{
public:
    void f();
};

using f_method = cpp::static_reflection::Method<decltype(&MyClass::f), &MyClass::f>;

std::cout << f_method::spelling(); // Gives "f"
f_method::invoke(MyClass()); // invokes "f" method on temporary MyClass instance
```

*Why two parameters, one for the type and another for the pointer itself?
Well, there's no generic way to pass a function pointer as template
parameter (any non-type template parameter actually). So the only way is
to pass the pointer type first, then the pointer. Check
`std::integral_constant` template, it works under the same principle.*


