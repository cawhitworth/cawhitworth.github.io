---
layout: post
draft: true
title: ConditionalWeakTable and dynamic properties in .NET 4+
---

In version 4.0 of the .NET Framework, Microsoft added a new class to the
`System.Runtime.CompilerServices` namespace: `ConditionalWeakTable<TKey,
TValue>`. On the surface, this looks like a straightforward dictionary class -
it allows you to look up instances of `TValue` by instances of `TKey`.  But
we've already got `Dictionary<TKey, TValue>`, so what's special about this new
type?

## Static and Dynamic languages

Before we answer that, it's worth taking a brief detour into static vs dynamic
languages. In a static language like C#, when you define a class, its
properties and fields (and methods, excepting extension methods) are all fixed
at compile-time; accessing a field or property that does not exist on an object
is a compile error.

(aside: I'm aware that if you gather `n` developers in a room, there will be
`n+1` opinions about what constitutes a 'static' versus a 'dynamic' lanaguage;
I'm using the words 'static' and 'dynamic' as a convenience here, not arguing
for a definitive interpretation)

However, in a language like Python or Ruby, objects are dynamic at runtime –
that is, classes are not fixed at compile-time, and objects can have members
added and removed at runtime. Below is an example in Python:

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

There are a number of situations where it might be useful to have this kind of
behaviour - the ability to add properties to objects at runtime. For example,
when building dynamic data structures from a self-describing data format like
XML or JSON. Or, when building support for a language like Python or Ruby on
top of the CLR. More concretely, attached properties in WPF require something
very similar in order that they can be attached to an arbitrary object.

## Dynamic behaviour in .NET

.NET added support for `dynamic` in v4.0, and with it, the `ExpandoObject`.
This is an object that can be treated like our Python `MyClass` objects above:
it's an empty object that can have arbitrary properties assigned to and queried
at runtime; it can even be treated like a dictionary, just like in Python:

{% highlight csharp %}
dynamic a = new ExpandoObject();
a.Field = "Hello";
bool hasKey = ((IDictionary<string, object>) a).Any(kvp => kvp.Key == "Field"); // True
{% endhighlight %}

The downside of `ExpandoObject` is that it is sealed: it's not possible to
inherit from it to create a class that has some compile-time defined behaviour
that can also have arbitrary data attached to it at run-time. This is fine if
you're building a dynamic data structure (when parsing XML, for example), but
isn't so useful if you want to attach data to an existing class, as you might
when implementing something like WPF attached properties.

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
the `Dictionary` - and, in this case, the lifetime of the `MyClass` we
originally assigned to `b` is now determined by the lifetime of
`DynamicProperties.PropertyDict`, unless we remember to manually remove it
once we're finished with `b`.

This is obviously not ideal: if this was, say, a WPF control or dialog with an
attached property, we would leak both the control or dialog and the attached
dynamic object every time we used this pattern unless we also remember to
manually remove it from the `Dictionary`. And this is where
`ConditionalWeakTable` comes in.

## ConditionalWeakTable

`ConditionalWeakTable<TKey, TValue>` solves the problem of `Dictionary`
highlighted above by _not_ holding a reference to the `TKey` objects; rather,
it holds a table of special key-value pairs. These key-value pairs, called
`DependentHandle`s internally, hold a strong reference from the key to the
value, but do _not_ hold a strong reference to the key - the key is not kept
alive by the `ConditionalWeakTable`, unlike a `Dictionary`.

The upshot of this is that the value is only held alive by they key, and when
the key is garbage collected, the value is too (assuming nothing else holds a
reference to it). So, given the following class definitions:

{% highlight csharp %}

class MyClass
{
    ~MyClass() { Console.WriteLine("{0} finalized", GetType()); }
}

class MyClass2 : MyClass { }

{% endhighlight %}

and the following setup code:

{% highlight csharp %}

var conditionalWeakTable = new ConditionalWeakTable<MyClass, MyClass2>();

var first = new MyClass();
var second = new MyClass2();

conditionalWeakTable.Add(first, second);

{% endhighlight %}

If we set `second` to null and do a `GC.Collect()`, the ConditionalWeakTable
will keep the `MyClass2` instance alive, as we'd expect from a normal
dictionary:

{% highlight csharp %}
// Will print nothing to the console - the MyClass2 instance is still
// held alive through the ConditionalWeakTable
second = null; GC.Collect(); Console.ReadLine();
{% endhighlight %}

However, if we now set `first` to null and then `GC.Collect()`, this will mean
that the MyClass2 instance is no longer being held alive, we will see both the
`MyClass2` and the `MyClass` instance collected and finalized:

{% highlight csharp %}
// Will print:
// MyClass2 Finalized
// MyClass Finalized
first = null; GC.Collect(); Console.ReadLine();
{% endhighlight %}

You might be wondering how this is done - surely if the ConditionalWeakTable
somehow 'knows about' the keys and values, it must have a reference to them?
The answer is that it cheats - the internal `DependentHandle` class is an
implementation of a structure known as an
[_ephemeron_](https://en.wikipedia.org/wiki/Ephemeron) - a structure designed
to solve exactly this problem in garbage-collected systems. It is implemented
at the CLR level to allow precisely this kind of behaviour, which would
otherwise be impossible - the [source code](http://referencesource.microsoft.com/#mscorlib/system/runtime/compilerservices/ConditionalWeakTable.cs)
for `ConditionalWeakTable` and `DependentHandle` are available if you'd like
to dig into this further.

## Implications for profiling

Because `ConditionalWeakTable`s depend on support from the CLR, they also
require special-case support from profiling tools. Without this support,
objects kept alive only through `ConditionalWeakTable`s would show as having no
path to the Garbage Collection root - for the example above, after `second` has
been set to null, we would see the following in instance retention graph in
ANTS Memory Profiler:

![No GC root for MyClass2]({{ site.url }}/assets/no_gc_root.png)

This is obviously not true - but the normal reference data we get back from
the .NET profiling interfaces don't include data from `ConditionalWeakTable`s.
Fortunately, Microsoft have provided a profiling interface ([`ICorProfilerCallback5`](https://msdn.microsoft.com/en-us/library/jj160349(v=vs.110).aspx))
to allow us to discover references held through `ConditionalWeakTable`s, and
so as of the latest version of ANTS Memory Profiler, this data is combined with
the normal reference data, and so what we see is:

![ConditionalWeakTable reference shown by dashed light-blue arrow]({{ site.url }}/assets/with_conditional_weak_ref.png)

The dashed light-blue arrow from `MyClass` to `MyClass2` indicates that this
reference is held in a `ConditionalWeakTable` rather than being a 'normal'
strong reference.

## Conclusion

`ConditionalWeakTable` lives in the `System.Runtime.CompilerServices`
namespace, and the [MSDN documentation](https://msdn.microsoft.com/en-us/library/dd287757(v=vs.110).aspx)
talks about it being useful to compiler writers implementing dynamic
languages on the .NET runtime. However, as discussed above they are also useful
in a number of other circumstances - and indeed are used in a number of places
in the .NET Framework, including in WPF for attached properties and weak events
(forgetting to detach events being another common source of memory leaks).
Like `WeakReference`, they're not a panacaea for all memory problems in
.NET but, used carefully and in specific circumstances, they can be a useful
addition to a developer's toolkit.

You can download a trial of ANTS Memory Profiler from the [RedGate website](http://www.red-gate.com/products/dotnet-development/ants-memory-profiler/).
