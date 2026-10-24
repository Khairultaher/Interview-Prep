# Shallow Clone vs. Deep Clone in C#

Cloning creates a new object based on an existing object. The key difference is whether referenced objects are shared or copied.

## 1. Shallow clone: `MemberwiseClone()`

`Object.MemberwiseClone()` is a **protected** method provided by .NET. It creates a new instance and copies the current object's fields:

- **Value-type fields** (such as `int` and `bool`) are copied by value.
- **Reference-type fields** (such as `Address`) copy the reference, so the original and clone point to the same referenced object.

### Example

```csharp
public sealed class Address
{
    public string City { get; set; } = "";
}

public class Person
{
    public string Name { get; set; } = "";
    public Address Address { get; set; } = new();

    public Person ShallowClone() =>
        (Person)MemberwiseClone();
}

var original = new Person
{
    Name = "Alice",
    Address = new Address { City = "New York" }
};

var clone = original.ShallowClone();

clone.Name = "Bob";
clone.Address.City = "London";

Console.WriteLine(original.Name);         // Alice
Console.WriteLine(original.Address.City); // London
Console.WriteLine(ReferenceEquals(original.Address, clone.Address)); // True
```

`Name` is a `string`. Assigning a different string to `clone.Name` does not change `original.Name`. The `Address` property, however, points to the same `Address` instance in both objects, so changing its `City` is visible through both references.

> `MemberwiseClone()` is shallow: it copies fields, not the entire referenced object graph.

## 2. Deep clone

A **deep clone** creates a separate copy of the object and of the mutable objects it references, recursively as required by the application's data model.

### Example

```csharp
public class Person
{
    public string Name { get; set; } = "";
    public Address Address { get; set; } = new();

    public Person DeepClone() =>
        new()
        {
            Name = Name,
            Address = new Address
            {
                City = Address.City
            }
        };
}

public sealed class Address
{
    public string City { get; set; } = "";
}

var original = new Person
{
    Name = "Alice",
    Address = new Address { City = "New York" }
};

var clone = original.DeepClone();
clone.Address.City = "London";

Console.WriteLine(original.Address.City); // New York
Console.WriteLine(clone.Address.City);    // London
Console.WriteLine(ReferenceEquals(original.Address, clone.Address)); // False
```

The `Address` object is explicitly created again, so changes to the cloned address do not affect the original. This implementation is deep only for the fields shown; if the model gains more mutable reference-type fields, those must be copied too.

## 3. Comparison

| Aspect | Shallow clone (`MemberwiseClone`) | Deep clone |
|---|---|---|
| New top-level object | Yes | Yes |
| Value-type fields | Copied | Copied |
| Reference-type fields | References are copied and shared | Referenced mutable objects are copied as defined by the cloning logic |
| Nested changes independent? | Not necessarily | Yes, for objects that are deeply copied |
| Built into `Object`? | Yes | No universal built-in deep-clone method |
| Implementation effort | Minimal | Requires explicit copying or a suitable cloning strategy |

## 4. When should you use each?

- **Use a shallow clone** when sharing referenced objects is intentional or when the object is designed so that sharing is safe.
- **Use a deep clone** when the clone must be independently mutable, including its nested objects.
- **Consider immutable types or records** when you need safer value-like data handling. A C# record's `with` expression is a *shallow copy*; nested reference-type members remain shared unless you explicitly replace or copy them.

## 5. Important considerations

- `MemberwiseClone()` is `protected`, so you normally expose it through a method inside the class or a derived class.
- `ICloneable` exists in .NET, but its `Clone()` contract does not specify whether the result is shallow or deep. For public APIs, a clearly named method such as `ShallowClone()` or `DeepClone()` is less ambiguous.
- JSON serialization/deserialization can sometimes be used to copy simple DTO graphs, but it may change behavior for types with cycles, special reference relationships, unsupported members, custom converters, or non-serializable state. It should not be treated as a universal deep-clone solution.
- Deep cloning object graphs with cycles or shared references needs care: a correct implementation may need a map of already-cloned objects to preserve relationships and avoid infinite recursion.

**In short:** `MemberwiseClone()` creates a new outer object but shares referenced objects. A deep clone also creates independent copies of the nested objects that need to be independent.
