CSS Layout API Explained
========================

The CSS Layout API is being developed to improve the extensibility of CSS.

Specifically the API is designed to give web developers the ability to write their own layout
algorithms in addition to the native algorithms user agents ship with today.

For example user agents today currently ship with:
 - Block Flow Layout
 - Flexbox Layout

However with the CSS Layout API web developers could write their own layouts which implement:
 - Constraint based layouts
 - Masonry layouts
 - Line spacing and snapping

Initial Concepts - Writing Modes
--------------------------------

This API uses terminology which may be foreign to many web developers initially. Everything in the
CSS Layout API is computed in the [logical coordinate
system](https://drafts.csswg.org/css-writing-modes-3/#text-flow).

This has the primary advantage that when you write your layout using this system it will
automatically work for writing modes which are right-to-left (e.g. Arabic or Hebrew), or for writing
modes which are vertical (many Asian scripts including Chinese scripts, Japanese and Korean).

For a developer who is used to left-to-right text, the way to translate this back into "physical"
coordinates is:

| Logical     | Physical |
| -----------:|:-------- |
| inlineSize  | width    |
| inlineStart | left     |
| inlineEnd   | right    |
| blockSize   | height   |
| blockStart  | top      |
| blockEnd    | bottom   |

Getting Started
---------------

First you'll need to import a script into the layout worklet.

```js
if ('layoutWorklet' in CSS) {
  CSS.layoutWorklet.import('my-layout-script.js').then(() => {
    console.log('layout script installed!');
  });
}
```

See the worklets [explainer](../worklets/EXPLAINER.md) for a more involved explanation of worklets.

After the promise returned from the `import` method resolves the layouts defined in the script will
apply to the page.

A Centering Layout
------------------

The global script context for the layout worklet has exactly one entry method exposed to developers:
`registerLayout`.

```js
registerLayout('centering', class extends Layout {
  static blockifyChildren = true;

  static inputProperties = super.inputProperties;

  *layout(constraintSpace, children, styleMap) {
    // (1) Resolve our inline size (typically 'width').
    const inlineSize = resolveInlineSize(constraintSpace, styleMap);

    // (2) Determine our (inner) available size.
    const bordersAndPadding = resolveBordersAndPadding(constraintSpace, styleMap);
    const scrollbarSize = resolveScrollbarSize(constraintSpace, styleMap);
    const availableInlineSize = inlineSize -
                                bordersAndPadding.inlineStart -
                                bordersAndPadding.inlineEnd -
                                scrollbarSize.inline;

    const availableBlockSize = resolveBlockSize(constraintSpace, styleMap) -
                               bordersAndPadding.blockStart -
                               bordersAndPadding.blockEnd -
                               scrollbarSize.block;

    // (3) Loop over each child and perform layout.
    const childFragments = [];
    const childConstraintSpace = new ConstraintSpace({
      inlineSize: availableInlineSize,
      blockSize: availableBlockSize
    });
    let maxChildInlineSize = 0;
    let maxChildBlockSize = 0;
    for (let child of children) {
      const childFragment = yield child.layout(childConstraintSpace);

      maxChildInlineSize = Math.max(maxChildInlineSize, childFragments.inlineSize);
      maxChildBlockSize = Math.max(maxChildBlockSize, childFragments.blockSize);
      childFragments.push(childFragment);
    }

    // (4) Determine our block size.
    const blockOverflowSize = maxChildBlockSize +
                              bordersAndPadding.blockStart +
                              bordersAndPadding.blockEnd;
    const blockSize = resolveBlockSize(constraintSpace, styleMap, blockOverflowSize);

    // (5) Loop over each fragment to position in the center.
    for (let fragment of childFragments) {
      fragment.inlineOffset = bordersAndPadding.inlineStart +
                              (inlineSize - fragment.inlineSize) / 2;
      fragment.blockOffset = bordersAndPadding.blockStart + 
                             Math.max(0, (blockSize - fragment.blockSize) / 2);
    }

    // (6) Return our fragment.
    return {
      inlineSize: inlineSize,
      blockSize: blockSize,
      inlineOverflowSize: maxChildInlineSize,
      blockOverflowSize: blockOverflowSize,
      childFragments: childFragments,
    }
  }
});
```

There are a lot of things going on in this example so lets step through them one-by-one.

The `layout` function is your callback into the browsers layout phase in the
rendering engine. You are given:
 - `constraintSpace`, the space of constraints which the fragment you produce should meet.
 - `children`, the list of children boxes you should perform layout upon.
 - `styleMap`, the style for the current layout.

Layout eventually will return a dictionary will what the resulting fragment of
that layout should be.

The above example would be used in CSS by:
```css
.centering {
  display: layout(centering);
}
```

### Step `(1)` - Resolving the Inline Size ###

The first thing that you'll want to do for most layouts is to determine your `inlineSize` (`width`
for left-to-right modes). If you want to use the default algorithm you can simply call:
```js
const inlineSize = resolveInlineSize(constraintSpace, styleMap);
```

This will compute the `inlineSize` using the standard CSS rules. E.g. for:
```css
.parent {
  width: 100px;
}

.layout {
  display: layout(centering);
  writing-mode: horizontal-tb;
  width: 80%;
}
```

`resolveInlineSize` will resolve the `inlineSize` of `.layout` to `80px`. See
[developer.mozilla.org](https://developer.mozilla.org)
for an explaination of what [width](https://developer.mozilla.org/en-US/docs/Web/CSS/width) and
[height](https://developer.mozilla.org/en-US/docs/Web/CSS/height), etc will resolve to.

### Step `(2)` - Resoliving the "inner" Available Size ###

CSS typically subtracts the border and padding of the current fragment from the available space
provided to the children fragments. Step `(2)` does this.

### Step `(3)` - Perform Layout on `children` ###

After we have the our `inlineSize` we can now perform layout on our children.

The first step is to create a constraint space for our child. E.g.

```js
const childConstraintSpace = new ConstraintSpace({
  inlineSize: availableInlineSize,
  blockSize: availableBlockSize
});
```

There are more options for the constraint space than used here, but for this simple example the
child constraint space just sets the size available to the children.

We now loop through all of our children and perform layout. This is done by:

```js
const childFragment = yield child.layout(childConstraintSpace);
```

`child` has a very simple API. You can query the style of a child and perform layout - e.g.

```js
child instanceof LayoutChild; // true
child.styleMap.get('--a-property');

const fragment = yield child.layout(childConstraintSpace);
```

The result of performing layout on a child is a `Fragment`. A fragment is read-only except for
setting the offset relative to the parent fragment.

```js

fragment instanceof Fragment; // true

// The resolved size of the fragment.
fragment.inlineSize;
fragment.blockSize;

// The dominant baseline of the fragment.
fragment.dominantBaseline;

// We can set the offset relative to the current layout.
frgment.inlineOffset = 10;
frgment.blockOffset = 20;
```

In step `(3)` we do some additional book-keeping to keep track of the largest child fragment so far.

### Step `(4)` - Resolving the Block Size ###

The now that we know how large our biggest child is going to be, we can calculate our actual
`blockSize`.

We perform a call to `resolveBlockSize` again, except this time we also pass in the size that we
would be if the height is `auto`.

`resolveBlockSize` will also apply rules like `max-height` etc. upon the result. E.g. for:
```css
.layout {
  display: layout(centering);
  writing-mode: horizontal-tb;
  max-height: 200px;
}
```

If we called:
 - `resolveBlockSize(constraintSpace, styleMap, 400)` the result would be `200`.
 - `resolveBlockSize(constraintSpace, styleMap, 180)` the result would be `180`.

### Step `(5)` - Positioning our Children Fragments ###

In this example layout we are centering all of our children. This step sets the offset of the child
relative to the parent fragment. E.g.
```js
fragment.inlineOffset = 20;
fragment.blockOffset = 40;
```

The above would place `fragment` at an offset `(20, 40)` relative to its parent.

### Step `(6)` - Returning our Fragment ###

Finally we return a dictionary which represents the `Fragment` we wish the rendering engine to
create for us. E.g.
```js
const result = {
  inlineSize: inlineSize,
  blockSize: blockSize,
  inlineOverflowSize: maxChildInlineSize,
  blockOverflowSize: blockOverflowSize,
  childFragments: childFragments,
};
```

The important things to note here are that you need to explicitly say which `childFragments` you
would like to render. If you give this an empty array you won't render any of your children.

Text Layout
-----------

We used a little trick in the above example:
```js
static blockifyChildren = true;
```

This one line forces all of the children to be blockified in your layout. This means for example if
you have:
```html
<div class="layout">
  I am some text
  <div class="child"></div>
</div>
```

The engine will force the text `I am some text` to be surrounded by a `<div>`. E.g.
```html
<div class="layout">
  <div>I am some text</div>
  <div class="child"></div>
</div>
```

This is important as the above `centering` layout would have to deal with text _fragmentation_, a
few native layouts use this trick to simplify their algorithms, for example grid and flexbox.

### Text Fragmentation ###

In the above `centering` example, we forced each `LayoutChild` to produce exactly one `Fragment`.

However each `LayoutChild` are able to produce more than one `Fragment`. For example:

```text
|---- Inline Available Size ----|
The quick brown fox jumped over the lazy dog.
```

```js
child instanceof LayoutChild;

const fragment1 = yield child.layout(constraintSpace);
const fragment2 = yield child.layout(constraintSpace, fragment1.breakToken);

fragment2.breakToken == null;
```

In the above example the text child produces two fragments. Containing:
1) `The quick brown fox jumped over`
2) `the lazy dog.

The critical detail here to be aware of is the concept of a `BreakToken`. The `BreakToken` contains
all of the information necessary to continue/resume the layout where we the child finished.

We pass the `BreakToken` to add back into the `layout()` call in order to produce the next fragment.

### A Basic Text Layout ###

```js
registerLayout('basic-inline', class extends Layout {
  static inputProperties = super.inputProperties;

  *layout(constraintSpace, children, styleMap) {
    // Resolve our inline size.
    const inlineSize = resolveInlineSize(constraintSpace, styleMap);

    // Determine our (inner) available size.
    const bordersAndPadding = resolveBordersAndPadding(constraintSpace, styleMap);
    const scrollbarSize = resolveScrollbarSize(constraintSpace, styleMap);
    const availableInlineSize = inlineSize -
                                bordersAndPadding.inlineStart -
                                bordersAndPadding.inlineEnd -
                                scrollbarSize.inline;

    const availableBlockSize = resolveBlockSize(constraintSpace, styleMap) -
                               bordersAndPadding.blockStart -
                               bordersAndPadding.blockEnd -
                               scrollbarSize.block;

    const childFragments = [];
    let maxInlineSize = 0;

    let currentLine = [];
    let usedInlineSize = 0;
    let maxBaseline = 0;

    let lineOffset = 0;
    let maxLineBlockSize = 0;

    // Just a small little function which will update the above variables.
    const nextLine = function() {
      if (usedInlineSize > maxInlineSize) {
        maxInlineSize = usedInlineSize;
      }

      currentLine = [];
      usedInlineSize = 0;
      maxBaseline = 0;

      lineOffset += maxLineBlockSize;
      maxLineBlockSize = 0;
    }

    let child = children.shift();
    let breakToken = null;
    while (child) {
      // Make sure we actually have space on the current line.
      if (usedInlineSize > availableInlineSize) {
        nextLine();
      }

      // The constraint space here will have the inline size of the remaining
      // space on the line.
      const remainingInlineSize = availableInlineSize - usedInlineSize;
      const constraintSpace = new ConstraintSpace({
        inlineSize: availableInlineSize - usedInlineSize,
        blockSize: availableBlockSize,
        percentageInlineSize: availableInlineSize
      });

      const fragment = yield child.layout(constraintSpace, breakToken);
      childFragments.push(fragment);

      // Check if there is still space on the current line.
      if (fragment.inlineSize > remainingInlineSize) {
        nextLine();
      }

      // Insert fragment on the current line.
      currentLine.push(fragment);
      fragment.inlineOffset = usedInlineSize;

      if (fragment.dominantBaseline > maxBaseline) {
        maxBaseline = fragment.dominantBaseline;
      }

      // Go through each of the fragments on the line and update their block
      // offsets.
      for (let fragmentOnLine of currentLine) {
        fragmentOnLine.blockOffset =
          lineOffset + maxBaseline - fragmentOnLine.dominantBaseline;

        const lineBlockSize =
          fragmentOnLine.blockOffset + fragmentOnLine.blockSize;
        if (maxLineBlockSize < lineBlockSize) {
          maxLineBlockSize = lineBlockSize;
        }
      }

      if (fragment.breakToken) {
        breakToken = fragment.breakToken;
      } else {
        // If a fragment doesn't have a break token, we move onto the next
        // child.
        child = children.shift();
      }
    }

    // Determine our block size.
    nextLine();
    const blockOverflowSize = lineOffset +
                              bordersAndPadding.blockStart +
                              bordersAndPadding.blockEnd;
    const blockSize = resolveBlockSize(constraintSpace, styleMap, blockOverflowSize);

    // Return our fragment.
    return {
      inlineSize: inlineSize,
      blockSize: blockSize,
      inlineOverflowSize: maxInlineSize,
      blockOverflowSize: blockOverflowSize,
      childFragments: childFragments,
    }
  }
});
```

The above example is more complex than the previous centering layout because of the ability for text
children to fragment. This example positions all of its children on a line if there is space, if not
it moves to the next.

That said it has all the same steps as before:
 1) Resolving the `inlineSize`.
 2) Resolving the (inner) available size.
 3) Performing layout and positioning children fragments.
 4) Resolving the final block size.
 5) Returning the fragment.

Scrolling
---------

We have been handling scrolling in the above example but we haven't talked about it yet.

```js
const scrollbarSize = resolveScrollbarSize(constraintSpace, styleMap);
scrollbarSize.inline;
scrollbarSize.block;
```

The above code snippet queries the size of the scrollbar. `resolveScrollbarSize` handles the CSS
`overflow` property. For example if we are `overflow: hidden`, `resolveScrollbarSize` will
report 0 for both directions.

The overflow size is reported when returning the new fragment, e.g.
```js
return {
  inlineSize: 200,
  blockSize: 300,
  inlineOverflowSize: 400,
  blockOverflowSize: 600
};
```
In the above example the scrollable area is double the actual fragments size.

### `overflow: auto` Behaviour ###

One of the complexities with scrollable fragments is `auto`. In the current CSS Layout API your
layout may get called up to three times.
 1) 
 2)
 3)

TODO

Block Fragmentation
-------------------

Some native layouts on the web support what is known as block fragmentation. For example:

```html
<style>
.multicol {
  columns: 3;
}
</style>
<div class="multicol">
This is some text.
<table>
<!-- SNIP! -->
</table>
This is some more text.
</div>
```

In the above example the `multicol` div may produce three (3) fragments.
 1) {fragment}This is some text.{/fragment}
 2) {fragment}{fragment type=table}{/fragment} This is{/fragment}
 3) {fragment}some more text.{/fragment}

We can replicate this behaviour with the CSS Layout API by creating our own break tokens. E.g.

```js
registerLayout('basic-inline', class extends Layout {
  static inputProperties = super.inputProperties;

  *layout(constraintSpace, children, styleMap, breakToken) {

    // We can get the start child to start layout at with the breakToken. E.g.
    let child = null;
    let childToken = null;
    if (breakToken) {
      childToken = breakToken.childTokens[0]; // We can actually have multiple
                                              // children break. But for now
                                              // we'll just use one.
      child = childToken.child;
    } else {
      child = children;
    }

    // SNIP!

    return {
      inlineSize: inlineSize,
      blockSize: blockSize,
      breakToken: {
        data: /* you can place arbitary data here */,
        childTokens: [childToken]
      }
    }
  }
});
```

There are two important parts the above code example.

TODO.

Closing Words
-------------

Yes this is a complex API, yes it uses foreign terminology. But we really want to give you, the web
developer, the power that the rendering engines have when it comes to layout. Enjoy! :)

