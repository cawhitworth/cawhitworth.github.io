---
layout: post
draft: true
title: ConditionalWeakTable and dynamic properties in .NET 4+
---

In version 4.0 of the .NET Framework, Microsoft added a new class to the 
`System.Runtime.CompilerServices` namespace: `ConditionalWeakTable<TKey,
TValue>`.  On the surface, this looks like a straightforward dictionary-style
class - it allows you to look up instances of `TValue` by instances of `TKey`.
But we've already got `Dictionary<TKey, TValue>`, so what's special about this
new type?

## Static and Dynamic languages

Before we answer that, it's worth taking a brief detour into static vs dynamic
languages. In a language like C#, when you define a class, its properties and
fields (and methods, excepting extension methods) are all fixed at compile-time;
accessing a field or property that does not exist on an object is a compile
error.

However, in a language like Python or Ruby, objects are dynamic at runtime –
that is, classes are not fixed at compile-time, and can have members added and
removed at runtime. Below is an example in Python:

{% highlight python %}
# Declare a class with no fields or methods
class MyClass(object): pass

# Construct an instance of it
myInstance = MyClass()

# Create a field called myValue on the instance and assign something to it
myInstance.myValue = AnotherClass()
{% endhighlight %}

Here, we declared a class with no members (or 'attributes' in Python
terminology), created an instance of it, and then assigned something to the
myValue attribute. This attribute did not have to be forward-declared anywhere:
just accessing it is enough to bring it into existence. This works because in
Python objects are implemented as dictionaries – when you access a field on the
object, it's essentially syntactic sugar for accessing a special dictionary:

{% highlight python %}
# Declare and create an empty class/object
class MyClass(object): pass
myInstance = MyClass()

print(myInstance.__dict__)          # {}

myInstance.value = 30
print(myInstance.__dict__)          # { 'value' : 30 }

myInstance.__dict__['value'] = 20
print(myInstance.value)             # 20
{% endhighlight %}

## Probably a subheading goes here?

There are a number of situations where it might useful to have this kind of
behaviour - the ability to add properties to objects at runtime - in .NET. For
example, when implementing a language like Python or Ruby on top of the CLR,
or when building dynamic data structures from a self-describing data format
like XML or JSON. WPF requires something similar for attached properties.

One way of doing this is to use the support for dynamic objects in .NET 4+, as
below:

{% highlight csharp %)
dynamic a = new ExpandoObject();
a.Field = "Hello";
bool hasKey = ((IDictionary<string, object>) a).Any(kvp => kvp.Key == "Field"); // True
{% endhighlight %)

However, this requires you to use an `ExpandoObject`, which is sealed: you
can't extend it to provide your own concrete functionality and then attach
arbitrary runtime data to it.
