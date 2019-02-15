# Conditional Types

1. [Introduction](#introduction)
2. [Syntax](#syntax)
3. [Distribution](#distributive-conditional-types)
4. [Instantiation](#instantiation)
5. [Resolution](#resolution)
6. [Typing Relations](#typing-relations)

## Introduction

Conditional types are the type analogue to conditional expressions. A conditional expression produces an expression based on the satisfaction of a boolean condition; a conditional type produces a type based on the satisfaction of an assignability condition.

#### Example - *Conditional Expression*

The code below defines a function using a conditional expression. When the function is applied to an argument that is equal to `null` the function returns the argument `0`, otherwise the function returns the argument `x`.
```ts
const checkNull = x => x === null ? 0 : x;
checkNull(null) // 0
checkNull("hi") // "hi"
```

#### Example - *Conditional Type*

The code below defines a type alias using a conditional type. When the type alias is applied to a type that is assignable to `null` the alias returns the type `number`, otherwise the alias returns the type `X`.
```ts
type CheckNull<X> = X extends null ? number : X
CheckNull<null>   // number
CheckNull<string> // string
```

#### Overview

Conditional types are a powerful mechanism to compose types and construct rich interfaces. The central concepts of conditional types are distribution, instantiation, and reduction; each concept is discussed in this document. We also describe how conditional types relate to other types under the typing relations. 

First, we begin by presenting the syntax and declaration of conditional types.

## Syntax

Let `T`, `U`, `A`, and `B` denote types. Let `X` and `Y` denote type parameters. This syntax is convention, not specification. When we deviate from convention or introduce new syntax, we make this explicit.

A conditional type has the form
```ts
T extends U ? A : B
```

We refer to `T` as the _check_ type, `U` as the _extends_ types, `A` as the _true_ type, and `B` as the _false_ type. Conditional types must have both branches specified.

## Distributive Conditional Types

A distributive conditional type is a particular variant of conditional type. The essence of a distributive conditional type is that it will apply itself to each component of a union type, while a non-distributive conditional type will treat a union type atomically. 
The distributive property of a conditional type is determined at the definition of the type. A distributive conditional type is declared using the form:

```ts
X extends U ? A : B
```

Specifically, a distributive conditional type is declared by defining a conditional type where the check type is a _naked type parameter_.

#### Example - *Defining Distributive and Non-distributive Conditional Types*

The type `CheckNull` is a distributive conditional type because the check type is the naked type parameter `X`. The type `StringIs` is a non-distributive conditional type because the check type is _not_ a naked type parameter---the check type is the concrete type `string`. Naked type parameters that occur in the extends type do not affect distribution; distribution _only_ happens in the check type.
```ts
type CheckNull<X> = X extends null ? number : X
type StringIs<X>  = string extends X ? true : false
```

Distribution occurs when the check type is replaced by a union type; this process is known as [instantiation](#instantiation). The union type will be decomposed and the conditional type is applied to each component.

#### Example - *Distributing Over Union*
When `CheckNull` is applied to a union type the conditional type is first distributed over each union branch, then the conditional type is resolved for each branch. Resolving a conditional type is the process of simplifying a conditional type. Think of resolution as attempting to "evaluate" the conditional type. The semantics of resolution are defined in [Resolution](#resolution).

```ts
CheckNull<null | string> // number | string

//     CheckNull<null | string>
// --> (null extends null ? number : null) | (string extends null ? number : string)
// --> number | string
```

#### Example - *Treating Union Atomically*
A non-distributive conditional type treats union types atomically. When the extends type is replaced by the union type `null | string` we do not apply the conditional type to each componenent; the union type is viewed atomically. 
```ts
StringIs<null | string> // true

//     StringIs<null | string>
// --> string extends (null | string) ? true : false
// --> true
```

A second trait of distributive conditional types is that they _short-circuit_ when the check type is replaced by `never`. The technical intuition is that `never` denotes the _empty union type_; we are distributing over "nothing". The result of replacing the check type with `never` is immediately `never`.

Technically this trait follows immediately from the definition of mapping over union types, and is not an explicitly defined secondary behaviour. We distinguish the two traits because it practice it is not obvious that short-circuiting follows from distribution.

Non-distributive conditional types do not short-circuit. Replacing the extends type with `never` will not cause the conditional type to immediately resolve to `never`.

#### Example - *Distributing Over `never`, or Short-circuiting*
```ts
CheckNull<never> // never

//     CheckNull<never>
// --> never extends null ? number : null
// --> never


IsString<never> // false

//     IsString<never>
// --> string extends never ? true : false
// --> false
```

### Disabling Distribution

There are situations where it is desirable to define a non-distributive conditional type where the check type is a type parameter. The canonical example is the type `IsNever<X>`. The type should return `true` when `X` is `never` and `false` otherwise.

#### Example - *Failing to Capture `never`*
The following example is defined using a distributive conditional type which does not give the desired behaviour because it short-circuits on `never`.
```ts
type IsNeverWrong<X> = X extends never ? true : false
IsNeverWrong<never> // never

//     IsNeverWrong<never>
// --> never extends never ? true : false
// --> never
```

To prevent distribution, wrap the _check_ and _extends_ type using a one-tuple.

#### Example - *Suppressing Distribution*
```ts
type IsNever<X> = [X] extends [never] ? true : false
IsNever<never> // true

//     IsNever<never>
// --> [never] extends [never] ? true : false
// --> true
```

The check type is not a naked type parameter and therefore `IsNever` is not a distributive conditional type. Assignability lifts to tuples: if `A extends B`, then `[A] extends [B]`. Using the one-tuple in the type retains the correct conditional behaviour, without the distribution.

The core mechanism behind distribution is _instantiation_: the act of applying type arguments to a generic type. We now discuss the semantics of instantiation for conditional types.

## Instantiation

Applying type arguments to a generic type will _instantiate_ the type parameters in the body of the generic type; in other words, the type parameters are substituted for their arguments. Instantiations, or substitutions, can be modelled in multiple ways. We present the approach taken in the TypeScript compiler and use a function called a _mapper_. A type mapper is a function from type parameters to types, transforming type parameters into their instantiated type.

#### Example - *Type Instantiation and Mappers*

Applying `CheckNull` to the type `null | string` will instantiate type parameter `X` with the type `null | string`. The instantiation can be modelled using a mapper function on type parameters. 

```ts
type CheckNull<X> = X extends null ? number : X
CheckNull<null | string>

// Instantiate X to null | string
// The mapper is a function: Y => Y === X ? (null | string) : Y;
```

If the argument type parameter `Y` is equal to type parameter `X` then return the type argument `null | string`, otherwise return parameter `Y`.

We abstractly represent type mappers using the following notation. Let `M`, and `M'` range over type mappers. We write `(M . [X := U])` to denote a mapper that maps type parameter `X` to type `U`, and forwards all other type parameters to mapper `M`. Informally, this can be defined as:

```
(M . [X := U]) is defined as  Y => Y === X ? U : M(X)
```

We denote the application of a type mapper to a type variable using the notation `M(X)`.

We write `ID` for the identity type mapper; a mapper that sends all type parameters to themselves. Informally, this can be defined as:

```
ID is defined as Y => Y
```

In our example, the instantiation of `CheckNull<null | string>` can be represented using the mapper `(ID . [X := (null | string)])`. Unfolding the definition makes the behaviour clear:

```
(ID . [X := (null | string)]) is defined as Y => Y === X ? (null | string) : ID(Y)
```

We overload the notation `M(T)` to denote the direct application a mapper `M` to type `T`. This is the instantiation of type `T` using mapper `M`. In our example, we denote the instantiation of `CheckNull<null | string>` as:

```
(ID . [X := (null | string)])(CheckNull<X>)
```

### Conditional Type Instantiation

The instantiation of a conditional type implements the distributive behaviour. Take the distributive conditional type:

```ts
X extends U ? A : B
```

The semantics of instantiating a distributive condition type with mapper `M`, written `M(X extends U ? A : B)`, is defined as follows:

Define `M(X extends U ? A : B)` as:
- If `M(X)` is a union type `(L | R)`, for some types `L` and `R`, then distribute as:
    * `(M . [X := L])(X extends U ? A : B) | (M . [X := R])(X extends U ? A : B)`.
- If `M(X)` is `never`, then distribute over nothing and return `never`.
- Otherwise, `resolve(X extends U ? A : B, M)`.


Instantiation of a non-distributive conditional type immediately proceeds to resolution. Take the non-distributive conditional type:
```ts
T extends U ? A : B
```

The semantics of instantiating the type with mapper `M`, written `M(T extends U ? A : B)`, is defined as follows:

Define `M(T extends U ? A : B)` as:
- `resolve(X extends U ? A : B, M)`.

The function `resolve(T extends U ? A : B, M)` is defined in the following section.

## Resolution

## Typing Relations
