# Readonly fields in `with` and object initializers

Champion issue: TBD

## Summary

Readonly fields may now be updated in `with` expressions or object initializer expressions which appear within the same type that declares the field:

```cs
record ImmutableComponent
{
    private readonly int privateState;

    public int PublicProperty { get; init; }

    public int Compute() => privateState * PublicProperty;

    public ImmutableComponent Update()
    {
        return this with { privateState = privateState + 1 };
    }

    public ImmutableComponent Reset(int someParam)
    {
        return new ImmutableComponent { privateState = Computation(someParam) };
    }
}
```

This brings parity with the way that `init` accessors and constructors are able to set readonly fields during the construction phase of an object.

## Motivation

Records make it easy to use immutable object models because they provide the `with` expression to produce a modified copy nondestructively. Some objects have both public state and private state. The public state is exposed publicly on the type through a get-only or init property. The private state, as an implementation detail, should not be publicly visible on the type. The default vehicle for storing private state is a private field. To easily verify that the type is actually immutable, the private field is naturally marked `readonly`.

An immutable object may already be using `with` to return modified copies of itself, calling private init accessors in a coordinated fashion. In the example below, a SetRange method is provided instead of allowing Min and Max properties to be set directly by the caller, so that validation is simplified:

```cs
record Example
{
    public int Min { get; private init; }
    public int Max { get; private init; }

    public Example SetRange(int min, int max)
    {
        // ... [validate inputs]
        return this with { Min = min, Max = max };
    }
}
```

An issue arises when the object also wants to update its private state. If the typical choice of a private readonly field was used, the `with` expression can no longer be used for this update. The object will have to be constructed from scratch with every member assigned, losing the benefits of `with` which preserve members that are not being reassigned.

Alternatively, a class can use private init properties instead of readonly fields:

```cs
record ImmutableComponent
{
    private int PrivateState { get; init; }

    public int PublicProperty { get; init; }

    public int Compute() => PrivateState * PublicProperty;

    public ImmutableComponent Update()
    {
        return this with { PrivateState = privateState + 1 };
    }

    public ImmutableComponent Reset(int someParam)
    {
        return new ImmutableComponent { PrivateState = Computation(someParam) };
    }
}
```

This works, but it is unusual to use a property for private state. Properties are a slightly "heavier" language concept. They are recommended as a way to mediate public access to state, but that recommendation is not relevant for private state. Properties are also not doing anything here that a field could not do, in concept. Private readonly fields can already be set within `init` accessors. Setting them during the object construction phase after the constructor ends is already something the language can do.

Thus, this feature removes a non-essential blocker from the language which has penalized the usability of fields in immutable types. The cost to do so is low, as well, since the only change in compiler behavior is to loosen an error.

## Detailed design

Readonly fields may now be updated in `with` expressions and object initialization expressions, subject to the same conditions in which they may be assigned to in `init` setters. The object is still in its construction phase until the `with` expression or object initialization expression ends. During that time, its own code may assign to its readonly fields.

Readonly fields are thus assignable in all the same ways that a property having a `private init` accessor is assignable.

The message of error CS0191 is updated from:

> A readonly field cannot be assigned to (except in a constructor or init-only setter of the type in which the field is defined or a variable initializer)

to:

> A readonly field cannot be assigned to (except in its initializer, or in a constructor, `with` expression, or init-only setter of the type in which the field is defined)

## Alternatives

A larger feature could be introduced, `init` fields, which would gain the ability to be assigned only in `with`, object initializers, and `init` accessors. The ability that is gained would be to extend this behavior to non-private fields. However, given that properties are recommended instead of fields when non-private, this ability has questionable value.
