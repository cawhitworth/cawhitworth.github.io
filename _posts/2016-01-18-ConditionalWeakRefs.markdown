---
layout: post
draft: true
title: ConditionalWeakTable and dynamic properties in .NET 4+
---

# ConditionalWeakTable and Dynamic Properties in .NET 4

In version 4.0 of the .NET Framework, Microsoft added a new class to the
`System.Runtime.CompilerServices` namespace: `ConditionalWeakTable<TKey,
TValue>`.  On the surface, this looks like a straightforward dictionary-style
class - it allows you to look up instances of `TValue` by instances of `TKey`.
But we've already got `Dictionary<TKey, TValue>`, so what's special about this
new type?

Before we answer that, it's worth taking a brief detour into static vs dynamic
languages.  In a language like Python or Ruby, however, objects are dynamic at
runtime – they can have members added and removed at runtime. For example, see
the following Python code:

{% highlight python %}
# Declare a class with no fields or methods
class MyClass(object): pass

# Construct an instance of it
myInstance = MyClass()

# Create a field called myValue on the instance and assign something to it
myInstance.myValue = AnotherClass()
{% endhighlight %}

Here, we declared a class with no members (or 'attributes' in Python
terminology), create an instance of it, and then assign something to the
non-existent myValue field on the class. This works because in Python objects
are implemented as dictionaries – when you access a field on the object, that's
just syntactic sugar for accessing a special dictionary:

````
# Declare and create an empty class/object
class MyClass(object): pass
myInstance = MyClass()

# This will print an empty dict, "{}"
print(myInstance.__dict__) 

myInstance.value = 30

# This will print "{ 'value': 30 }"
print(myInstance.__dict__)
````

In a language like C#, when you define a class, its properties and fields (and
methods, excepting extension methods) are all fixed at compile-time; accessing
a field or property that does not exist on an object is a compile error. In
fact, the .NET requires that a module has a complete formal description of
itself - all type information, all structural information - in its metadata.

Implementing a language like this on top of the .NET CLR causes a problem: the
CLR expects to have metadata available for all objects that detail all the
members of an object; the garbage collector uses this metadata

ConditionalWeakTable was added to make building dynamic languages on top of
.NET easier

