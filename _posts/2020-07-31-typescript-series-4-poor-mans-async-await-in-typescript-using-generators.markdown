---
layout: post
title: "TypeScript - Poor man's async await using generators"
comments: true
categories: [typescript]
date: 2020-07-31 17:37:25 +0200
excerpt: "I implement async await in terms of generators using TypeScript, going a bit deeper into TypeScript's typings for generators."
---

{% include typescript.html %}

> _Other articles on the web explain this idea<sup><a href="#cite-1">[1]</a></sup> <sup><a href="#cite-2">[2]</a></sup>. This is my take on it using TypeScript. I find it interesting because it lets me explore how generator functions are typed. Also, some libraries have implemented their versions of this before `async await` existed. The two I could find are [bluebird](https://github.com/petkaantonov/bluebird/blob/master/src/generators.js#L191)'s `coroutine` and [co](https://github.com/tj/co/blob/master/index.js#L43)._

The **goal** of this post is:

- To see how generator functions can be used to fashion your own `async` and `await` functionality without using the ES2017 keywords.
- To find out whether this implementation can be made type-safe.
- To explain the topic more comprehensively than in the posts that I've linked.

Why, though? Because it's **fun**. Please use a polyfill in your projects!

By the end, we should have something that looks like this:

```typescript
asynq(function* () {
  const a = yield Promise.resolve('a')
  const b = yield Promise.resolve('b')
  const c = yield 'c'

  return [a, b, c]
}).then((array) => {
  expect(array).toEqual(['a', 'b', 'c'])
})
```

Instead of `async` and `await`, the stars of the show are going to be our own function `asynq` and a JavaScript keyword `yield`, specific to generator functions. If generator functions and iterators are new concepts to you, I suggest you read some documentation<sup><a href="#cite-3">[3]</a></sup> <sup><a href="#cite-4">[4]</a></sup> first.

If you'd like to see the code before you read the full post, [here it is on GitHub](https://github.com/fnune/async-await-with-generators/blob/master/async-await-with-generators.ts).

## High-level overview

Our function `asynq` will need to do two things:

1. Initialize the generator that it gets passed.
2. Iterate over the initialized generator in such a way that our `a`, `b` and `c` variables receive the value from the resolved promise, just like it would happen if we could use `await`.

TypeScript types generator functions simply as `() => Generator`, so this is what we're using for the single argument.

```typescript
function asynq(func: () => Generator) {
  const iterator = func()

  // Iterate over `iterator` here.
}
```

Now the question is how to iterate.

## Iterating over the generator

A first impulse could be to use a `for..of` loop because it's made for iterators.

```typescript
function asynq(func: () => Generator) {
  const iterable = func()

  for (const value of iterable) {
    console.log(value) // Can't call `next` on this.
  }
}
```

This loop calls `iterable.next()` with no arguments automatically for us. This is not what we want! We want to pass our own values to `next`. We need fine-grained control over what we pass to each `next` call, and a `for..of` loop would be making that decision for us by passing `undefined` to `next` until the generator is done.

Why do we need fine-grained control over what we pass to each `next` call? I'll steal the example from [this StackOverflow answer](https://stackoverflow.com/questions/37354461/how-does-generator-next-processes-its-parameter):

```
const [variable] = yield [expression]
```

Regardless of what `[expression]` evaluates to, `yield` will assign to `[variable]` the value passed to `next`. This means whatever we pass to `next` will be what the user of `asynq` receives in their variables.

To do this, we can use **recursion**. We want to call our recursive helper until the generator is done. I think `awwait` is a good name for it since awaiting is more or less what it does.

```typescript
function asynq(func: () => Generator) {
  const iterable = func()

  function awwait(result: IteratorResult<unknown>) {
    if (result.done) {
      // This is our base case.
    }

    // We still need to figure out the recursive call.
  }

  return awwait(iterable.next())
}
```

Our `awwait` helper needs to serve three purposes:

- Help us iterate over the generator until `result.done` is `true`.
- Await the promises of each iteration.
- Call the right method on the generator with the result of each promise.

## From `IteratorResult`s to `Promise`s

Generators have two methods that are of interest two us because they map very well to promises:

- `next` can be called whenever a promise resolves successfully.
- `throw` can be called whenever a promise is rejected.

Since we now know `awwait` needs to map from `IteratorResult` to `Promise`, we can easily figure out the base case: the generator is done and we can return a resolved promise.

```typescript
function awwait(result: IteratorResult<unknown>) {
  if (result.done) {
    return Promise.resolve(result.value)
  }
}
```

Our recursive call must now implement the mapping between `IteratorResult` and `Promise` we talked about earlier:

```typescript
function asynq(func: () => Generator) {
  const iterable = func()

  function awwait(result: IteratorResult<unknown>): Promise<unknown> {
    if (result.done) {
      return Promise.resolve(result.value)
    }

    return (
      result.value
        // Things worked out fine, we can call `awwait`.
        .then((value) => awwait(iterable.next(value)))
        // Oops. Calling `throw` will cause an exception,
        // so there's no need to continue the recursion.
        .catch((error) => iterable.throw(error))
    )
  }

  return awwait(iterable.next())
}
```

With this, the example from the beginning of this post almost works. Here's how far we've gotten:

```typescript
asynq(function* () {
  const a = yield Promise.resolve('a')
  const b = yield Promise.resolve('b')

  return [a, b]
}).then((array) => {
  expect(array).toEqual(['a', 'b'])
})
```

Error handling is supported to thanks to our `catch`-to-`throw` mapping:

```typescript
await asynq(function* () {
  try {
    yield Promise.resolve('a')
    yield Promise.reject('b')
  } catch (error) {
    assert.deepStrictEqual(error, 'b')
  }
})
```

## Some improvements

This implementation already works for "`asynq`" functions that `yield` promises. It's cool, but we can do better.

### Supporting non-`Promise` values

`asynq` won't work if a user tries to `yield` a value that's not a promise. We'll add an early return to our `awwait` helper to solve this:

```typescript
if (!(result.value instanceof Promise)) {
  return awwait(iterable.next(result.value))
}
```

Note that this is a naïve way to check whether something is a promise since it will only work for native promises and not for shims, but that is beside the point.

### Supporting type inference

Type inference isn't working at all. In `const a = yield Promise.resolve('a')`, `a` should be `string`, but it's `any`. What's wrong?

Ultimately, what's happening is that there are some limitations on the TypeScript language that we need to sidestep somehow. Support for generator functions is one of the longest-running topics for TypeScript.

So, how can we sidestep this limitation? We need to use type assertion. The `Generator<T, TReturn, TNext>` built-in type takes three type arguments:

- `T` represents the type of the expression that can be `yield`ed, which can be retrieved in `result.value`.
- `TReturn` represents the return type of the generator function, which can also be retrieved in `result.value`.
- `TNext` represents the argument that can be passed to `next`.

We know that our variables `const a` and `const b` receive the value passed to `next` from `asynq`, so this is the type argument that's of interest to us.

```typescript
asynq(function* (): Generator<Promise<string>, string[], string> {
  const a = yield Promise.resolve('a') // a is now `string`
  const b = yield Promise.resolve('b') // b is now `string`

  return [a, b]
})
```

This introduces some difficulties. If `a` were a number, then we would have to pass `string | number` as our type argument, and thus both `a` and `b` would be `string | number`. There's also the elephant in the room: we're asserting these types, so we could pass whatever we wanted and the checker wouldn't complain.

I'm looking forward to being able to update this post with a solution that's fully type-safe. The discussion in the [relevant issue](https://github.com/microsoft/TypeScript/issues/32523) sounds very promising.

---

Related documentation and links:

<ol>
  <li id="cite-1"><a href="https://hackernoon.com/async-await-generators-promises-51f1a6ceede2">Async-Await ≈ Generators + Promises</a></li>
  <li id="cite-2"><a href="https://www.freecodecamp.org/news/how-to-implement-async-and-await-with-generators-11ab0859010f/">Implementing Async And Await With Generators</a></li>
  <li id="cite-3"><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*">MDN web docs for generator functions</a></li>
  <li id="cite-4"><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators">MDN web docs for iterators</a></li>
  <li><a href="https://stackoverflow.com/questions/37354461/how-does-generator-next-processes-its-parameter">How does Generator.next() processes its parameter?</a></li>
  <li><a href="https://github.com/microsoft/TypeScript/issues/32523">Progress on type inference for generator functions</a></li>
  <li><a href="https://github.com/fnune/async-await-with-generators/blob/master/async-await-with-generators.ts">The code in this post on GitHub</a></li>
</ol>
