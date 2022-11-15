---
layout: post
title: 'TypeScript - Record and the <code>satisfies</code> operator'
comments: true
categories: [typescript]
date: 2022-11-15 21:16:00 +0200
excerpt: 'TypeScript 4.9 introduces the <code>satisfies</code> operator. It can be used together with the <code>Record</code> type to solve a problem I posted about some years ago.'
---

{% include typescript.html %}

In a previous entry I argued that [the use of the `Record` type is dangerous][previous-post] in some circumstances and suggested using a `Dictionary` type to solve that. The goal of the `Dictionary` type is to make sure the type of values from property access returns `T | undefined` as opposed to just `T` (which would suggest that we have a `T` for any `K` in the dictionary).

[The release of TypeScript 4.9][typescript-4.9] includes a new operator called `satisfies`. It can be used to solve the situation I presented in that other post:

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

Instead of introducing a `Dictionary` type, you can now simply do this:

```ts
const dogsByBreed = {
  akita: [{ name: 'Walter' }, { name: 'Gracie' }],
  dachshund: [{ name: 'Charlie' }],
} satisfies Record<Breed, Dog[]>

dogsByBreed.poodle // Type error!
```

TypeScript gives us a static type-check with the message:

```
Property 'poodle' does not exist on type '{ akita: { name: string; }[]; dachshund: { name: string; }[]; }'.
```

Which is absolutely correct and gives us the best of both worlds:

- We made sure that `dogsByBreed` satisfies our `Record<Breed, Dog[]>` constraint.
- We lost no type information at all about the `dogsByBreed` variable, and TypeScript was able to tell us that `poodle` is not one of its keys.

I'm happy to be able to update the previous post to link to this one. Until next time!

[typescript-4.9]: https://devblogs.microsoft.com/typescript/announcing-typescript-4-9/
[previous-post]: {% post_url 2019-01-30-typescript-series-1-record-is-usually-not-the-best-choice %}

