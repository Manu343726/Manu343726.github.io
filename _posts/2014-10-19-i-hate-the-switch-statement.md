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
published: false
---

C++ inherits many of its features from the C programming language, some of them fits well with modern programming, and some others do not. 
One cathegory of such features is the syntax for flow control, a topic where C shines thanks to their constructions cappable of doing almost
anything via almost any imaginable hack. See the `for` loop. Is not a simple three arguments construction. You can do whatever you like.

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

In a time when the C syntax is reaching the top of its power, see the C++11 trailling return types or the C11 generic macros, there are some old
and ugly constructions that still remains here, with its lack of expresiveness and a long list of potential issues related to their usage.

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

You pass as `switch` parameter

## To switch or not to switch