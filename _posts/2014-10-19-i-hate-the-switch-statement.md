---
layout: post
title: I Hate the Switch Statement
modified:
categories: C++
excerpt:
tags: [C++]
image:
  feature:
date: 2014-10-19T01:46:34+02:00
comments: true
---

C++ inherits many of its features from the C programming language, some of them fit well with modern programming, and some others do not.     
One cathegory of such features is the syntax for flow control, where the C language stands above the rest of the languages thanks to some constructions able to do almost any trick you can ever imagine. 

Look at the `for` loop: Is not a simple three arguments construction. You can do whatever you like.

{% highlight C %}
for(;;;)
{
    //I can't scape from the loop!!!
}

for(;;;); //The most useful statement in the history of CS

int i = -1;
for(; ++i , (i < 10) ;)
{
    //cool...
}
{% endhighlight %}

In a time when the C syntax is reaching the top of its power, see the C++11 trailling return types or the C11 generic macros, there are some old and ugly constructions that still remain here, with their lack of expressiveness and a long list of potential issues related to their usage.

One of such constructions is the `switch` statement.

## Switch statement anatomy

The `switch` statement from C is one of the pilars of control flow in C and C++. It executes conditionally a block of code corresponding to an specific value, given 
a set of `(value,code)` pairs, also known as *"cases"*:

{% highlight C %}
int c = read_from_somewhere();

switch(c)
{
    case 1:

    case 2:

    case 3:

    default:

}
{% endhighlight %}

You pass a value as `switch` parameter, that value is evaluated and then the program jumps to the proper case tag. That low-level way of explaining this, resembling assembly, is the perfect choice here since **thats exactly the behavior of the `switch`**, besides how its actually implemented (Jump tables, etc).

So what's the problem with the *eval and jump* approach? Simple: **Think about the C `switch`  as a collection of `goto`s**.  
Each case code is not a block unless you specify it using `{}`. This produces some problems when defining variables in a `switch`, and the programmer should be careful and sure about what is doing.   
Also **the program does not *exit* the case code after executing its last instruction**, we should exit manually via `break`. Remember, there is really no case block of code, so there is no block to exit from. 

The *eval and jump* C `switch` behaves like just a couple of `goto`s to specific labels, `case`s, with all the problems explained above. But C programmers have been ignoring such problems since the beginning of the language because that simple approach allows us to perform **amazing and *"Oooh, I'm the coolest programmer!"* crazy hacks**.  
The best example of this fever, the [Duff's device](http://www.drdobbs.com/a-reusable-duff-device/184406208)

{% highlight C %}
int n = (len + 8 - 1) / 8;
switch (len % 8) {
case 0: do { HAL_IO_PORT = *pSource++;
case 7: HAL_IO_PORT = *pSource++;
case 6: HAL_IO_PORT = *pSource++;
case 5: HAL_IO_PORT = *pSource++;
case 4: HAL_IO_PORT = *pSource++;
case 3: HAL_IO_PORT = *pSource++;
case 2: HAL_IO_PORT = *pSource++;
case 1: HAL_IO_PORT = *pSource++;
} while (--n > 0);
}
{% endhighlight %}
*Thats a loop unroller. Ugh... We all have to thank modern compiler research.*

My intention is not to critize this kind of hacks, they were reasonable in the context they were designed for. In the case of Duff's device, there were no compiler magic like what we have available these days. They only had a stupid compiler and their need to squeeze each CPU cicle.

What I'm trying to show here is that, even with all that potential sorcery, **the C `switch` statement is far from being really useful and comfortable for the average programmer and the most common programming tasks**.

Which are such common tasks?

### Range-dependent code execution

Its a common pattern to execute different code if a value (Variable, function return, whatever) belongs to one specific range of values or other. This simple task is not easy to perform with a `switch`, even if the switch seems to be the most natural candidate for this.  
The `select case` of Visual Basic, its `switch` equivalent, allowed such construction. But we as C++ programmers, should do some workarounds like this:

{% highlight cpp %}
int f(int n)
{
    if( n >= 0 && n < 10)
        return 1;
    else if(n >= 10 && n < 20)
        return 2;
    else if(...)
        ...
} 
{% endhighlight %}

A chain of `if`s. I have nothing against the `if`, but consider what happens the day you have to change that `[n, n+10)` ranges into `[n, n+30)`. That syntax is not the natural way to express that, even if us as C/C++ programmers know perfectly that pattern, and we are able to get its meaning directly. 

### Simple value --> code mapping

This task, this simple task, is exactly what the `switch` statement is supposed to do. But the community knows the issues of the `switch` explained above, so using a map (Or even a lookup table) instead of a `switch` have become a good practice in our guidelines:

{% highlight cpp %}
void dispatch_event(const Event& e)
{
    std::unordered_map<EventType,Action> mapping;

    mapping[EventType::MouseClick] = [](const EventArgs& args){ std::cout << "Mouse click!" << std::endl; };
    mapping[EventType::MouseMove]  = [](const EventArgs& args){ std::cout << "Mouse move!" << std::endl; };
    mapping[EventType::MouseWheel] = [](const EventArgs& args){ std::cout << "Mouse wheel!" << std::endl; };

    mapping[e.type()](e.args());   
}
{% endhighlight %}


## To switch or not to switch

My question is: **Is there any alternative for the `switch` statement?**

I'm a big fan of the *"Use algorithms instead of raw loops"* motto, it increases code readability and improves 
maintainability. But my attempts to find an equivalent alternative for the `switch` were unsuccessful.  
The patterns shown above work, but they are still to verbose compared to what we are really trying to say with that bunch of characters.

What are the alternatives? Maybe a pattern matching system for C++? Maybe some cool DSL? I'm not sure, but what I know is that we really need a replacement and/or improvement for the `switch` statement.
