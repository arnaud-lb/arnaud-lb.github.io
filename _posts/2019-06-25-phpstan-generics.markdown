---
layout: post
title:  "PHPStan Generics"
date:   2019-06-25 10:56:00 +0200
categories: phpstan php generics
---

> This post discusses a work in progress.  
>
> Current status:  
> ☑︎ Generic functions  
> ☐ Generic classes  
> ☐ class-string<T>, T::class

Typing is a great way to document code, to detect some programming errors statically, and to be more confident when refactoring.

PHPStan allows to check the types in a program, and to report any typing issue. However, there is a limit: when re-using the same code with different types (like using different Collection instances with different types), we can not type our code, and we lose PHPStan's type checking.

### Introducing generic typing

Generic typing makes it possible to re-use the same piece of code with different types, while still making it statically type safe.

The idea is to declare one or more types as _templates_ on a function or class. Then, use them as parameter types or return types. PHPStan will determine the real type of the templates by looking at the call site of the function, or the instantiation site of the class. It will then make sure that we use the right types consistently.

In the following example, we declare one template type named `T`, and use it on one parameter and the return value. When calling `f()`, PHPStan determines the real type of `T` from the passed arguments. In this example, one effect of generic typing is that the return type of `f()` is known, so using the return value is safer. 

``` php
<?php

/**
 * @template T // Declares one template type named T 
 * @param T $x // Declares that the type of $x is T
 * @return T
 */
function f($x) {
    // here, the type of $x is the abstract type T
    return $x;
}

f(1); // PHPStan knows that this returns a int
f(new DateTime()); // PHPStan knows that this returns a DateTime
```

#### Propagation

Template types propagate themselves when passing them from one generic code to an other generic code:

``` php
<?php

/**
 * @template T // Declares one template type named T 
 * @param T $x // Declares that the type of $x is T
 * @return T
 */
function f($x) {
    return g($x); // Returns U, which is a T
}

/**
 * @template U
 * @param U $x
 * @return U
 */
function g($x) {
    return $x;
}
```

#### Bounds

Template types can be bound by using the following notation: `@template <name> of <bound>`. The _bound_ is constraining the template type to be a sub-type of the specified bound.

In the following example, we bind `T` to `DateTimeInterface`. This has two distinct effects:
 - We can call `greater()` only with sub-types of `DateTimeInterface`
 - In the function body, we know that `T` is a sub-type of `DateTimeInterface`, so we can use it like a `DateTimeInterface`


``` php
<?php

/**
 * @template T of DateTimeInterface
 * @param T $a
 * @param T $b
 * @return T
 */
function greater($a, $b) {
    // Here, we know that $a and $b are sub-types of DateTimeInterface,
    // so it is legal to call getTimestamp() on them.
    if ($a->getTimestamp() > $b->getTimestamp()) {
        return $a;
    }
    return $b;
}

f(new DateTime("yesterday"), new DateTime("now")); // returns a DateTime
f(new DateTimeImmutable("yesterday"), new DateTimeImmutable("now")); // returns a DateTimeImmutable
f(1, 2); // error: int is not a sub type of DateTimeInterface
```

### Nested types

Template types can be used as standalone types or as embedded types, such as in arrays, callables, iterables, etc.


``` php
<?php

/**
 * @template T
 * @param array<T> $x
 * @return T|null
 */
function first($x) {
    foreach ($x as $value) {
        return $value;
    }
    return null;
}

first([1,2,3]); // returns a int|null
```


``` php
<?php

/**
 * @template T
 * @param callable(T): T $a
 * @param T[] $b
 * @param iterable<T> $c // Note: implementation in progress
 * @param array{a: T, b: T} $d // Note: implementation in progress
 * @return T|null
 */
function example($a, $b, $c, $d) {
    ...
}
```

### class-string&lt;T&gt;, T::class

Generic typing can also work with string class names, like this:


``` php
<?php

/**
 * @template T
 * @param T::class $className
 * @return T
 */
function instance(string $className) {
    return new $className();

    // This is also allowed:
    $object = getObject();
    if ($object instanceof $className) {
        return $object;
    }

    // This, too, is allowed:
    $myClassName = 'DateTime';
    if (is_subclass_of($myClassName, $className)) {
        return new $myClassName;
    }
}

instance("DateTime"); // returns a DateTime
```

Note: Implementation is in progress

### Classes

Generic typing shows its full value when using classes. Like functions, it is possible to declare a template type on a class, and to use this type anywhere in the class definition (properties, parameters, return types).

PHPStan determines the real type of the templates by looking at the instantiation.


``` php
<?php

/** @template T */
class Collection {
    /** @var array<int,T> */
    private $elements;

    /** @param array<T> $elements */
    public function __construct(array $elements) {
        $this->elements = $elements;
    }

    /** @param T $element */
    public function add($element) {
        $this->elements[] = $element;
    }

    /** @return T */
    public function get(int $index) {
        if (!isset($this->elements[$index])) {
            throw new OutOfBoundsException();
        }
        return $this->elements[$index];
    }
}

$coll = new Collection([1,2,3]); // This is a Collection<int> : a collection of ints
$coll->add(4);
$coll->add(""); // Error
$coll->get(0); // int
```

Note: implementation is in progress

### Interfaces, inheritance

When implementing a generic interface, it is possible to specify its template types by using the `@implements` annotation:

``` php
<?php

/** @template T */
interface Collection {
    /** @return T */
    public function get();
}

/** @implements Collection<T> */
class ArrayCollection implements Collection {
    public function get() {
        // ...
    }
}
```

Similarly, we can use the `@extends` or `@uses` annotations for inherited classes or used traits.

The most notable effect of these annotations is that the class is known to be a sub-type of the given interface, class, or trait with the given types.

``` php
<?php

/**
 * @template T
 * @param Collection<T> $coll
 * @return T
 */
function first($coll);

$coll = new ArrayCollection([1,2,3]);

first($coll); // int
```

Note: implementation is in progress.

### Variance

Variance defines how a type can be substituted while maintaining type safety.

#### Covariance

Covariance is when a type can be substituted by itself or any of its subtypes.

For example, callable types are covariant on their return type: If some code works after receiving a B from a callable, the same code will still work after receiving a subtype of B from an other callable (thanks to the liskov substitution principle, if I can use a type, I can use any of its subtypes).

``` php
<?php

class A {}
class B extends A {
    function hello() {}
}
class C extends B {}

/** @param callable():B $cb */
function f($cb) {
    $b = $cb();
    $b->hello();
}

f(function (): B { return new B(); }); // OK
f(function (): C { return new C(); }); // OK
f(function (): A { return new A(); }); // Not allowed
```

#### Contravariance

Contravariance is when a type can be substituted by itself or any of its supertypes.

For example, callable types are contravariant on their parameter types: If some code can call a callable, it can also call the same callable whose parameters are supertypes of the original parameters and whose return type is the same. Conversely, it can not call the same callable whose parameters are subtypes.

``` php
<?php

class A {}
class B extends A {
    function hello() {}
}
class C extends B {}

/** @param callable(B):void $cb */
function f($cb) {
    $cb(new B());
}

f(function (A $a) {}); // OK
f(function (B $a) {}); // OK
f(function (C $a) {}); // Not allowed
```

#### Invariance

Invariance is when a type can be substituted only by itself.

#### Variance in generics

A class can be seen as a collection of callables who follow the same rules as explain above: Each method is covariant on its parameter types, and contravariant on its return type.

In a generic class a template type can be used as a parameter type and as a return type at the same time, so making the template type either covariant or contravariant would break type safety.

Because of this, generic types are invariant on the template types by default.

This becomes clear by looking at an example:

``` php
<?php

/**
 * @template T
 */
class Collection {
    /** @param T $item */
    public function __construct($item);

    /** @return T $item */
    function get(int $index);

    /** @param T $item */
    function set(int $index, $item);
}

class A {}
class B extends A {}
class C extends B {}

/** @var Collection<B> */
function f($coll): B {
    $coll->set(0, new B());
    return $coll->get(1);
}

$coll = new Collection(new B()); // Collection<B>
$f($coll); // OK

$coll = new Collection(new C()); // Collection<C>
$f($coll); // Not OK, because f() sees $coll as a Collection<B>, which would
           // allow it to call $coll->set(0, new B())

$coll = new Collection(new A()); // Collection<A>
$f($coll); // Not OK, because f() sees $coll as a Collection<B>, although
           // $coll->get(1) returns A.
```

As can be seen in this example, allowing variance in template types would not be type safe.

There are ways for a type system to allow variance selectively; this is a planned feature.

### Acknowlegements

Thanks to [Mention](https://mention.com) for letting me work on this feature. BTW, Mention is [hiring](https://mention.workable.com/).

Thanks to [Ondřej Mirtes](https://twitter.com/ondrejmirtes), author of PHPStan, for his help.
