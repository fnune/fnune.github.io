---
layout: post
title: 'First thoughts on Remix'
comments: true
date: 2021-12-24 18:29:51 +0200
excerpt: 'I gave Remix a quick spin following their main tutorial and gathered some notes, focusing on the TypeScript'
---

I gave [Remix](https://remix.run) a quick spin following their "Jokes App" [tutorial][rm-tutorial]
and gathered some notes. These are just my initial thoughts during my first contact with the
framework. For a proper introduction, read [the documentation][rm-documentation].

<figure>
  <blockquote cite="https://remix.run">
    Remix is a seamless server and browser runtime that provides snappy page loads and instant
    transitions by leveraging distributed systems and native browser features instead of clunky static
    builds. Built on the Web Fetch API (instead of Node) it can run anywhere. It already runs natively
    on Cloudflare Workers, and of course supports serverless and traditional Node.js environments, so
    you can come as you are.
  </blockquote>
  <figcaption>From the Remix homepage.</figcaption>
</figure>

The website is the result of a great amount of marketing effort. It's amazing what JS frameworks do
these days to get noticed. Regardless of whether the framework lives up to the hype or not, I get a
strange feeling from the website: the language attempts to be humorous and light, and this doesn't
help me build trust.

From my notes, the most interesting parts are probably on [the `<ScrollRestoration />`
component](#the-scrollrestoration--component) and on [the `loader` pattern](#on-typescript-use).

## On setting up Remix

- Remix nuked my `README.md` file (on which I already had some notes) and I had to recreate it.
  Should have expected that!
- Since Remix picks up on certain exports from files (like `export const meta`) but my IDE doesn't
  know about that, I'm getting a bunch of "unused export" linter errors on those lines. I'm not sure
  where the errors are coming from, since from my recollection `eslint` doesn't complain about
  unused exports. It has no knowledge of the other files. It's probably IntelliJ, then.
- The default `root.tsx` contains several interesting elements:
  - `<Outlet />` which is like a Svelte `<slot />`: it renders children components.
  - `<ScrollRestoration />`, probably aptly-named. Its presence surprises me because this used to be
    a feature of `react-router` that got [dropped][rr-scroll-res] after the developers noticed
    scroll restoration working increasingly better out of the box on major browsers.
    - [x] `TODO`: find
          out [how `<ScrollRestoration />` is implemented](#the-scrollrestoration--component).
    - Remix [documentation on scroll restoration][rm-scroll-res] mentions that this component works
      by restoring the scroll level before rehydration. This should eliminate the jarring effect of
      having the scroll point being restored after the page loads completely.
      - The "before rehydration" part is true only by virtue of the `<ScrollRestoration />`
        component being used one line before the `<Scripts />` component.
- The `yarn build` command finished in 0.44 seconds. Nice!
- I like that the default app doesn't contain images or marketing text selling Remix. It just has
  some documentation links.

### The `<ScrollRestoration />` component

It's implemented [here][rm-scroll-res-impl] and works roughly so:

- It disables the browser implementation with `window.history.scrollRestoration = "manual"`, but
  only in a `useEffect` call that runs after `useLayoutEffect` (used for other effects in the
  component).
- Provides a `<script>` tag that runs a simpler effect (it's simpler because it doesn't need to know
  about browser history, since it runs only on the initial script load before hydration).
- Only after the component has hydrated does the component run some more complex logic (e.g. to
  respect `location.hash`) that ultimately may run `scrollTo(0, position)`.
- Scroll positions are stored in an in-memory `positions` dictionary where the keys
  are `location.key`s.
- To survive for longer, `positions` gets added to `localStorage` and is used to restore `positions`
  into memory when the script runs again.

## On first-contact DX

- Live reload works great and spinning up the development server takes no time.
- While adding some code and missing an import, the app showed me a build error (expectedly). But
  then after fixing the error, the app didn't go back to a normal state, and the error remained,
  even though the build didn't show the error any longer.
- Adding the new routes, I'm thinking that the structure recommended by the tutorial
  where `jokes/new` shares a namespace with `jokes/$jokeId` is not great. What if I wanted slugs
  instead of IDs, and someone created a joke titled `new`? Other solutions aren't as pretty,
  though: `jokes/show/$jokeId + jokes/new`, or `new-joke + jokes/$jokeId`.
- By exporting a `links` array, you can add `<link>` elements to `<head>` sort of like
  with [`react-helmet`][rh]. They then get picked up by a top-level `<Links />` component.
- The tutorial recommends a file structure for styles that's not great: a `styles` directory
  separate from other parts of the app, such as `routes`. If the CSS I write is encapsulated
  together with a route, then why shouldn't their files live together? Having a separate `styles`
  directory adds quite a bit of indirection and could make a project hard to navigate.

## On TypeScript use

- The default `entry.server.tsx` file contains a `handleRequest` function that takes a `request`.
  That's fine, but it also takes a `responseStatusCode` and `responseHeaders`. Am I still able to
  decide what status code to respond with? This signature is a bit weird to me.
  - Looks like there's a default `200` response and I still have a chance to change it, as expected.
    Still, the signature feels awkward.
- I wonder why the tutorial recommends using `export let`. It looks to me as if the things I'm
  exporting shouldn't ever be reassigned. I'm changing these to `export const`, hoping that nothing
  explodes.
  - It looks like in other areas of their documentation they use `export const`, which makes more
    sense.
- The `loader` pattern (see [related docs][rm-loader]) with an `export const loader` coupled with a
  `useLoaderData` call imported from `remix` feels a bit weird to me. I suppose there must be quite
  a bit going on at hydration time for this to be worth the indirection. My first impression is that
  it shouldn't be necessary? Why can't I go `export const loader = someRemixUtility(async () => {})` instead, removing the `useLoaderData` call inside the component? The client and server
  bundles can still provide different implementations of `someRemixUtility`.
  - [x] `TODO`: find out why `export const loader` can't be used directly and needs to be accessed
        via `useLoaderData`. My guess: the purpose is to call the loader during server rendering and
        then to reuse the same data during rehydration, to initialize a frontend cache with. Remix
        developers simply decided to go for a standard way to access Remix functionality, and this just
        looks consistent.
    - It looks like I was correct: Remix [hands off][rm-route-handoff] serialized data to the client
      as a string and then has the client route reuse this data. It works similarly to
      what [Apollo GraphQL recommends][apollo-ssr].
- There's another insidious result of the indirection introduced by the `useLoaderData`
  and `export const loader` pattern, aggravated by the fact that the suggested type `LoaderFunction`
  isn't generic: discrepancies between what `loader` actually returns and what `useLoaderData`
  returns aren't going to be caught unless a type is shared between them. But `LoaderFunction` isn't
  generic, so there's no enforcement from Remix to make sure that this is the case. I can have
  a `loader` that returns `number` but then access `string[]` in `useLoaderData` and unless I
  actively share the types, TypeScript won't have a chance to complain. In my
  opinion, `LoaderFunction` should take a required type argument for the return type, and another (
  perhaps optional?) for query parameters.
  - One could argue that this is the responsibility of the developer, and it is, but by not
    requiring any type arguments, Remix isn't helping.
  - It's also inconsistent because `LoaderFunction` takes no type arguments but `useLoaderData`
    takes a type argument for the returned data.
- When submitting data, the type mismatch is more accentuated, because `POST` requests
  contain `FormData`, which bears (as far as I know) no information on the shape of the data from
  the form. In this case, however, it matters less, because the tutorial calls the user to perform
  backend validation/parsing of this form data, by going `form.get("field-name")` and then
  validating the result.
  - I wonder whether building a way to type-check JSX forms and providing e.g. a TS `eslint` plug-in
    that uses the full power of the AST to build type-safe forms would be worth it. I suppose
    exporting a simple component for use, `TypedForm<T>` would be enough, or perhaps even more
    magic, somehow simply type-check `<form>` elements based on their current children. This is
    probably not possible because one would need to go into other modules. Maybe the `TypedForm<T>`
    approach is good as long as one shares `T` with child controls.

---

That's it for now! I'll probably give Remix a proper try once I have a need for it. I feel like the
typing issues I mentioned can be avoided through discipline, and the DX looks promising.

---

Relevant documentation:

- [Remix documentation][rm-documentation]
- [The `loader` pattern in Remix][rm-loader]
- [React Router documentation on scroll restoration][rr-scroll-res]
- [`useLayoutEffect` documentation](https://reactjs.org/docs/hooks-reference.html#uselayouteffect)

[rm-loader]: https://remix.run/docs/en/v1/api/conventions#loader
[rr-scroll-res]: https://v5.reactrouter.com/web/guides/scroll-restoration
[rm-scroll-res]: https://remix.run/docs/en/v1/api/remix#scrollrestoration
[rm-scroll-res-impl]: https://github.com/remix-run/remix/blob/1fd70960e4d88740df5bf407a6ba2cd2b9549459/packages/remix-react/scroll-restoration.tsx
[rm-route-handoff]: https://github.com/remix-run/remix/blob/1fd70960e4d88740df5bf407a6ba2cd2b9549459/packages/remix-server-runtime/server.ts#L448-L453
[rm-tutorial]: https://remix.run/docs/en/v1/tutorials/jokes
[rm-documentation]: https://remix.run/docs/en/v1
[rh]: https://github.com/nfl/react-helmet
[apollo-ssr]: https://www.apollographql.com/docs/react/performance/server-side-rendering/#executing-queries-with-getdatafromtree
