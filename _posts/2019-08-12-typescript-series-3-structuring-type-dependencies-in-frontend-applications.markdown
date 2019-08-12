---
layout: post
title: 'TypeScript - Structuring type dependencies in frontend applications'
comments: true
categories: [typescript]
date: 2019-08-12 19:55:00 +0200
---

{% include typescript.html %}

In your frontend applications, you may have types that represent a resource that stays the same regardless of what part of the code it's used in. They may be part of your [domain model](https://en.wikipedia.org/wiki/Domain_model), and they can grow pretty fat. For lack of a better word, let's call them **big types**.

In practice, they may come from generated code if using tools like [swagger-codegen](https://github.com/swagger-api/swagger-codegen) or GraphQL code generation utilities like the [Apollo CLI](https://github.com/apollographql/apollo-tooling#apollo-clientcodegen-output) or [GraphQL Code Generator](https://github.com/dotansimha/graphql-code-generator). They may also be hand-written type definition files that represent response types from APIs you use.

In this post I'll try to explain why although tempting, it's a bad idea to use big types across your application code. A simple example could be a tool for finding and comparing hotels. In it, there are many big types such as the `Hotel` type, which contains a ton of information about each hotel. There may also be an interface for a `Room` or a `Booking`.

Let's start with a visual representation of what I'm trying to stop you from doing:

```
├── types
│   └── api.ts
└── views
    ├── booking
    │   ├── Booking.spec.tsx
    │   └── Booking.tsx
    ├── hotel
    │   ├── Hotel.spec.tsx
    │   ├── Hotel.tsx
    │   ├── HotelDescription.tsx
    │   ├── HotelRooms.tsx
    │   └── room
    │       ├── Room.spec.tsx
    │       └── Room.tsx
    └── search
        ├── Search.spec.tsx
        └── Search.tsx
```

In `types/api.ts`, you have complete interfaces for what your API responds with when you request a `Booking`, a `Hotel`, and when you request and interact with a `Booking`. You may also have types like `BookingCreateInput` and `SearchParameter`.

- The `Hotel` component takes a `Hotel`. It renders a wrapper with a title and navigation, and the hotel's description, details and rooms.
- `HotelRooms`, which renders a row for each room in the hotel, takes a `Hotel` and a list of `Room`s.
- `HotelDescription`, which renders a free-text description field and a list of details, takes the whole `Hotel`.
- The `Room` component needs to know about the `Hotel` and the `Room`.
- The `Search` component knows about `Hotel`s, `Room`s and `SearchParameter`s.
- The `Booking` component is even scarier: it knows about the `Booking` type, the `Hotel`, `Room` and `BookingCreateInput` types. It handles displaying open bookings and creating new ones.
- All the `.spec` files mock the entire types for unit tests.

What could go wrong?

### Code ergonomics

When the type changes, it's very likely that you need to change a lot of code to go with that change as well. While the introduction of a new field in a type will likely not require changes in your application code, a breaking change on a big type could start a refactoring chain. The fact that the type is used indiscriminately everywhere makes it hard to see where logic should change.

In the case of generated types, a change in the code generation library will likely result in a situation similar to the one described above as well.

If you unit-test a lot, you're going to need to mock a lot more things in tests for them to compile. This can make your unit tests a lot less clear and remove them from their purpose.

### Proper structure and architecture

You may stop thinking about the design of your components' API and just use the type instead. Using a big type directly instead of thinking of what your component's API should be can lead to code that's harder to maintain, review and fix.

Transformations to the underlying data necessary for a component may be done repeatedly in different components that use the same type as a parameter. This could affect performance, but most importantly, it could make other developers rewrite code that's already somewhere else.

Subtly, using big types directly instead of thinking of a lean API for your components also goes against layering of your applications concerns. If you want to separate the presentation layer from the domain and data layers, using a type that represents your data in the presentation layer is likely to introduce coupling.

## An alternative

In the example above, the `Hotel` is a dependency of six components. The `Room` type is a dependency of four components. In a more realistic application, this number is usually a lot bigger.

A good refactoring goal is to **reduce the number of touchpoints between your big types and your components** while still keeping type safety. There are many ways to do this, but for example:

- Make a dedicated `HotelController` component. It should be the only one that uses the `Hotel` big type directly.
- Do the same for `Room` and `Booking`: `RoomController` and `BookingController`.
- Make those controllers feed their children with exactly what they need, and nothing else.

Another is to **make your components more flexible**:

- Reduce the number of assumptions about the data for each component. For example, instead of making the `HotelDescription` take the entire hotel and then decide what to do with it, have it receive data that's already munched for it.
- Let callers figure out how to serve that munched data. For example, make `HotelDescription` take a `HotelDescription` interface that contains a `description` and a `string[]` list of details. It's very easy to render that.
- Choose prop types that rule out bad inputs, but are as generic as possible. This will help make your component more reusable and easier to test.

Using big types in your presentational components is tempting because it's a quick solution. It also makes logical sense. You want to render a `Hotel` and you have a type called `Hotel`, why not just use that? In addition to that, writing more types by hand to represent your interfaces may seem like a waste of time if the "real" type is already somewhere in your code. Hopefully, this post helps convince you otherwise.

Related:

- [Martin Fowler on presentation domain data layering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html){:target="\_blank"}
- [Eric Evan's Domain-Driven Design](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215){:target="\_blank"}
- [TypeScript documentation on type compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility.html){:target="\_blank"}
