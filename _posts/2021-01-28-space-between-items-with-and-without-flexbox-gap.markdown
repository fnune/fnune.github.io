---
layout: post
title: 'Space between items with and without Flexbox gap'
comments: true
date: 2021-01-28 05:32:51 +0200
excerpt: 'An introduction to the new gap attribute, previously only for Grid layouts, now also available for Flex layouts with almost full support in 2021. Includes a quick look at how developers have worked their way around this before, and probably still have to today.'
---

Adding gaps between items became really easy with the introduction of the `gap` attribute for Grid layouts around 2018. And at the time of writing, `gap` is also supported for Flexbox layouts by [most major browsers, with the notable exception of Safari](https://caniuse.com/flexbox-gap).

As a quick reminder, this is how `gap` works with Grid layouts:

```css
.boxes {
  display: grid;
  gap: 12px;
}
```

<div class="boxes boxes--wrap boxes--flexible-grid">
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
</div>

<aside class="no-support-warning">
  Your browser does not support the <code>gap</code> property in Flexbox layouts. The examples using it wont't make sense. You have been warned. Support checked following ideas from <a style="color: white; font-weight: bold;" href="https://ishadeed.com/article/flexbox-gap/">Ahmad Shadeed</a>.
</aside>

And you'll get the exact same experience with Flexbox layout:

```css
.boxes {
  display: flex;
  gap: 12px;
}
```

<div class="boxes boxes--gap">
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
</div>

The main difference is that you can now combine it with `flex-wrap: wrap`.

<div class="boxes boxes--gap boxes--wrap">
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
</div>

If you need support for Safari (and it's early 2021) or older versions of Chrome and Firefox, you'll have to resort to one of the many workarounds.

### Margins on everything but the last thing

This is easy to get right: every item gets a margin on the right except for the very last one.

```css
.boxes > :not(:last-child) {
  margin-right: 12px;
}
```

<div class="boxes boxes--last-child">
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
</div>

A drawback is that we can't support potential wrapping items, since we would need a conditional bottom margin, and even then, the horizontal margins stop making sense:

<div class="boxes boxes--last-child boxes--wrap">
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
</div>

### The lobotomized owl

I've used this one the most. If I'm being honest, probably because of the memorable name. There are some advantages, though, but those are best explained by [A List Apart](https://alistapart.com/article/axiomatic-css-and-lobotomized-owls/).

```css
.boxes > * + * {
  margin-left: 12px;
}
```

<div class="boxes boxes--lobotomized-owl">
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
</div>

Sadly, the lobotomized owl doesn't handle wrapping either:

<div class="boxes boxes--lobotomized-owl boxes--wrap">
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
</div>

### Hidden negative margins

This one works by wrapping the layout with `overflow: hidden`.

```css
.wrapper {
  overflow: hidden;
}

.boxes > * {
  margin: 12px 0 0 12px;
}

.boxes {
  display: flex;
  flex-wrap: wrap;
  margin: -12px 0 0 -12px;
  width: calc(100% + 12px);
}
```

<div class="boxes-wrapper--negative-margins">
  <div class="boxes boxes--wrap boxes--negative-margins">
    <div class="box"></div>
    <div class="box"></div>
    <div class="box"></div>
    <div class="box"></div>
    <div class="box"></div>
    <div class="box"></div>
    <div class="box"></div>
    <div class="box"></div>
  </div>
</div>

To illustrate why the wrapper is necessary, here it is without it:

<div style="height: 12px;"><!-- Offset the negative margins of the example --></div>
<div class="boxes boxes--wrap boxes--negative-margins">
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
  <div class="box box--bad"></div>
</div>

### Flexible grid

We can achieve a very similar result using a Grid layout:

```css
.boxes {
  display: grid;
  gap: 12px;
  grid-template-columns: repeat(auto-fill, minmax(20%, 1fr));
}
```

<div class="boxes boxes--wrap boxes--flexible-grid">
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
  <div class="box"></div>
</div>

Why only "very similar"? [What can Flexbox do that Grid can't?](https://github.com/rachelandrew/cssgrid-ama/issues/15) Flexbox is content-driven and Grid is layout-driven. A wrapping layout with items of different computed sized isn't possible with Grid. Here's the example from above repurposed to attempt an auto-sized, auto-wrapping layout, but sadly the items all have the same width:

<div class="boxes boxes--flexible-grid boxes--flex-exclusive-grid">
  <div class="box box--bad">Raindrops on roses</div>
  <div class="box box--bad">Whiskers on kittens</div>
  <div class="box box--bad">Bright copper kettles</div>
  <div class="box box--bad">Warm woolen mittens</div>
  <div class="box box--bad">Brown paper packages tied up with strings</div>
  <div class="box box--bad">These are a few of my favorite things</div>
</div>

Here's another attempt with `grid-auto-flow: column` and `white-space: nowrap` which unfortunately no longer features auto-wrapping columns:

<div class="boxes boxes--flexible-grid boxes--flex-exclusive-grid-auto-flow">
  <div class="box box--bad">Raindrops on roses</div>
  <div class="box box--bad">Whiskers on kittens</div>
  <div class="box box--bad">Bright copper kettles</div>
  <div class="box box--bad">Warm woolen mittens</div>
  <div class="box box--bad">Brown paper packages tied up with strings</div>
  <div class="box box--bad">These are a few of my favorite things</div>
</div>

As mentioned in the beginning, this is trivial with `flex-wrap: wrap`.

```css
.boxes {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
}

.box {
  flex: 0 0 auto;
}
```

<div class="boxes boxes--gap boxes--flex-exclusive boxes--wrap">
  <div class="box">Raindrops on roses</div>
  <div class="box">Whiskers on kittens</div>
  <div class="box">Bright copper kettles</div>
  <div class="box">Warm woolen mittens</div>
  <div class="box">Brown paper packages tied up with strings</div>
  <div class="box">These are a few of my favorite things</div>
</div>

If you manage to do this with Grid, please <a href="{{ site.repository_url }}/blob/master/{{ page.path }}">submit a pull-request to edit this post</a>! I'll be happy to merge it and learn something new along the way.

I'm looking forward to use `gap` with Flexbox. Especially once the next major version of Safari is released, which should include [the Webkit changeset that added support for it](https://trac.webkit.org/changeset/267829/webkit).

<style>
.boxes {
  display: flex;
  border-radius: 5px;
  background-color: #e6e6e6;
}

.boxes--gap {
  gap: 12px;
}

.boxes--wrap {
  flex-wrap: wrap;
}

.boxes--flexible-grid {
  display: grid;
  gap: 12px;
  grid-template-columns: repeat(auto-fill, minmax(20%, 1fr));
}

.boxes--flex-exclusive .box,
.boxes--flex-exclusive-grid .box,
.boxes--flex-exclusive-grid-auto-flow .box {
  flex: 0 0 auto;
  padding: 0 12px;
  color: #ffffff;
  font-size: 80%;
  height: auto;
}

.boxes--flex-exclusive-grid-auto-flow {
  overflow-X: scroll;
  grid-template-columns: auto;
  grid-auto-flow: column;
}

.boxes--flex-exclusive-grid-auto-flow .box {
  white-space: nowrap;
}

.boxes--last-child > :not(:last-child) {
  margin-right: 12px;
}

.boxes--last-child.boxes--wrap > :not(:last-child) {
  margin-right: 12px;
  margin-bottom: 12px;
}

.boxes--lobotomized-owl > * + * {
  margin-left: 12px;
}

.boxes--lobotomized-owl.boxes--wrap > * + * {
  margin-left: 12px;
  margin-bottom: 12px;
}

.boxes-wrapper--negative-margins {
  border-radius: 5px;
  overflow: hidden;
}

.boxes--negative-margins {
  margin: -12px 0 0 -12px;
  width: calc(100% + 12px);
}

.boxes--negative-margins > * {
  margin: 12px 0 0 12px;
}

.box {
  flex: 1 1 20%;
  height: 20px;
  border-radius: 5px;
  background-color: #b072d1;
}

.box--bad {
  background-color: #999999;
}

.no-support-warning {
  visibility: hidden;
  position: absolute;
  background-color: #df5273;
  padding: 12px 20px;
  color: #ffffff;
}
</style>

<script type="text/javascript">
// https://ishadeed.com/article/flexbox-gap/
function isFlexGapSupported() {
  // create flex container with row-gap set
  var flex = document.createElement("div");
  flex.style.display = "flex";
  flex.style.flexDirection = "column";
  flex.style.rowGap = "1px";

  // create two, elements inside it
  flex.appendChild(document.createElement("div"));
  flex.appendChild(document.createElement("div"));

  // append to the DOM (needed to obtain scrollHeight)
  document.body.appendChild(flex);
  // flex container should be 1px high from the row-gap
  var isSupported = flex.scrollHeight === 1;
  flex.parentNode.removeChild(flex);

  return isSupported;
}

if (!isFlexGapSupported()) {
  var supportAlert = document.querySelector(".no-support-warning");
  supportAlert.style.visibility = 'visible';
  supportAlert.style.position = 'relative';
}
</script>
