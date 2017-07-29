---
layout: post
title:  "Why won't my text overflow? Where's my ellipsis!?"
comments: true
date:   2017-07-29 12:47:51 +0200
---

The `text-overflow` property is a PITA to deal with because on its own, it **won't force text to overflow**. Say what? Its name IS _text overflow_. Anyway, what it actually does is to define the behavior of a text node _if_ it overflows. The job of actually making it overflow is yours and only yours.

I present to you a **checklist** that should once and for all truncate that stubborn `span` and print some beautiful [ellipses](https://en.wikipedia.org/wiki/Ellipsis) at the end of your one-liners.

In the checklist, I'll be using the word "container". So let's define it first, just for this post: the "container" I'm talking about is the _immmediate_ parent of the element that contains the text node you want to truncate. For example:

```html
<div> <!-- This div is the container -->
  <span>I will NOT truncate. Nononononono, no!</span> <!-- Let's call this one "stubborn child" -->
</div>
```

Is your container NOT a flex container? [Checklist A](#checklist-a-for-non-flex-containers) is your friend.
Is your container a flex container? [Checklist B](#checklist-b-for-flex-containers) is for you.

## Checklist A: for non-flex containers

1. Is the container a [block-level element](https://developer.mozilla.org/en-US/docs/Web/HTML/Block-level_elements)?
2. Does the container have a [computed size](https://developer.mozilla.org/en-US/docs/Web/API/CSS_Object_Model/Determining_the_dimensions_of_elements) determined by your CSS, or does it inherit one?
3. Did you add the `white-space: nowrap;` property to the **container**?
4. Did you add the `overflow: hidden;` property to the **container**?
5. Did you add the `text-overflow: ellipsis;` property to the **container**? (duh)

## Checklist B: for flex containers

1. Does the container have a [computed size](https://developer.mozilla.org/en-US/docs/Web/API/CSS_Object_Model/Determining_the_dimensions_of_elements) determined by your CSS, or does it inherit one?
2. Does the child also have a [computed size](https://developer.mozilla.org/en-US/docs/Web/API/CSS_Object_Model/Determining_the_dimensions_of_elements) determined by your CSS, or does it inherit one?
3. Did you add the `white-space: nowrap;` property to the **child**?
4. Did you add the `overflow: hidden;` property to the **child**?
5. Did you add the `text-overflow: ellipsis;` property to the **child**? (duh)

Hope that helped. Here's some nice [documentation from MDN about the text-overflow property](https://developer.mozilla.org/en-US/docs/Web/CSS/text-overflow) which is well worth the read.
