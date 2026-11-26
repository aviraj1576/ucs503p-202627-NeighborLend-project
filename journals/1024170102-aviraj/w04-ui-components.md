# Week 4 : Shared UI Components and Design Tokens

## Goal

Decide how every page looks, and build the small set of components that all
pages use, so that screens built by different team members stay consistent.

## Relevant Context

The screen map (Week 3) contains many forms (register, create listing, offers)
and many lists of cards (listings, requests, notifications). Without shared
parts, each page would style its own buttons and fields.

## Key Observation

Most visual differences between pages come from a few repeated decisions:
colours, spacing, corner radius, and the states of a form field (normal,
focused, error, disabled). If these are decided once, a page only has to
arrange components.

## Solution

1. **Design tokens** as CSS custom properties in `src/design/theme.css`.
   Pages refer to the token, never to a raw colour. The current values (after
   the Week 16 redesign) include:

    ``` css
    :root {
      --p-bg: #ffeed6;
      --p-surface: #f3e1cc;
      --p-ink: #2b061e;
      --p-muted: #76575d;
      --p-accent: #d2bf55;
      --p-border: #8e7275;
      --p-danger: #b3261e;
    }
    ```

2. **Five primitives** in `src/components/ui/`: `Button`, `Input`, `Card`,
   `Modal` and `Badge`. `Input` renders a text field, a `<textarea>` or a
   `<select>` depending on `type`, and always places its label and error
   message the same way:

    ``` jsx
    <Input id="title" label="Item name" value={title}
           onChange={(e) => setTitle(e.target.value)} error={errors.title} />
    ```

**Because**

Changing a token restyles every page at once, so the later redesign was
mostly a matter of editing tokens and layouts, not rewriting pages. Keeping
the label and the error inside `Input` means no form can forget to show its
validation errors.
