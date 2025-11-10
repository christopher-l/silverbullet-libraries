#meta/library

Hides the frontmatter of all pages until clicked.

Via https://community.silverbullet.md/t/hiding-frontmatter/830/7

# Implementation

```space-style
.sb-frontmatter.sb-line-frontmatter-outside:has(+ .sb-frontmatter) ~ .sb-frontmatter {
    display:none
}
```
