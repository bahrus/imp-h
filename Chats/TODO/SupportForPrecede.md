# Support for Precede

---

## Human Ask

The root folder of the project we're in has a few alternative HTML Element Enhancement custom attribute libraries.

[pipe-in](/pipe-in/README.md#custom-element-template-handoff) supports the ability of changing

```html
<script type=precede></script>
```

to

```html
<script type=cede></script>
```

Can the same thing be done with [imp-h](/imp-h/README.md)?



## Kiro Response

Yes, the same `precede` → `cede` handoff can be supported by `imp-h`, but the implementation and semantics differ because `imp-h` is intentionally bare-bones compared with `pipe-in`.

### What `imp-h` currently does

1. It finds elements with `imp-h` and fetches the referenced HTML file.
2. It extracts the content between `<!--begin-->` and `<!--end-->` markers.
3. It creates a `<template>` from that content.
4. It clones the template into the element's shadow root.
5. It stores the template in `<head>` and records its id on `element.dataset.impH`.

There is no streaming, no URL rewriting, no sanitizer, and no memoization/sharing. This makes the precede handoff simpler in some ways but also means we must decide exactly what gets handed off.

### How the precede protocol would map onto `imp-h`

The natural equivalent of `pipe-in`'s behavior is:

1. When `imp-h` processes an element whose `localName` includes a dash, check for a child `<script type="precede">`.
2. If found, build the template from the extracted `<!--begin-->`/`<!--end-->` content (the same string already used to populate the shadow root).
3. Attach that template to the script element with `Symbol.for('imp-h:template')` — or reuse `Symbol.for('pipe-in:template')` if we want cross-library consistency.
4. Flip the script's `type` from `precede` to `cede` so that `templ-maker` / `mount-observer` proceeds with custom-element registration.

### Key design choices to resolve

1. **Symbol key.**
   - Option A: `Symbol.for('imp-h:template')` keeps the producer namespace clean.
   - Option B: `Symbol.for('pipe-in:template')` would let the same `templ-maker` logic work unchanged for both libraries.
   - My recommendation is **Option A** (`Symbol.for('imp-h:template')`), because `imp-h` is a distinct package. If `templ-maker` is meant to consume both, it can check for either symbol. That avoids pretending `imp-h` is `pipe-in`.

2. **Where the template content comes from.**
   - Unlike `pipe-in`, `imp-h` already extracts a specific slice via markers, then parses it into a template before injecting it. The cleanest handoff is to use that same template object (or a clone of its `.innerHTML`) rather than re-scraping the shadow root. This matches the rationale in `pipe-in`'s requirements: using the original string avoids complications from later DOM mutations or bindings.

3. **Gate on custom elements.**
   - Same rule as `pipe-in`: only perform the handoff when `enhancedElement.localName.includes('-')`. This prevents unnecessary work on standard elements.

4. **Only the first instance matters.**
   - The `precede` script is only meaningful for the seed element. Subsequent `<hello-world></hello-world>` instances will rely on the custom element definition, not on `imp-h`.

5. **No sharing / memoization needed yet.**
   - `imp-h` currently does not deduplicate fetches. Since there is no shared-stream mechanism, the precede handoff simply runs per element. If `imp-h` later adds caching, the handoff should still run from the cached string/template.

### Proposed implementation outline (imp-h side)

```js
const isCustomElement = element.localName.includes('-');
const precedeScript = isCustomElement
    ? element.querySelector('script[type="precede"]')
    : null;

// ... fetch, extract content, create templ ...

if (precedeScript) {
    Object.defineProperty(precedeScript, Symbol.for('imp-h:template'), {
        value: templ,
        configurable: true
    });
    precedeScript.setAttribute('type', 'cede');
}
```

If we want `templ-maker` to consume it without modification, we could use a conditional getter that lazily clones the template only when accessed:

```js
Object.defineProperty(precedeScript, Symbol.for('imp-h:template'), {
    get() {
        if (!cachedTemplate) {
            cachedTemplate = document.createElement('template');
            cachedTemplate.innerHTML = extractedContent;
        }
        return cachedTemplate;
    },
    configurable: true
});
```

But `imp-h` already has the parsed `templ`, so returning it directly is simpler and avoids a second parse.

### Caveats

- `imp-h` currently has no state attributes (`aria-busy`, `imp-h-state`) and no events. If a `precede` consumer needs to know when the template is ready, it can watch the `type` attribute change, or `imp-h` could dispatch a `load` event to mirror `pipe-in`.
- `imp-h` does not support URL rewriting, so relative URLs in the fetched HTML remain relative to the host document. Any components relying on `pipe-in`'s `-base` behavior would not work the same way here.
- `imp-h` does not run scripts inside the imported HTML; `pipe-in` can via `runScripts`. If the imported HTML contains `<script>` tags that define the custom element, they will not execute with `imp-h`. The expectation is that the `cede` script (via `templ-maker` / `mount-observer`) handles definition.

### Bottom line

The precede/cede protocol ports cleanly to `imp-h` with a small addition: detect the `precede` child script, attach the already-built template under a symbol, and flip the type. The only significant decision is whether to use a library-specific symbol (`imp-h:template`) or share `pipe-in:template`. I lean toward the library-specific symbol for honesty, with `templ-maker` checking both.
