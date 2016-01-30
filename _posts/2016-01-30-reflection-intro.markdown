---
layout: post
title: Reflection (Intro)
modified:
categories: 
excerpt:
tags: []
image:
  feature:
date: 2016-01-30T01:05:23+01:00
---

For three years I have been giving some C++ courses in my university as part
of my spare time (Aka "the time I should have been studing"...), most of 
them focused in getting C++ right from the beginning. This year
I decided to split the course in two: One for beginners (The usual course I
give) and an advanced C++ course focused at people in the game programming 
master degree.

For them "advanced C++" means I can write whatever crazy code I like so they learn how
to write simple and maintainable modern C++ while at the same time learn some
techniques that can be useful when writing game engines or whatever
thing they end up working with. *'Whatever' is the key here, tell me six
months ago I will be working writing network comunication protocols...*
Is not a course about "How to write C++" but, "You already know C++? Let's see...".

For me that means I can use the [course repo](https://github.com/GueimUCM/siplasplas) as a sandbox.
Currently the code there covers topics from allocators to variant, but for
me the most amusing to write, and the one I want to talk here, is reflection.

## Reflection

From Wikipedia:

> reflection is the ability of a computer program to examine and modify 
> its own structure and behavior (specifically the values, meta-data, properties and functions) at runtime

Reflection is the kind of black magic one loves when playing with dynamically 
typed languages, such as Python or C#, that thing that makes possible to do 
some type introspection like "given this object, give me a list with all its 
methods":

{% highlight python %}
[method for method in dir(object) if callable(getattr(object, method))]
{% endhighlight %}

*Yes, I [just googled](http://stackoverflow.com/questions/34439/finding-what-methods-an-object-has) 'Python get list of methods',it's easier than
getting an example out of my head...*

A simple-but-useful example of reflection is automatic serialization:
Since you can get the set of fields a class has, writing a serialization tool is
in its simple form more or less a foreach loop writing or reading that fields.
By automatic I mean that you don't have to care about class fields manually: We
have serialization tools for C++, but these usually cover simple types only
and leave class serialization up to the user.

### Static vs dynamic reflection

We all agree that the set of fields and member functions is known at compile 
time (We are talking about C++ after all, not about Javascript...), so giving
an standard interface to that information [should be possible](https://www.google.es/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&ved=0ahUKEwi2tcmTutDKAhXM7hoKHejYAtgQFggpMAE&url=https%3A%2F%2Fisocpp.org%2Ffiles%2Fpapers%2Fn3996.pdf&usg=AFQjCNEuhwqFC70Cmi8d5ZCp3rbxSdve1w&sig2=ygIJuD4dlOxCfRtQsljOQg). But, in my opinion,
that's not very useful if playing with that info means having a codebase that
looks more like HTML than C++ (Try [Hana](https://github.com/boostorg/hana), it may ease the pain).

So for most of the cases, I don't like compile-time (static) reflection, or to be more
specific, I don't like reflection/type-introspection tools that cannot be
queried using simple syntax.

Instead, what I have implemented is a reflection system similar to Qt's metatype
system (Actually inspired in): Each class has a set of metadata associated that
can be queried at runtime using C++.

{% highlight cpp %}

class MyClass
{
public:
    void f() const;
    void g() const;
};

for(const auto& keyValue : MyClass::reflection().functions())
{
    const auto& name = keyValue.first;
    const auto& function = keyValue.second;

    MyClass instance;
    const auto& bindedFunction = function(instance);

    // Calls member function on instance
    bindedFunction();
}

{% endhighlight %}

## Reflection step by step

The idea of both the code of the course and this post(s) is to show **how
C++ reflection works**, so if you take Qt or Unreal Engine you have a brief
idea of what's going on under the hood. Is not, and is not intended to be, a 
reference on how to implement relfection in C++, it's just the way I did it.

In order to achieve the code above we should realize that:

 - **C++ is a statically typed language, but with runtime reflection we are
 introducing runtime typing**: Since functions and filds are queried and accessed
 at runtime, the type of the data depends on runtime values. We need a way to do runtime typing (At least limited) in C++.
 
 - **We have to collect class information in order to store the meta-information**: As I said
 above, I don't want to do this manually, so this would involve some kind of 
 parsing.

I would try to write a post for each feature we need to reach the goals above, 
more or less:

 1. **TypeInfo**: Storing C++ type information.
 2. **MetaType**: Type-erased type information. It's the main building block of the
    reflection system, it knows about a type `T`, how to construct `T`s, etc.
 3. **MetaObject**: RAII objects with runtime type.
 4. **(Meta)Field**: Represents a field of a class, giving read/write access to it.
 5. **(Meta)Function**: A memeber function of a class, you can dynamically invoke it.
 6. **MetaClassData**: Collects meta-information of a class, giving `Field`s and `Function`s.
 7. **Automatically collecting class information**: libclang! (Or static reflection in C++17, I hope)
 8. **CMake integration**

*I'm missing something...?*

That's all! I hope you would enjoy it. Here's a little spoiler: 
Project bootstrapping and the reflection system running

[![asciicast](https://asciinema.org/a/c13nlez3fhd86xdkicw7l6y8q.png)](https://asciinema.org/a/c13nlez3fhd86xdkicw7l6y8q)

*The parser looks slow? Don't try to develop with an Atom netbook with 1GB of RAM...*
