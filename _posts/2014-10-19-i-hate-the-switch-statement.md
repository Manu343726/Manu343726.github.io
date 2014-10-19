---
layout: post
title: I Hate the Switch Statement
modified: null
categories: C++
excerpt: null
tags: 
  - C++
  - DSL
image: 
  feature: null
date: {}
published: true
---

C++ inherits many of its features from the C programming language, some of them fits well with modern programming, and some others do not.   
One cathegory of such features is the syntax for flow control, a topic where C shines thanks to their constructions cappable of doing almost anything via almost any imaginable hack.   

See the `for` loop: Is not a simple three arguments construction. You can do whatever you like.

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

In a time when the C syntax is reaching the top of its power, see the C++11 trailling return types or the C11 generic macros, there are some old and ugly constructions that still remains here, with its lack of expresiveness and a long list of potential issues related to their usage.

One of such constructions is the `switch` statement.

## Switch statement anatomy

The `switch` statement from C is one of the pilars of control flow in C and C++. It executes conditionally a block of code corresponding to an specific value, given 
a set of `(value,code)` pairs, formely known as *"cases"*:

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

So what's the problem with the *eval and jump* approach? Simple: Think about the C `switch`  as a collection of `goto`s.  
**Each case code is not a block unless you specify it using `{}`**. This produces some problems when defining variables in a `switch`, and the programmer should be careful and sure about what is doing.   
Also **the program does not *exit* the case code after executing its last instruction**, we should exit manually via `break`. Remember, there is really no case block of code, so there is no block to exit from. 

The *eval and jump* C `switch` behaves like just a couple of `goto`s to specific labels, `case`s, with all the problems explained above. But C programmers have been ignoring such problems since the beggining of the language because that simple approach allows us to perform **amazing and *"Oooh, I'm the coolest programmer!"* crazy hacks**.  
The best example of this fever, the [Duff's device](http://en.wikipedia.org/wiki/Duff's_device)
























## To switch or not to switch