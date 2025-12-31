#meta/library

Eagerly break links that are the first text after a bullet point. This avoids links moving to a new line, leaving an empty line after the bullet-point indicator.

```space-style
li > span.p:first-child > a:first-child {
    word-break: break-all;
}
```

Eagerly break links in tasks.

```space-style
.sb-line-task > .sb-naked-url,
.sb-line-task > .cm-task-checked > .sb-naked-url
{
    word-break: break-all;
}
```
