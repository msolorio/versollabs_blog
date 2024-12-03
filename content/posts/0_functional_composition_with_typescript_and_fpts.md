+++
date = '2022-11-10T15:17:33-08:00'
draft = false
title = 'Grasping functional composition with TypeScript and fp-ts'
+++

I had the opportunity to work with <a href="https://gcanti.github.io/fp-ts/" target="_blank">fp-ts</a> on a couple of recent production projects that involved transformations and validations on data. fp-ts is a <a href="https://www.typescriptlang.org" target="_blank">TypeScript</a> library for providing popular patterns from typed functional languages like Scala and Haskell. Functional composition helped streamline operations on data and resulted in code that was surprisingly easy to reason about.

The project culminated in a workshop I lead for the engineering department to share some of the team's learnings and open a discussion around other projects at the company that could benefit from these patterns.

I wanted to share out some of those learnings here for the wider community and the result is this compact primer.

---

### Composition
With functional composition, multiple functions are strung together in a sequence to create a larger function that has their combined behavior. Data can then be piped through that sequence; each function outputting the input to the next function.

```ts
import { pipe } from 'fp-ts/lib/function'

const result = pipe(
  'apple',
  getLength,
  double
) // => 10
```

fp-ts's `pipe` method allows us to compose functions together. The first argument is the initial input and subsequent arguments are the functions that make up the composition.

The string `'apple'` is fed to the `getLength` function. The return value of `getLength` is fed to the `double` function. And the return value of `double` is the return value of the `pipe` invocation.

---

We could parameterize this composition.
```ts
import { pipe } from 'fp-ts/lib/function'

const getDoubleLength = (myString: string): number => {
  return pipe(
    myString,
    getLength,
    double
  )
}

getDoubleLength('apple') // => 10
```

---

fp-ts's `flow` function is another tool for function composition. `flow` returns a function that can be invoked later.

```ts
import { flow } from 'fp-ts/lib/function'

const getDoubleLength: (str: string) => number = flow(
  getLength,
  double
)

getDoubleLength('apple') // => 10
```

So far, our model of composition can be visualized by a set of railroad track pieces in a line.

![function composition](/images/function-composition.png)

---

### Either

When composing functions there are times when a function may return an error, or may return one of two value types.

Using fp-ts's `Either` we can specify a `left` return value and a `right` return value for a function.

Take a function that checks if a provided user has pets.
```ts
const checkHasPets = (user: User): Either<Error, User> => {
  return user.hasPets
    ? E.right(user)
    : E.left(generateError('user has no pets'))
} 
```

The function returns a `Left` holding an error or a `Right` holding the original user.

The `Either` type is the union of types `Left` and `Right`.
```ts
type Either<E, A> = Left<E> | Right<A>
```

We might want to compose together many functions that return `Either`s.

![function composition with eithers](/images/function-compostion-w-eithers.png)

If the result of one function is of type `Right` pass the value to the next function. If the return value is of type `Left` exit the compostion and return early.

However, these pieces don't exactly fit.

Each function has type `(a: A) => Either<E, B>`. Each function returns a value wrapped in an `Either` but each function takes as input an unwrapped value.

This is where `chain` comes in...

---

### Chain

Chain allows us to convert a one-to-two track function to a full two-track function.

![chain function](/images/chain.png)

Chain converts a function from type `(a: A) => Either<E, B>` to a function of type `(ma: Either<E, A>) => Either<E, B>`. Put another way `chain` converts a function that accepts an unwrapped value to a function that accepts a value wrapped in an `Either`.

Now that we have full two-track functions these can be composed properly.

![chained function composition](/images/chained-function-composition.png)

An example in fp-ts could use composition to check if a given user has a paid account, is a senior member, and has a number of pets between 1 and 10. If so return the user. Otherwise return an error.

```ts
const validateUser = (user: User): E.Either<Error, User> => {
  return pipe(
    user,
    hasPaidAcct,
    E.chain(isSeniorMember),
    E.chain(hasCorrectPetAmnt)
  )
}
```

Here are the sequence of events.
1. `hasPaidAcct` returns an `Either`.
1. This `Either` is fed to the `chain`ed version of `isSeniorMember`, which can now accept an `Either`.
1. If the `Either` is a `Left`, exit the composition with that `Left`.
1. If the `Either` is a `Right`, the value is unwrapped and fed as input to `isSeniorMember`.

![chained unwrapping](/images/chained-unwrapping.png)

<details>
  <summary><u>See the full code</u></summary>


```ts
import { pipe } from 'fp-ts/lib/function'
import * as E from 'fp-ts/lib/Either'

interface Pet {
  name: string
  age: number
}

interface User {
  pets: Array<Pet>
  hasPaidAcct: boolean
  startMembershipYear: number
  username: string
  id: number
}

const currentYear = 2022

const hasPaidAcct = (user: User): E.Either<Error, User> => {
  return user.hasPaidAcct
    ? E.right(user)
    : E.left(Error('user does not have paid account'))
}

const isSeniorMember = (user: User): E.Either<Error, User> => {
  const isSeniorMember = (currentYear - user.startMembershipYear) >= 10

  return isSeniorMember
    ? E.right(user)
    : E.left(Error('user is not senior member'))
}

const hasCorrectPetAmnt = (user: User): E.Either<Error, User> => {
  return user.pets.length && user.pets.length <= 10
    ? E.right(user)
    : E.left(Error('user has no pets'))
}

const validateUser = (user: User): E.Either<Error, User> => {
  return pipe(
    user,
    hasPaidAcct,
    E.chain(isSeniorMember),
    E.chain(hasCorrectPetAmnt)
  )
}

const user1 = {
  pets: [
    { name: 'Fido', age: 5 },
    { name: 'Felix', age: 1 },
  ],
  hasPaidAcct: true,
  startMembershipYear: 2000,
  username: 'animallover789',
  id: 1234
}

const result = validateUser(user1)
```

</details>

---

### Map

`map` is a utility similar to `chain`. It allows us to convert a full one-track function to a full two-track function.

![map function](/images/map.png)

`map` converts a function of type `(a: A) => B` to a function of type `(fa: Either<E, A>) => Either<E, B>`. It converts a function that receives and returns an unwrapped value to a function that recieves and returns a value wrapped in an `Either`.

This allows us to include a function of type `(a: A) => B` into a composition that deals in `Eithers`.

Take the function `getPets`. It takes a user and returns their list of pets. This function has no knowledge of `Eithers` and works only in unwrapped values.
```ts
const getPets = (user: User): Array<Pet> => user.pets
```

Now take our previous example that validate if a user has a paid account, is a senior member, and has the correct amount of pets. If they pass all criteria, return the user's list of pets.

```ts
const getPetsOrErr = (user: User): E.Either<Error, Array<Pet>> => {
  return pipe(
    user,
    hasPaidAcct,
    E.chain(isSeniorMember),
    E.chain(hasCorrectPetAmnt),
    E.map(getPets)
  )
}
```

Here are the sequence of events.
1. `hasCorrectPetAmnt` returns an `Either`.
1. This `Either` is fed to the `map`ped version of `getPets`, which can now accept an `Either`.
1. If the `Either` is a `Left`, exit the composition with that `Left`.
1. If the `Either` is a `Right`, the value is unwrapped and fed as input to `getPets`.
1. The result of `getPets` is then wrapped in a `Right` and returned from the composition.

![chain and map composition](/images/chain-and-map-composition.png)

---

### Further Reading

- <a href="https://gcanti.github.io/fp-ts/" target="_blank">fp-ts - docs</a>
- <a href="https://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html" target="_blank">Functors, Applicatives, and Monads</a>