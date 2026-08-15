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

In the imported node_modules/my-package, it looks for file root.html, and within that file, it looks for xml comments begin and end, and inserts the contents from root.html into the shadowDOM of the my-html-based-web-component tag:

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
