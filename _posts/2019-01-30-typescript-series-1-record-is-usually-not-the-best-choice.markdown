---
layout: post
title: 'TypeScript - Using Record is usually not the best choice'
comments: true
categories: [typescript]
date: 2019-01-30 11:03:00 +0200
excerpt: 'Using the TypeScript <code>Record</code> type can result in unexpected type unsafety. In this post, I suggest using a <code>Dictionary</code> type that solves this.'
---

{% include typescript.html %}

It's often tempting to define a type for key-value stores using TypeScript's `Record` if you don't know your object's keys during development.

```ts
type Record<K extends string, T> = { [P in K]: T }
```

By using it alongside a type that could be used for an infinite set of values as an argument for `K`, we're promising TypeScript that our object will contain a value of type `T` _for any key_. An object with values for an infinite set of keys does not exist, and by forgetting this we might introduce bugs in our code:

```ts
type Breed = string

interface Dog {
  name: string
}

const dogsByBreed: Record<Breed, Dog[]> = {
  akita: [{ name: 'Walter' }, { name: 'Gracie' }],
  dachshund: [{ name: 'Charlie' }],
}

dogsByBreed.poodle // Type is Dog[], value is undefined.
```

We never told TypeScript that our property access may not return a value, so it will accept the type of the returned value to be `Dog[]`. Use your own type for these situations. Here are example implementations:

```ts
type Dictionary<K extends string, T> = Partial<Record<K, T>>
type Dictionary<K extends string, T> = { [P in K]?: T }

// If you need to support numbers and symbols as keys:
type Dictionary<K extends keyof any, T> = Partial<Record<K, T>>
type Dictionary<K extends keyof any, T> = { [P in K]?: T }

const dogsByBreed: Dictionary<Breed, Dog[]> = {
  akita: [{ name: 'Walter' }, { name: 'Gracie' }],
  dachshund: [{ name: 'Charlie' }],
}

dogsByBreed.poodle // Type is Dog[] | undefined, value is undefined.
```

This way, you'll be forced to handle the `undefined` case.

Ideal use of the `Record` type requires knowledge during development of the specific keys that will be used. You can do this by using a discriminated union type with literals:

```ts
type Breed = 'akita' | 'dachshund'

const dogsByBreed: Record<Breed, Dog[]> = {
  akita: [{ name: 'Walter' }, { name: 'Gracie' }],
  dachshund: [{ name: 'Charlie' }],
}

// Here, the compiler will yield a type error:
// Property 'poodle' does not exist on type 'Record<Breed, Dog[]>'.
dogsByBreed.poodle
```

---

Related documentation:

- [TypeScript handbook entry on advanced types, section on discriminated unions](https://www.typescriptlang.org/docs/handbook/advanced-types.html){:target="\_blank"}
