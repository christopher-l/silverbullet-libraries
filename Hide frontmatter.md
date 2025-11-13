#meta/library

Hides the frontmatter of specific pages.

Via https://community.silverbullet.md/t/hiding-frontmatter/830/7

# Usage

Add the following to the frontmatter of a page:

```yaml
pageDecoration:
  cssClasses:
    - no-frontmatter
```

# Implementation

```space-style
.no-frontmatter .sb-frontmatter {
    display: none;
}
```
