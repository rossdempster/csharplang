# Skeleton Classes

* [x] Proposed
* [ ] Prototype: [Not Started](https://github.com/PROTOTYPE_OWNER/roslyn/BRANCH_NAME)
* [ ] Implementation: [Not Started](https://github.com/dotnet/roslyn/BRANCH_NAME)
* [ ] Specification: [Not Started](pr/1)

## Summary
[summary]: #summary

A mechanism whereby nominated members of a class _or any of its derivations_ can be declared "bodyless" and implemented dynamically. Such members are redirected via standard compiler pattern-matching techniques to members of the hosting class. The resulting potential for implementing so-called "cross-cutting concerns" automatically (e.g. INotifyPropertyChanged, thread affinity, automatic persistence etc.) is considerable.

## Motivation
[motivation]: #motivation

This proposal is aimed at _framework authors_, specifically those producing frameworks where consumers declare their own model classes that interoperate with the framework. The most obvious example being Object-Relational Mapping (ORM) frameworks such as Entity Framework, where the framework consumer typically produces model classes particular to their own database schema. Another less obvious example is WPF's binding framework, where INotifyPropertyChanged becomes a perennial burden for anything touching the UI. 

While the proposal is aimed at framework authors, it is primarily _for the benefit of framework consumers_.

The two examples above are of particular note because they demonstrate the problem perfectly. _People do not like writing boilerplate code._ We like our POCOs. The developers of popular frameworks will go to great lengths to accomodate this.

* In the case of Entity Framework, proxy classes are generated and substituted on-the-fly. These proxies inject additional code for the purpose of change tracking and lazy loading. Generating proxies in this manner is a considerable undertaking for the framework author, yet deemed worthwhile for the full POCO experience. The alternative is to have the framework _consumer_ produce all the boilerplate for each property by hand, or attempt to automate it with tools.

* INotifyPropertyChanged is a similar pain point, spurring the popular **PropertyChanged.Fody** package. This tool rewrites your auto-implemented properties _after they've been built to MSIL_ and injects the necessary code for raising PropertyChanged. All to avoid that boilerplate!

Of course, this is exactly why auto-implemented properties exist in the first place. The issue of boilerplate code is well understood. These two examples are just an extension of that concept using a different implementation. But these techniques are advanced, difficult to experiment with, and not without drawbacks of their own. There is a considerable cost to make use of them.

If we recognise that there exists a _general case_, whereby a base class could dictate a custom implementation pattern for _any member_, the barrier of entry can be lowered considerably. Simple POCOs could become extremely powerful. The applications are endless:

* INotifyPropertyChanged
* Object Relational Mappers
* "Strong Wrappers* (i.e. wrapping an IDictionary)
* Thread Affinity/Enforcement
* Binding, Animation, Freezable (e.g. WPF)
* Immutable Objects (e.g. the "WithX" pattern)
* Automatic Persistence

Such "cross-cutting" requirements come up all the time.

A number of experimental ideas also come to mind that could benefit from such a capability, potentially fostering some framework innovation on the C# platform.

In summary: by making it easier for _framework authors_ to enhance our POCOs, we can simplify code _for everyone_.

## Detailed design
[design]: #detailed-design

The design concept behind this proposal is very simple.

Let's imagine the following class. We derive PropertyChangedBase and declare some properties using the `auto` keyword. This keyword instructs the compiler to implement these properties using pattern-matching. Such properties cannot have a body.

```csharp
public class MyModelClass : PropertyChangedBase
{
    public auto string Name { get; set; }
    public auto DateTimeOffset DateOfBirth { get; set; }
}
```

The **compiler expansion** of this class is as follows. There is very little work for the compiler to perform. The implementation of each `auto` property is provided via delegates that are initialized in the **static** class constructor. Static initialization is used for performance reasons.

```csharp
public delegate TValue Getter<THostType, TValue>(THostType host);
public delegate void Setter<THostType, TValue>(THostType host, TValue value);

public class MyModelClass : PropertyChangedBase
{
    private static Getter<MyModelClass, string> __get_Name = GetPropertyGetImplementation(propertyinfoof(Name));
    private static Setter<MyModelClass, string> __set_Name = GetPropertySetImplementation(propertyinfoof(Name));
    private static Getter<MyModelClass, DateTimeOffset> __get_DateOfBirth = GetPropertyGetImplementation(propertyinfoof(DateOfBirth));
    private static Setter<MyModelClass, DateTimeOffset> __set_DateOfBirth = GetPropertySetImplementation(propertyinfoof(DateOfBirth));

    public string Name
    {
        get => __get_Name(this);
        set => __set_Name(this, value);
    }

    public DateTimeOffset DateOfBirth
    {
        get => __get_DateOfBirth(this);
        set => __set_DateOfBirth(this, value);
    }
}
```

PropertyChangedBase is responsible for providing the implementation. The actual implementation is not shown here because it is irrelevant with respect to the design. The key understanding is that PropertyChangedBase can provide a pattern-based implementation for all properties marked with `auto`, considerably reducing the implementation burden for anything deriving PropertyChangedBase.

```csharp
public class PropertyChangedBase : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;

    protected static Getter<THost, TValue> GetPropertyGetImplementation<THost, TValue>(PropertyInfo propertyInfo)
    {
        // Put whatever you like here. The world is your oyster!
    }

    protected static Setter<THost, TValue> GetPropertySetImplementation<THost, TValue>(PropertyInfo propertyInfo)
    {
        // Put whatever you like here. The world is your oyster!
    }
}
```

The power of this concept becomes apparent with more advanced patterns. **The classes shown below are full implementations.** The `auto` keyword supports convention-based implementation based on custom logic (such as naming conventions, metadata attributes etc.). It is incredibly flexible.

```csharp
public class MyImmutableModel : ImmutableBase
{
    public auto string Name { get; }
    public auto DateTimeOffset DateOfBirth { get; }

    public auto MyImmutableModel WithName(string name);
    public auto MyImmutableModel WithDateOfBirth(DateTimeOffset dateOfBirth);
}
```

```csharp
public class MyDependencyObject : DependencyObject
{
    public static auto DependencyProperty NameProperty { get; }
    public static auto DependencyProperty DateOfBirthProperty { get; }

    public auto string Name { get; set; }
    public auto DateTimeOffset DateOfBirth { get; set; }
}
```

The compiler expansion of static members and methods is no different than that shown for properties.

## Drawbacks
[drawbacks]: #drawbacks

TBC. For discussion.

## Alternatives
[alternatives]: #alternatives

Alternative techniques are available, but are considerably less accessible.

* **Code Generation.** Boilerplate code can be generated using compile-time tools. Typically a domain-specific language or markup language is used to specify the structure of required classes. The overhead of setting up the tool-chain is not insignificant, and it is not particularly easy to deploy as part of a framework if your framework requires consumers to provide their own models. Declaring classes in markup outside of C# is not ideal.

* **Runtime Proxy Generation.** Advanced technique requiring runtime code generation via Expression Trees or Reflection Emit. Much friendlier to the consumers of your framework, but can still be a burden if external instantiation of proxies is required. The cost/benefit is questionable for something as simple as INotifyPropertyChanged. Is this technique even possible in environments requiring ahead-of-time compilation such as Xamarin iOS?

* **Fody.** Post-compilation MSIL generation/injection. Similar to runtime proxy generation but instead of generating code at runtime it is done at compile-time. This is also an advanced technique requiring a good understanding of MSIL and some considerable patience. Requires the installation of Fody into your tool-chain.

These are all viable alternatives but the "cognitive load" of setting them up and maintaining them is fairly prohibitive. The range of techniques already in use demonstrates a clear desire to solve this problem. The proposal outlined here provides a simple, elegant solution that is baked into the language itself, much easier to understand, and flexible enough to cover a wide variety of use cases.

## Unresolved questions
[unresolved]: #unresolved-questions

One idea to consider further is the generation of "data slots" on the target class.

Taking PropertyChangedBase as an example, any potential implementation of this class must find a way to dynamically store and retrieve property values. A naive approach is to store the properties in a dictionary, but the performance of this would be poor. A smarter implementation could potentially assign slot numbers in an array, or pack primitive types into a byte array to avoid the heap allocation. The static initialization permits a range of decisions to be made ahead-of-time and thus optimize the delegate accordingly.

To help with optimization, the compiler could generate a backing field on the class for each `auto` property and make it available for the implementation to use. Property values could then be stored inline as before. Furthermore, if the slot type was some implementation-specific `Struct<T>` derived from the property type, the class could in theory store more than just a single value against the property. This would support scenarios including old/new values for ORM change tracking etc.
    
It is not clear whether the added complication would destroy the simplicity of the concept.

It is also not clear how these structs would be accessed from utility methods. While the getter/setter methods can accept a `ref Struct<T> backingField` to work with, an implementation like DependencyObject would require its SetBinding() method to have access to the backing field too. The field could be retrieved through reflection (and cached during static initialization) but this is not particularly clean and would require the implementation to have knowledge of the compiler's field-naming conventions.

Is this worth exploring further?

## Design meetings

Link to design notes that affect this proposal, and describe in one sentence for each what changes they led to.


