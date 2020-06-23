---
layout: post
title: 'Thoughts on atomic commits and quality of life'
comments: true
date: 2018-02-19 12:47:51 +0200
excerpt: "In this post, I try to share some of the confirming experiences and observations that have helped me in the process of following git best practices."
---

Reading up on best practices regarding the shape of commits and their messages is something most software developers have done. It's not hard to realize that there's a consensus, and that there's plenty of valid arguments supporting it. This consensus usually boils down to:

- [Correctly formatting commits](https://chris.beams.io/posts/git-commit/): using an editor instead of the `-m` flag, never exceeding the maximum length, using the [imperative](https://en.wikipedia.org/wiki/Grammatical_mood#Imperative) because it's shorter and clearer, using the description, and making sure to maximize the expressiveness and meaning of the message.
- Agreeing on, and adopting a [branch workflow](http://nvie.com/posts/a-successful-git-branching-model/) that suits the nature of the product and the development team. This usually entails making sure the `master` branch always reflects a stable production state, and using a development branch as a staging environment.
- Never [rewriting history](https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History) on the `master` and development branches.
- [Committing often](https://sethrobertson.github.io/GitBestPractices/#sausage_metaphor) and keeping commits small.

Working on a project of medium-to-big size with awesome colleagues has turned blind belief in the aforementioned best practices into defendable reasoning. In this post, I'll try to share some of the confirming experiences and observations that have helped me in the process. Hopefully this will inspire more experienced developers to share their ideas, too.

## Table of contents

- [Git built-ins work better if your commits are thought-through](#git-built-ins-work-better-if-your-commits-are-thought-through)
  - [git bisect](#bisect)
  - [git blame](#blame)
  - [git revert](#revert)
  - [git log](#log)
- [Thinking of how to keep your commits small and clean helps write better code](#thinking-of-how-to-keep-your-commits-small-and-clean-helps-write-better-code)
- [Keeping tests together with their relevant code whenever possible helps](#keeping-tests-together-with-their-relevant-code-whenever-possible-helps)
- [CI and notification set-ups benefit from commits that build and pass tests](#ci-and-notification-set-ups-benefit-from-commits-that-build-and-pass-tests)
- [Peers will not know what your code does, and eventually, neither will you](#peers-will-not-know-what-your-code-does-and-eventually-neither-will-you)
- [You can integrate best practices with a comfortable personal workflow](#you-can-integrate-best-practices-with-a-comfortable-personal-workflow)

---

### Git built-ins work better if your commits are thought-through

There are several Git tools which benefit from a clean and sound commit history. In fact, they are kind of useless _if_ your commit history is not good. The most important are `bisect`, `revert`, `blame` and `log`. I won't go about explaining them - it takes basic search engine usage skills to find explanations much better than those I could dish out myself. But I will try to argue why you can make better use of them if your commits are neatly organized.

<a name='bisect'></a>

```
git bisect
```

The wonderful lifesaver `git bisect` relies on the assumption that every one of your commits is a single logical unit of change. It's not of any help if you "successfully" find a commit with `bisect` which supposedly introduced a bug if the commit actually does several things; even worse if it does anything that's not stated in the commit's message or description.

You might be thinking: Gee! I've never used `bisect` anyway, nor have I ever needed it. Fear not, the day will come, and you _will_ regret not having separated your commits properly.

Or maybe somebody else will, and that would be even worse, because the person bisecting will not be happy about the state of the repository, and maybe you won't even learn about the whole situation.

<a name='blame'></a>

```
git blame
```

This is not particularly about the _whodunnit_ question which `git blame` answers. It's more about the commit messages and descriptions you can see when using it.

Here's an example situation which hopefully better illustrates the case for `git blame`:

1. You want to understand a piece of code that you introduced some time ago, but it was modified. It's a method called `addOilToPasta`, and some evil person (maybe you) seems to have introduced a side effect. Yikes. You fire up `git blame` in hopes of finding a clue.
2. Just like you suspected, you remember your commit message that first added the method. It reads _Implement adding oil on the Pasta component as a class method_. But the three last lines, which add the side effect, were added by another commit.
3. That commit reads _Style the header in the Pasta view_.

Now you have two choices: read the whole commit and find out which part of it actually has anything to do with the faulty code, or go and ask that person directly (hopefully, it's not you). Wouldn't it have been much nicer if there were another commit, separated from the _Style the header in the Pasta view_ one, that read _Implement adding pepper functionality on the Pasta component_? Maybe with a proper description you'd even have all the answers you need already.

<a name='revert'></a>

```
git revert
```

Pretty obvious - if your commits are nicely separated and distinctive, then using `git revert` might be an option. If they aren't, then you'll need to resort to adding a new commit unrelated to the culprit, or rewriting history.

<a name='log'></a>

```
git log
```

This one should be pretty straightforward too. The quality of your commit messages and their structure and separation translates directly to a better log, and happier teammates.

---

### Thinking of how to keep your commits small and clean helps write better code

Just like test-driven development helps with planning ahead and makes you feel certain about what you're doing and what's next, trying to figure out what your pull request is going to look like when it's ready is also beneficial.

Planning your tests and commits before you start to code is a way to avoid tunnel vision, auto-piloting and silly mistakes. Thinking of the logical units of which a feature or bug-fix should be composed is a great first step towards solving a problem.

---

### Keeping tests together with their relevant code whenever possible helps

Although this just goes well with the idea that a commit should be atomic, and that its changes should conform a logical unit.

This, however, introduces a question that's interesting and (as far as I understand) difficult to answer. The case for functionality-related changes and their corresponding tests living together in the same commit is easy to defend. It's easy to say "keep your commits atomic" but you can mold the definition of _atomic_ to your preferences like chewing gum.

For example, say you're working on a task about adding a button to a view that deletes some resource, and confirms deletion with the user using a modal with a _delete_ button and a _cancel_ button. This is a React + Redux application, so to complete this, you need to do several things:

- Add some tests before or after each of the following steps.
- Make sure that the reducer you'll be using responds to the events related to the deletion of that resource. If they're not there, implement them.
- Implement functionality to communicate with your API to actually delete the resource.
- Add the button and possibly style it.

What's the right way to structure these changes into commits? The are many plausible answers, but ultimately, I think you and your team should agree on _one answer_. Depending on how your application is set up, doing all that stuff might be no more than a few lines of code changes. Then maybe it makes sense to keep it all in one commit.

Maybe, if you needed to refactor something to make any of the steps happen, then it makes sense to put that refactor in a different commit.

It's hard to go wrong if you actually _bring up these questions_ with your team and figure out a solution together. Hopefully, by now I've managed to convince you that it's worth dedicating time to this.

---

### CI and notification set-ups benefit from commits that build and pass tests

Checking for a successful build and regressions after every commit by adding appropriate tests helps if you want to check out the project at any particular commit hash or (God forbid) roll back a deployment.

If you've configured, for example, Slack notifications from your CI system or from GitHub, you'll find that their usefulness increases largely if your commits are correctly formatted and expressive. A build failure message that includes commits with their descriptions is only as useful as the messages and descriptions.

---

### Peers will not know what your code does, and eventually, neither will you

Ultimately, you can only write a good message and a good description for your commit if its contents conform a single logical unit. Your commits can become your documentation, and navigating your repository's history can be an extremely powerful tool.

If you haven't learned how to take advantage of some of the more advanced Git tools, maybe somebody else or you, in the future, will. And they will be extremely glad that the commits they're navigating look great and make sense.

---

### You can integrate best practices with a comfortable personal workflow

I usually start writing changes and commit as soon as I find I've done anything significant. I like to keep my first commits as small as possible, even if that doesn't make a lot of sense at the start. This is a safe way to go: it allows me to eventually squash everything into bigger but more meaningful commits without the hassle of resetting. Sometimes, if the original commits I created are good enough, I'll leave it at that.

The point is, your feature branch does not need to come out perfect from the moment of its creation: create a lot of mini-commits at the start. You can package them up properly later with an interactive rebase or by amending. But _do_ think right from the start of how you want the final product to look.

Knowing `git` in and out will obviously help. I've used [gitsh](https://github.com/thoughtbot/gitsh) for a long time but since I [moved from Vim to Emacs](https://fnune.com/2017/12/27/making-emacs-work-like-my-vim-setup/) I've been using [Magit](https://github.com/magit/magit). The thing is just amazing and I've never used `git` so quickly and freely. It was one of my reasons to move to Emacs, too.

---

I hope this has given you new ideas, reminded you of old ones, or maybe at least motivated you to bring up these topics with your team. I'm conscious some of the points I've tried to make are debatable. If you disagree with anything, please leave a comment because I'd love to benefit from the discussion. Thanks for making it until the end of the rant, and see you next time!
