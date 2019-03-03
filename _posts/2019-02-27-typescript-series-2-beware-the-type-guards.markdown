---
layout: post
title: 'TypeScript - Beware the user-defined type guards'
comments: true
categories: [typescript]
date: 2019-02-27 18:15:00 +0200
---

{% include typescript.html %}

The `is` keyword in TypeScript is used to specify that a function is a type predicate. Here's [the Wikipedia entry for predicates](<https://en.wikipedia.org/wiki/Predicate_(mathematical_logic)>):

> In mathematical logic, a predicate is commonly understood to be a Boolean-valued function `P`: `X → {true, false}`, called the predicate on `X`. [...] Informally, a predicate is a statement that may be true or false depending on the values of its variables.

A _type_ predicate is a function returning a boolean value that can be used to identify the type of an expression. In TypeScript, type predicates can be expressed using the following syntax:

```ts
function isCollie(dog: Dog): dog is Collie {
  return dog.breed === 'collie' // Must be a Collie
}
```

Type guards can be implemented using the `instanceof` and `typeof` JavaScript operators, but can also be implemented with the `is` keyword like in the example above. Type guards of this nature are _user-defined_ type guards.

While useful, user-defined type guards are a vector for runtime errors in TypeScript applications. It's very easy to implement them in a way that makes a piece of code unsafe. Unsuspecting developers that use them from there on may expand the surface of unsafety. Here's some example code:

```ts
interface Animal {}

interface Bird extends Animal {
  fly: Function
}

interface Dog extends Animal {
  bark: Function
}

function isBird(animal: Animal): animal is Bird {
  return true // If you say so...
}

const norbert: Dog = { bark: () => console.log('Wharg!') }

if (isBird(norbert)) {
  norbert.fly() // Compiles, error during runtime
}
```

Granted, that's a sloppy snippet, but it should serve to illustrate the example. Out in the wild, there are some things you can try to keep in mind in order to avoid this type of problem:

- Implement type guards in a way that leaves out opportunities for runtime failure.
  - Make use of TypeScript's implementations around [discriminated unions](https://www.typescriptlang.org/docs/handbook/advanced-types.html) and the type inference they offer.
  - Use them for simpler cases like filtering out `null` and `undefined`.
- Read the implementation of type guards you use (but haven't implemented) critically.
- Remember that after all, your application is still a JavaScript application.

---

Related documentation and links:

- [TypeScript handbook entry on advanced types, section on type guards](https://www.typescriptlang.org/docs/handbook/advanced-types.html){:target="\_blank"}
- [This article](https://www.matthewgerstman.com/ts-tricks-type-guards/){:target="\_blank"}, as posted on Hacker News, and [its comments](https://news.ycombinator.com/item?id=18975373){:target="\_blank"}
