# imp-h

For small, html based web components, the benefits of streaming and ShadowDOM support of [pipe-in](https://github.com/bahrus/pipe-in) is outweighed by all the dependencies / code needed.  But for declarative web components, the delay in just rendering the unhydrated html makes that delay not worth it.

So this is a bare bones version of be-importing.

## Usage

```html
<html>
    <head>
        <script type=importmap>
            {
                "imports": {
                    "my-package/": "/node_modules/my-package/"
                }
            }           
        </script>
    </head>
    <body>
        <my-html-based-web-component imp-h="my-package/root.html"></my-html-based-web-component>


        <script type=module>
            import './imp-h.js';
        </script>
    </body>
</html>
```

In the imported `node_modules/my-package`, it looks for file `root.html`, and within that file it looks for `<?start>` and `<?end>` markers and inserts the contents from `root.html` into the shadow DOM of the `my-html-based-web-component` tag:

```html
<html>
    <head>
    </head>
    <body>
        <?start>
        <div>my content</div>
        <?end>
    </body>
</html>
```


## Custom element template handoff

When `imp-h` is used on a custom element (any tag with a dash in its name) that contains a `<script type="precede">` child, `imp-h` coordinates with custom element features such as [templ-maker](https://github.com/bahrus/templ-maker) to provide a template for the element's definition.

```html
<hello-world imp-h="my-package/hello-world.html">
    <script type="precede" data-extends="el-maker"></script>
</hello-world>
```

After fetching and extracting the content between `<?start>` and `<?end>`, `imp-h`:

1. Builds a `<template>` from the extracted HTML.
2. Attaches it to the script element via `Symbol.for('imp-h:template')`.
3. Flips the script's `type` from `precede` to `cede`, triggering `mount-observer` or equivalent logic to proceed with custom element registration.

This is the same coordination protocol used by [pipe-in](/pipe-in/README.md#custom-element-template-handoff), but `imp-h` uses the simpler `<?start>` / `<?end>` markers and does not stream, rewrite URLs, or deduplicate fetches.
