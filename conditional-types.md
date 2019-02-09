# Conditional Types

Conditional types are the type analogue to conditional expressions. A conditional expression produces an expression based on the satisfaction of a boolean condition; a conditional type produces a type based on the satisfaction of an assignability condition.

#### Example
The code below defines a function using a conditional expression. When the function is applied to an argument that is equal to `null` the function returns the argument `0`, otherwise the function returns the argument `x`.
```ts
const checkNull = x => x === null ? 0 : x;
checkNull(null) // 0
checkNull(42)   // 42
```

The code below defines a type alias using a conditional type. When the type alias is applied to a type that is assignable to `null` the alias returns the type `number`, otherwise the alias returns the type `X`.
```ts
type CheckNull<X> = X extends null ? number : X
CheckNull<null>   // number
CheckNull<string> // string
```


Conditional types are a powerful mechanism to compose types and construct rich interfaces. The central concepts of conditional types are distribution, instantiation, and reduction; each concept is discussed in this document. First, we describe the syntax and declaration of conditional types.

## Syntax

Let `T`, `U`, `A`, and `B` denote types. Let `X` and `Y` denote type parameters. This syntax is convention, not specification. When we deviate from convention, or introduce new syntax, we make this explicit.

A conditional type has the form
```ts
T extends U ? A : B
```

We refer to `T` as the _check_ type, `U` as the _extends_ types, `A` as the _true_ type, and `B` as the _false_ type. Conditional types must have both branches specified.

## Distributive Conditional Types

A distributive conditional type is a particular variant of conditional type that is characterised by its instantiation and reduction behaviour. The distributive property of a conditional type is determined at the definition of the type. A distributive conditional types is declared using the form:

```ts
X extends U ? A : B
```

Specifically, a distributive conditional type is declared by defining a conditional type where the check type is a _naked type parameter_.

#### Example

The type `CheckNull` is a distributive conditional type because the check type is the naked type parameter `X`. The type `StringIs` is a non-distributive conditional type because the check type is _not_ a naked type parameter---the check type is the type `string`.
```ts
type CheckNull<X> = X extends null ? number : X
type StringIs<X>  = string extends X ? true : false
```

The salient trait of a distributive conditional type is its ability to distribute over union types.

#### Example
When `CheckNull` is applied to a union type the conditional is first distributed over each union branch, then the conditional type is resolved for each branch.
```ts
CheckNull<null | string> // number | string

//     CheckNull<null | string>
// --> CheckNull<null> | CheckNull<string>
// --> (null extends null ? number : null) | (string extends null ? number : string)
// --> number | string
```

#### Example
A non-distributive conditional type  treats union types atomically; the conditional type is resolved using the complete union type.
```ts
StringIs<null | string> // true

//     StringIs<null | string>
// --> string extends (null | string) ? true : false
// --> true
```

A second trait of distributive conditional types is that they _short-circuit_ on application to never. The technical intuition is that `never` denotes the _empty union type_, and therefore there is nothing to distribute over; consequently, the result is the empty union type `never`. Non-distributive conditional types do not short-circuit, rather, they treat `never` like any other type.

#### Example
```ts
CheckNull<never> // never

IsString<never> // false

//     IsString<never>
// --> string extends never ? true : false
// --> false
```

### Disabling Distribution

There are situations where it is desirable to define a non-conditional type, where the check type is a type parameter. The canonical example is the type `IsNever<X>`. The type should return `true` when `X` is `never` and `false` otherwise.

#### Example
The following example is defined using a distributive conditional type and does not give the desired behaviour because it short-circuits on `never`.
```ts
IsNeverWrong<X> = X extends never ? true : false
IsNeverWrong<never> // never
```

To prevent distribution, wrap the _check_ and _extends_ type using a one-tuple.

#### Example
```ts
IsNever<X> = [X] extends [never] ? true : false
IsNever<never> // true
```

The check type is not a naked type parameter and therefore `IsNever` is not a distributive conditional type. Assignability lifts to tuples: if `A extends B`, then `[A] extends [B]`. Using the one-tuple in the type retains the correct conditional behaviour, without the distribution.

The mechanism behind distribution is _instantiation_: the act of applying type arguments to a generic type. We know discuss the semantics of instantiation for conditional types.

## Instantiation

Applying type arguments to a generic type will _instantiate_ the type parameters in the body of the generic type; in other words, the type parameters are substituted for their arguments. Instantiations, or substitutions, can be modelled in multiple ways. We present the approach taken in the TypeScript compiler and use a function called a _mapper_. A type mapper is a function from type parameters to types, transforming type parameters into their instantiated type.

#### Example
Applying `CheckNull` to the type `null | string` will instantiate type parameter `X` with the type `null | string`. The instantiation can be modelled using a mapper function on type parameters. If the argument parameter `Y` is equal to `X` then return the type argument `null | string`, otherwise return parameter `Y`.
```ts
type CheckNull<X> = X extends null ? number : X
CheckNull<null | string>

// Instantiate X to null | string
// The mapper is a function: Y => Y === X ? (null | string) : Y;
```

We abstractly represent type mappers using the following notation. Let `M`, and `M'` range over type mappers. We write `(M . [X := U])` to denote a mapper that sends type parameter `X` to the type `U`, and forwards all other type parameters to mapper `M`. We write `ID` for the identity type mapper; a mapper that sends all type parameters to themselves. In our example, the instantiation of `CheckNull<null | string>` can be represented using the mapper `(ID . [X := (null | string)])`.

We denote the application of a type mapper to a type using the notation `M(T)`. We overload the notation `M(T)` to denote the direct application of the mapper when `T` is a type parameters `X`. 

In our example, we denote the instantiation of `CheckNull<null | string>` as:
```
(ID . [X := (null | string)])(CheckNull<X>)
```

### Conditional Type Instantiation

The instantiation of a conditional type implements the distributive behaviour. Take the distributive conditional type:
```ts
X extends U ? A : B
```

The semantics of instantiating the type with mapper `M`, written `M(X extends U ? A : B)`, is defined as follows:

- If `M(X)` is a union type `(L | R)`, for some types `L` and `R`, then distribute as:
    * `(M . [X := L])(X extends U ? A : B) | (M . [X := R])(X extends U ? A : B)`.
- If `M(X)` is `never`, then distribute over nothing and return `never`.
- Otherwise, _resolve_ conditional type `(X extends U ? A : B)` using mapper `M`.

Resolving a conditional type is the process of simplifying a conditional type in the context of a type mapper. Think of resolution as attempting to "evaluate" the conditional type in the presence of type arguments. The semantics of resolution are defined in the following section.

Instantiation of a non-distributive conditional type immediately proceeds to resolution. Take the non-distributive conditional type:
```ts
T extends U ? A : B
```

The semantics of instantiating the type with mapper `M`, written `M(T extends U ? A : B)`, is defined as follows:

- Resolve conditional type `(T extends U ? A : B)` using mapper `M`.


## Conditional Type Resolution





