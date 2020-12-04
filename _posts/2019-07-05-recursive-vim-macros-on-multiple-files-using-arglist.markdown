---
layout: post
title: 'Recursive vim macros on multiple files using arglist'
comments: true
date: 2019-07-05 11:10:00 +0200
excerpt: 'I use vim macros to implement a repeated code change in a relatively big codebase. I learn of a way to apply macros that exhaust occurrences in one file across multiple files using only vim.'
---

For readers in a hurry:

```bash
qq
# Record your macro and make it exhaust all instances in a file by making it recursive
q

# This populates the argument list
:args `grep -lr 'match lines in the files you are interested in'`
# This runs your macro on each file in the argument list
:argdo norm @q
# This saves all open buffers
:wa
```

---

Yesterday I learned some useful things about vim. I had this:

```typescript
import styles = require('./styles.scss')
import colors = require('./colors.json')
```

And my goal was to turn it into ES6 imports:

```typescript
import styles from './styles.scss'
import colors from './colors.json'
```

Why we used the initial syntax has to do with Webpack and TypeScript, but that's for another post.

```diff
- import styles = require('./styles.scss')
+ import styles from './styles.scss'
- import colors = require('./colors.json')
+ import colors from './colors.json'
```

I thought it wouldn't be too terrible to do it manually with a vim macro, so I went ahead and made one that works on a line. Started recording with `qq`:

```vim
0f=ct(from<Esc>lds(i<Space><Esc>0
```

And stopped recording with `q`. That's very sloppy but it's my daily reality. Here's a breakdown:

- `0`: go to the start of the line.
- `f=`: move the cursor to where the next `=` is.
- `ct(from`: change until the next `(` and insert "from".
- `<Esc>lds(`: change to normal mode, move the cursor to where the next `(` is, and delete the surrounding parentheses.
- `i<Space><Esc>`: insert a space and go back to normal mode.
- `0`: go back to the start of the line. I think this helps me navigating after using a macro.

After the second or third file, I decided to assess the task better. I wanted a list of files with lines matching `import.*= require`, so `grep` seemed like an obvious choice. It could have been The Silver Searcher or ripgrep as well.

```bash
man grep
```

```
  -l, --files-with-matches
    Suppress normal output; instead print the name of each input file from
    which output would normally have been printed. The scanning will stop on
    the first match.

  -r, --recursive
    Read all files under each directory, recursively, following symbolic links
    only if they are on the command line. Note that if no file operand is
    given, grep searches the working directory. This is equivalent to the -d
    recurse option.
```

Perfect!

```bash
grep -lr 'import.*= require' app
```

Around two hundred files used that, and some had three or four matching lines. Some solutions came to mind:

- Use something similar to `codemod`, or `jscodeshift` but for TypeScript.
- Use `sed` in "edit in-place" mode with a regular expression with capture groups.
- Bite the bullet and just go ahead using my macro on every line.

But I was feeling adventurous so I decided to find out if I could apply a macro on all the files that matched. After all, I already had that list of files (the result of the previous `grep`) and I had a macro that worked on one line. This looked like a great learning opportunity.

## Making a macro recursive

This is the idea I tried to follow: first I would search for a matches of my pattern using `/`. Then, I needed to apply the previous macro I created, and move on to the next occurrence from my `/` search until vim returned an error code because there are no more matches, in which case the macro would stop calling itself.

First, I did a search on the buffer:

```vim
/import.*= require<Enter>
```

And then I recorded the same macro I had before on `q`:

```vim
n0f=ct(from<Esc>lds(i<Space><Esc>0@q
```

But with two changes. The new one first finds the next search result with `n`. If this fails because there are no more results, the macro stops. After applying the macro, it calls itself with `@q`.

And that's it! I tried it on one file and was very happy to see my imports modernize for me.

## Using a macro on a list of files

All documentation snippets here are taken from `:help arglist`.

```
The argument list

If you give more than one file name when starting Vim, this list is remembered
as the argument list. You can jump to each file in this list.
```

The documentation had an answer to all my questions:

How can I add things to my argument list?

```
:ar :args
:ar[gs]

Print the argument list, with the current file in square brackets.

:ar[gs] [++opt] [+cmd] {arglist}

Define {arglist} as the new argument list and edit the first one.
```

How can I do things on files in the argument list?

```
:argdo :[range]argdo[!] {cmd}

Execute {cmd} for each file in the argument list or, if [range] is specified,
only for arguments in that range.
```

So that means `args` will return the current argument list and `args {arglist}` will set it. Here we go, then!

```bash
:args `grep -lr 'import.*= require' app`
:args # Huge list of files
:argdo norm @q
:wa # Save all affected buffers
```

It took a while for vim to do its thing. Was it the best way possible? Probably not. Was it a worthwhile experience, and did I learn? Definitely. Am I going to do this again? Yes! Next time I'll be faster and wiser.
