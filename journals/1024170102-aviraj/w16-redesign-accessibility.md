# Week 16 : Visual Redesign: an Accessibility Pass and a Blank-Page Bug

After the second prototype, every page was moved to a common page layout and
the new design tokens. Two things came out of this work.

## Part 1 : Accessibility Checklist Applied to Every Page

### Problem

An audit of every page found the same problems repeated on many of them:

- no `<h1>`, or more than one, so the page had no clear title;
- search fields and the distance slider without a label;
- a `<button>` nested inside another button or link (invalid HTML, and the
  inner control cannot be reached with the keyboard);
- filter tabs that did not say which one was selected;
- a second "skip to content" link on the listing page.

### Solution

Each fix was made page by page, as a separate small commit, so that any one
could be reverted alone:

``` jsx
{/* exactly one page title */}
<h1>…</h1>

{/* every field has a label tied to its id */}
<label htmlFor="browse-search">Search items</label>
<input id="browse-search" type="search" … />

{/* a filter button says whether it is the selected one */}
<button aria-pressed={filter === 'active'} onClick={() => setFilter('active')}>
  Active
</button>
```

Touch targets in the Create Listing wizard were kept at least 44 px tall on
mobile.

**Because**

A screen reader announces the `<h1>` as the page title, and reads a field by
its label. `aria-pressed` tells it which tab is active, information that
otherwise exists only as a colour.

## Part 2 : Blank Pages Caused by an Undefined `children` in `Input`

### Error

After the redesign, the screens with a dropdown (Create Listing, Create
Request, and the offer form in Chat) went completely blank.

### Relevant Context

The shared `Input` component renders a `<select>` when `type="select"` and
places the caller's `<option>` elements inside it:

``` jsx
export default function Input({ id, type, label, value, onChange, error,
                                className = '', rows, ...rest }) {
  ...
  <select id={fieldId} value={value} onChange={onChange} {...rest}>
    {children}
  </select>
```

### Key Observation

`children` is used but never declared: it is not in the destructured
parameters, so it stays inside `rest`. Rendering a select throws
`ReferenceError: children is not defined`. With no error boundary, React
unmounts the whole tree and leaves an empty page.

### Solution

Destructure `children` together with the other props:

``` jsx
export default function Input({ id, type, label, value, onChange, error,
                                className = '', rows, children, ...rest }) {
```

**Because**

Only the select branch reads `children`, so pages without a dropdown kept
working. That made it look like a styling problem on some pages, not a crash
in one shared component. The browser console showed the `ReferenceError`
straight away.
