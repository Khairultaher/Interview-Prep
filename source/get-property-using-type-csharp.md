# Get a Property Using `Type` in C

In C#, you can use **reflection** to retrieve a property by its name at
runtime. The `Type.GetProperty()` method returns a `PropertyInfo`
object, which you can use to read the property's value.

## Example

``` csharp
using System;
using System.Reflection;

public class MyClass
{
    public int MyProperty { get; set; }
}

public class Program
{
    public static void Main()
    {
        var obj = new MyClass
        {
            MyProperty = 42
        };

        string propertyName = "MyProperty";

        // Get the runtime type of the object.
        Type type = obj.GetType();

        // Find the property by name.
        PropertyInfo? propertyInfo = type.GetProperty(propertyName);

        if (propertyInfo is not null)
        {
            // Read the property's value.
            object? value = propertyInfo.GetValue(obj);

            Console.WriteLine($"The value of {propertyName} is: {value}");
        }
        else
        {
            Console.WriteLine($"Property '{propertyName}' was not found.");
        }
    }
}
```

## How It Works

1.  `obj.GetType()` returns the runtime type of the object.
2.  `type.GetProperty(propertyName)` searches for a public property with
    the specified name.
3.  `propertyInfo.GetValue(obj)` reads that property's value from the
    object.
4.  If the property does not exist, `GetProperty()` returns `null`.

## If You Already Have a `Type`

If you have the type rather than an object instance, you can look up the
property like this:

``` csharp
Type type = typeof(MyClass);
PropertyInfo? propertyInfo = type.GetProperty("MyProperty");
```

To read the value, you still need an instance:

``` csharp
var obj = new MyClass { MyProperty = 42 };
object? value = propertyInfo?.GetValue(obj);
```

> **Note:** Reflection is useful when the property name is known only at
> runtime, but it is less type-safe and usually slower than accessing a
> property directly.
