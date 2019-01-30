---
layout: post
title: 'My experience contributing to Servo'
comments: true
date: 2017-08-12 12:47:51 +0200
---

Some months ago a colleague introduced me to Rust and to the [Servo](https://servo.org) project. It's a web browser engine led by Mozilla, and [its code is available on GitHub](https://github.com/servo/servo) and is open to [contributions](https://github.com/servo/servo/blob/master/CONTRIBUTING.md).

Working on Servo was attractive to me from the start for several reasons:

- It's written in Rust, and Rust has an exciting community, it's low level and modern, it's not shackled in any way by existing legacy code nor by industry requirements, and it just seemed like the perfect thing to help me quench my learning thirst.
- Servo is a web browser engine, and working for a web browser engine as a web developer feels like working on the biggest possible project, on the foundation on which everything I'd ever done took place.
- Servo has an enormous amount of [issues](https://github.com/servo/servo/issues) you're welcome to take. Some of these may seem extremely cryptic and complicated for someone new, but others are extremely trivial. You'll be sure to find the full gradient of difficulties in Servo issues, and there's even [Servo Starters](https://starters.servo.org/), which uses GitHub labels to show issues that are `Good first PR`s.

Let's talk about what actually working on the project feels like.

## Finding an issue

First, you'll need to find an [issue](https://github.com/servo/servo/issues) you find interesting. Helpful labels are `E-easy`, `Good first PR`, and `C-assigned`. You can filter for issues that haven't been assigned to anyone yet by searching with a negative prefix: `-label:C-assigned`. [Here's a good filter](https://github.com/servo/servo/issues?utf8=%E2%9C%93&q=is%3Aissue%20is%3Aopen%20label%3AE-easy%20-label%3AC-assigned) to start with:

```
is:issue is:open label:E-easy -label:C-assigned
```

Do not be discouraged by issues which don't have the `E-easy` label on them. In my experience, an `E-easy` task could end up being a bit more complicated anyway: the line between easy and difficult is blurry and you'll only find out what you're facing once you start working on it.

Also, don't forget [Servo Starters](https://starters.servo.org/).

## Working on your task

Once you've found the issue you want to work on, make sure to leave a comment saying you're working on it so there's no two people implementing the same thing separately.

Compiling Servo is understandably slow: it's a huge project. Your first run will likely take between thirty minutes and one hour depending on your machine and connection, and after some re-runs and testing you'll find yourself with a 15GB directory.

Servo has its own tool, `mach`, which you can use to build for development (`./mach build -d`), for release (`./mach build -r`), and to do many other things. When you submit your pull request, it must pass some CI tests which you can try locally to save time:

- `./mach build -d` for a development build.
- `./mach test-unit` for the unit tests.
- `./mach test-tidy` is the linter.
- `./mach build-geckolib` and `./mach test-stylo`. [Here's more information about those two](https://wiki.mozilla.org/Quantum/Stylo).

If all of those pass, you're almost safe to think it will pass the first CI tests. Check out [the complete `.travis.yml` file](https://github.com/servo/servo/blob/master/.travis.yml) to see the rest. Of course, even if your changes don't pass the tests, you can submit your pull request and expect help from the Servo project members.

## Submitting a pull request

What you need to do to submit a pull request is carefully explained in Servo's [Wiki Page about GitHub workflow](https://github.com/servo/servo/wiki/Github-workflow). Once you submit your pull request, you'll be promptly greeted by [a dog](https://github.com/bors-servo). You won't need to talk to `bors-servo` but Servo organization members can request CI retries by mentioning `@bors-servo`.

## The review process and regression tests

If needed, you're guaranteed to receive extensive help from the project maintainers. In fact, in some cases, I'm quite sure any of the Servo organization members could have solved the problem I was facing with less effort than it took to help me. [Here's some live proof](https://github.com/servo/servo/pulls?utf8=%E2%9C%93&q=is%3Apr%20author%3Abrainlessdeveloper). The help I receive when working on Servo **makes for an invaluable learning opportunity** and it makes contributing to the project all the more enjoyable. Do not be afraid to ask any doubts, and always do your research on the topic: if you study it enough, you'll be able to discuss with others and learn even more.

Once your pull request is all green, you can request review by commenting `r?`. The time until somebody reviews your PR ranges between minutes and a day or two. Somebody will automatically be assigned to the pull request depending on the code area you're working on. Servo organization members can request a full CI run by commenting `@bors-servo try`. This will trigger the rest of the CI suites, and the most important one is the [Web Platform Tests](https://developer.mozilla.org/en-US/docs/Mozilla/QA/web-platform-tests). It's ~~a regression test suite used for Firefox~~ a cross-browser regression test suite run for Firefox and (at least in part) by the Chrome, Edge and Safari teams (thanks for the correction [jgraham](https://disqus.com/by/disqus_Ddtz0h4wcm/)). Many of the tests that run on the suite for Servo come directly from the WPT, but you can also [write your own new tests, modify existing ones or modify the expectations for existing test results](https://github.com/servo/servo/blob/master/tests/wpt/README.md). Many tests are expected to fail for Servo, and you can also submit a pull request to fix those failures. Of [my four pull requests to Servo](https://github.com/servo/servo/blob/master/tests/wpt/README.md), two of them have caused failures on the WPT suite and **most of the work related to the issue went on fixing and improving the tests**.

Running the full test suite takes a long time on CI. Usually around one hour. If a regression test fails live and you're working on a fix, you can always run the test locally to avoid running the full suite online again. First, make a development build with `./mach build -d` and then run the specific test with `./mach test-wpt [path-to-test]`. Unexpected test results, such as `PASS, expected FAIL` will also make the CI suite fail: you'll need to update the test expectations by modifying the corresponding `.ini` file. On [the guide I linked above](https://github.com/servo/servo/blob/master/tests/wpt/README.md) there's all the information you need on how to work with the WPT suite.

## Resources to help you figure out how to solve an issue

Of course, this is highly dependent on what type of issue you picked. But in general, the most important resource is documentation. Reading [the HTML Living Standard](https://html.spec.whatwg.org/) is a great way to help you find the right way to solve a problem. Usually, all the problems are already solved and their solution is in the spec. You're just writing a different explanation of their solutions... in Rust. Quoting Kevlin Henney:

> The act of describing a program in unambiguous detail and the act of programming are one and the same.

You can also find a huge amount of valuable information in [the Mozilla Developer Network](https://developer.mozilla.org/en-US/). Pretty much the same as the spec, but with a lot of examples, and a lot more verbosity. Do not make the same mistake I did and skim through the examples trying to find the exact line that will solve your problem: reading everything thoroughly is what will help you understand the issue best.

If armed with a powerful enough tool, you'll be able to document your way out of your issue by just reading the source code and searching inside of it. I know all editors have project search functionality but this is a _huge_ project. As of commit `b1d7b6bfcf`, the Servo repository has a whopping 6,517,647 lines of code (that's six and a half million). I used [loc](https://github.com/cgag/loc) to count those. So use something fast like [ripgrep](https://github.com/BurntSushi/ripgrep): it'll make your life a lot easier.

In the end, you'll receive the most help from the project maintainers: just ask them. It's an extremely fun process.

If you think I've missed a great resource, please comment below and I'll be sure to include it in this post, and use it on my own as well.

## Conclusion

Working on Servo is one of my main sources of learning nowadays and I'll keep on trying to find issues I can tackle. The tasks I've carried out for now have ranged from [a two line change dependency removal](https://github.com/servo/servo/pull/17811/files) to [properly setting the origin of fetch requests](https://github.com/servo/servo/pull/16508). The fetch API issue took me almost three months to get merged, mostly because of my lack of understanding of the project. But the project maintainers proved to be exceptionally helpful and pleasant to work with. They also never tried to rush a solution and always "followed" (and this should read "accommodated to" or "slowed down to") my pace.

I encourage everyone reading this to check out the project and consider contributing to it. The time you spend working on a project like this is extremely valuable to you as a web developer, and in the end you can feel proud of helping build something on which your applications and websites will probably run in the future.
