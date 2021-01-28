---
layout: post
title: 'Being honest about code coverage'
comments: true
date: 2020-12-04 20:21:50 +0200
excerpt: 'Reframing the question as: how does one achieve high code coverage beneficially?'
---

In my experience, discussing code coverage has always been done with an expectation of a binary conclusion; it's either a good metric to shoot for or it isn't. It seems to me like there's more to be gained if the question is framed differently: how does one achieve high code coverage _beneficially_?

Asking it this way leads to more fruitful argumentation because the following points are no longer part of the question:

- Code coverage is a bad metric because it encourages writing sweeping tests that don't test anything.
- Code coverage sometimes forces developers to test library code.
- Code coverage ultimately makes a code base more difficult to maintain because of all the extra meaningless tests.

To those arguments, my answer is: the problem is not code coverage, but code that's hard to test, a dogmatic approach to unit testing leading to their misuse, or the very fact that one often tries to increase code coverage by writing more tests.

So, as an exercise, let's agree on this for the next couple of paragraphs: code coverage is a good metric as long as you're honest about how to increase it.

What does it mean to be honest about increasing coverage?

- Don't start by trying to test the uncovered lines. Start by questioning the implementation instead, and look for ways to reduce complexity.
- If you're testing library code or glue code, perhaps the implementation deals with mixed-up concerns that you can distinguish and test differently.
- A code base full of meaningless tests is indicative of a deeper problem with the implementation. Try to find that problem instead of writing more meaningless tests.

Code coverage can bear a lot of meaning or very little. If you decide to track it, you and your peers will face having to decide whether to be diligent about it or not, and I'd say that's a good thing.
