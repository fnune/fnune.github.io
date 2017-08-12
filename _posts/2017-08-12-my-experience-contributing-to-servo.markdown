---
layout: post
title:  "My experience contributing to Servo"
comments: true
published: false
date:   2017-08-12 12:47:51 +0200
---

Some months ago a colleague introduced me to Rust and to the [Servo](https://servo.org) project. It's a web browser engine led by Mozilla, and [its code is available on GitHub](https://github.com/servo/servo) and is open to [contributions](https://github.com/servo/servo/blob/master/CONTRIBUTING.md).

Working on Servo was attractive to me from the start for several reasons:

* It's written in Rust, and Rust has an exciting community, it's low level and modern, it's not shackled in any way by existing legacy code nor by industry requirements, and it just seemed like the perfect thing to help me quench my learning thirst.
* Servo is a web browser engine, and working for a web browser engine as a web developer feels like working on the biggest possible project, on the foundation on which everything I'd ever done took place.
* Servo has an enormous amount of [issues](https://github.com/servo/servo/issues) you're welcome to take. Some of these may seem extremely cryptic and complicated for someone new, but others are extremely trivial. You'll be sure to find the full gradient of difficulties in Servo issues, and there's even [Servo Starters](https://starters.servo.org/), which uses GitHub labels to show issues that are _Good first PR_s.

Let's talk about what actually working on the project feels like.

## Workflow

First, you'll need to find an [issue](https://github.com/servo/servo/issues) you find interesting. Helpful labels are `E-easy`, `Good first PR`, and `C-assigned`. You can filter for issues that haven't been assigned to anyone yet by searching with a negative prefix: `-label:C-assigned`. Here's a good filter to start with:

```
is:issue is:open label:E-easy -label:C-assigned
```

Do not be discouraged by issues which don't have the `E-easy` label on them. It's likely that your `E-easy` task ends up being a bit more complicated anyway, so obviously the line between easy and difficult is blurry.

What you need to do to submit a pull request to solve one of those issues is carefully explained in Servo's [Wiki Page about GitHub workflow](https://github.com/servo/servo/wiki/Github-workflow). Once you submit your pull request, you'll be promptly greeted by [a dog](https://github.com/bors-servo). You won't need to talk to `bors-servo` but Servo organization members can request CI retries by mentioning `@bors-servo`.

## TODO

Coming from dynamic languages (mainly JavaScript), I had an uninformed opinion that type systems were a burden more than a liberation and would only make sense in huge projects.

