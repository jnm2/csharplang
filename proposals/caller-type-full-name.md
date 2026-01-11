# Caller type full name

Champion issue: TBD

## Summary

```cs

```

## Motivation

There are steady and vocal requests for the ability to access a fully qualified type name string for use in logging.

## Detailed design

The compiler will recognize the following attribute by its namespace-qualified type name.

```cs
namespace System.Runtime.CompilerServices;

[AttributeUsage(AttributeTargets.Parameter, Inherited = false)]
public sealed class CallerTypeFullNameAttribute : Attribute;
```

An error is produced if the attribute is applied to a parameter when there is no implicit conversion from System.String to the parameter's type, or when the parameter does not have a default value.

As with other caller attributes, the compiler substitutes a constant value at call sites which do not specify the optional parameter. In this case, the value is a constant string representing the declaring type of the caller.

The caller is determined in the same way as with the [CallerMemberName attribute](https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/attributes.md#23564-the-callermembername-attribute). For example, lambda expressions and local functions are not considered to be callers.

The declaring type of the caller is formatted at compile time into a constant string as follows:

- If the type is nested, first determine the fully formatted type name of the containing type. Output this name, then output a `.` character. (This step is recursive.)
- If the type is not nested and it has a namespace, output the full namespace name with all namespace name parts joined with the `.` character, then output another `.` character. (No alias qualification is included.)
- Output the simple C# type name. (Not the metadata name, which may include a backtick for generic types.)
- If the type is generic, output `<`, then output a number of `,` characters, then output `>`. The number of `,` characters is one less than the type's generic arity.

For an alternative formatting, see [Metadata names instead of C# names](#metadata-names-instead-of-c-names).

## Specification

[§23.5.6 Caller-info attributes](https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/attributes.md#2356-caller-info-attributes) is updated as follows:

## Alternatives

### Metadata names instead of C# names

There are a few options for formatting of generic arities:

- C# syntax: `SomeDictionary<,>`
- Metadata name: ``SomeDictionary`2``
- No distinction: `SomeDictionary`

There are likewise a few options for formatting of type nesting:

- C# syntax: `SomeNamespace.SomeType.NestedType`
- Metadata name: `SomeNamespace.SomeType+NestedType`
- ECMA-335 IL syntax: `SomeNamespace.SomeType/NestedType`

### `GetType()` and dynamic string building

One possible alternative approach would be for the compiler to generate a `GetType()` call at runtime and to build the string at runtime using a generated compiler helper. This would show a difference in behavior in two ways:

1. The caller would appear to be the most derived instance of the class. This is likely a downside, since you lose the distinction of which of these two `M` methods is the caller:

  ```cs
  class Base
  {
      public static void HelperMethod([CallerTypeFullName] string callerType = null)
      {
          Console.WriteLine(callerType);
      }

      public virtual M()
      {
          HelperMethod()
      }
  }

  class Derived : Base
  {
      public override M()
      {
          base.M();
          HelperMethod()
      }
  }
  ```

1. Generic type arguments would optionally be able to be included in the full name. Rather than emitting `Namespace.SomeClass<>` or similar, the compiler could recursively format `Namespace.SomeClass<System.Collections.Generic.List<int>>` as the caller type full name. This fluctuation may or may not be helpful for loggers, and it opens a can of worms about formatting, e.g. `int` versus `System.Int32`.

This is expected to be a significant enough overhead that it is not suitable for logging, even with caching mechanisms.

## Expansions

A System.Type-based parameter could be easily provided by the compiler and would allow the user to make their own choices about how to format the type name. Accessing the `Namespace`, `Name`, and `DeclaringType` properties would put the user fully in the driver's seat. There would be a question whether the compiler would always emit the same code as for a `typeof` expression, or whether in instance members the compiler would emit `GetType()` which would allow users to also access `GenericTypeArguments` and follow their own policies on whether to include that information and how to format it.

There is a question whether the performance characteristics of passing and introspecting a System.Type are unfavorable enough that there should be no implicit passing of a System.Type object at all, and users should write out fully what they want: `typeof(MyClass<>)` vs `typeof(MyClass<T>)` vs `GetType()`.
