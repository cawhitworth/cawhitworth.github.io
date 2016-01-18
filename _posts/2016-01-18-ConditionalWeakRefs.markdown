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
# Declare an empty class and construct one
class MyClass(object): pass
myInstance = MyClass()

# Create a field called "myValue" on the instance
# and assign something to it
myInstance.myValue = AnotherClass()
{% endhighlight %}

Here, we declared a class with no members (or 'attributes' in Python
terminology), created an instance of it, and then assigned something to the
myValue attribute. This attribute did not have to be forward-declared anywhere:
just accessing it is enough to bring it into existence. This works because
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

There are a number of situations where it might useful to have this kind of
behaviour - the ability to add properties to objects at runtime. For example,
when building dynamic data structures from a self-describing data format like
XML or JSON. Or, when building support for a language like Python or Ruby on
top of the CLR. More concretely, attached properties in WPF require something
very similar in order that they can be attached to an arbitrary object.

## Dynamic behaviour in .NET

One way of doing this is to use the support for dynamic objects in .NET 4+, as
below:

{% highlight csharp %}
dynamic a = new ExpandoObject();
a.Field = "Hello";
bool hasKey = ((IDictionary<string, object>) a).Any(kvp => kvp.Key == "Field"); // True
{% endhighlight %}

However, this requires you to use an `ExpandoObject`, which is sealed: you
can't extend it to provide your own concrete functionality and then attach
arbitrary runtime data to it. This is probably fine if you're building a
dynamic data structure, but isn't so useful if you want to attach data to
an existing class.

One way to achieve this might be to use a static dictionary - here we can
conceptually 'attach' an `ExpandoObject` to any other object to give it a set
of runtime properties:

{% highlight csharp %}
class Program
{
    static IDictionary<object, dynamic> DynamicProperties = new Dictionary<object, dynamic>();

    static void Main(string[] args)
    {
        var b = new MyClass();
        DynamicProperties[b] = new ExpandoObject();
        DynamicProperties[b].Field = "Hello";
    }
}
{% endhighlight %}

We could be even smarter about this and use an extension method:

{% highlight csharp %}
static class DynamicProperties
{
    static readonly IDictionary<object, dynamic> PropertyDict = new Dictionary<object, dynamic>();

    public static dynamic Properties(this object key)
    {
        dynamic expando;
        if (!PropertyDict.TryGetValue(key, out expando))
        {
            expando = new ExpandoObject();
            PropertyDict[key] = expando;
        }
        return expando;
    }
}

class MyClass { }

class Program
{
    static void Main(string[] args)
    {
        var b = new MyClass();

        b.Properties().Field = "Hello";
    }
}
{% endhighlight %}

The problem with this approach, though, is that as soon as an object is added
to a `Dictionary`, as either a key or a value, that `Dictionary` takes a
strong reference to it, and so the object's lifetime is now tied to that of
the `Dictionary` - and, in this case, as it is static, this will be for the
rest of the runtime of the program. This obviously isn't ideal - and this
is where `ConditionalWeakTable` comes in.

# ConditionalWeakTable

`ConditionalWeakTable<TKey, TValue>` solves the problem of `Dictionary`
highlighted above by _not_ holding a reference to the `TKey` objects; rather,
it holds a table of key-value pairs _without_ holding a reference to the key
objects. It also creates a strong reference from the key to the value so when
the key is garbage-collected, the value may be as well (assuming the value has
no other references holding it in memory).



 rather,
it makes use of an internal CLR implementation of an
[_ephemeron_](https://en.wikipedia.org/wiki/Ephemeron) - a structure specifically
designed to solve this problem; in the .NET CLR, these are called
`DependentHandle`s. 

