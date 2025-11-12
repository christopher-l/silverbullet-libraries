#meta/library

This library implements the concept of a daily journal as the primary way to add information, similar to how Logseq works. It is meant to be used with the template [Today](./Today.md).

It implements the following capabilities to support the workflow:

- A scrollable page containing the most recent journal entries.
- Complete excerpts from linked entries on journal pages on every page.
- A list of undone tasks on journal pages.

# Usage

All of the following steps are optional and can be mixed-and-matched as desired.

## Journal

Include the following snippet on any page, e. g., [[Journal]] or your index page:

```
${journal(20)}
```

The given number limits the number of journal pages to be embedded.

## Tasks

Include the following snippet on any page, e. g. your index page:

```
${journalTasks()}
```

## Action buttons

Update your [[CONFIG]] to include buttons for your [[Journal]] page and the "Today" template:

```lua
config.set("actionButtons", {
  {
    icon = "book-open",
    description = "Journal",
    priority = 3,
    run = function()
      editor.navigate "Journal"
    end
  },
  {
    icon = "calendar",
    description = "Today",
    priority = 3,
    run = function()
      editor.invokeCommand "Journal: Today"
    end
  },
  -- ...
})
```

# Implementation

## Journal

Define a function to embed the full pages of the most recent journal entries.

```space-lua
local pageEmbed = template.new([==[
# [[${name}]]

![[${name}]]

]==])

---Embed the `n` most recent journal entries
---@param n integer
function journal(n)
  return template.each(
    query [[
      from index.tag 'page'
      where name:startsWith("Journals/")
      order by _.name desc
      limit n
    ]],
    pageEmbed
  )
end
```

## Linked journal entries

Include a more complete excerpt for links on journal pages than the usual snippet of the "linked mentions" widget on every page.

```space-lua
-- priority: 10
widgets = widgets or {}

local mentionTemplate = template.new [==[
**[[${_.ref}]]**
${_.section}

]==]

---Returns true if the link is within the position bounds of the given tree item.
local function includesLink(item, link)
  return item.from <= link.pos and item.to >= link.pos
end

---Returns the item of "children" that the given link lies in.
local function findChild(children, link)
  for _, item in ipairs(children) do
    if includesLink(item, link) then
      return item
    end
  end
end

---Calls stripTree on all children and returns the resulting array.
local function stripChildren(children, link)
  local strippedChildren = {}
  for i, item in ipairs(children) do
    table.insert(strippedChildren, stripTree(item, link))
  end
  return strippedChildren
end

---Strips a markdown parse tree of all items that are not relevant for the given link.
local function stripTree(tree, link)
  local children = tree.children
  if children == nil then
    return tree
  elseif tree.type == "Document" or tree.type == "BulletList" then
    local child = findChild(children, link)
    if child ~= nil then
      children = { child }
    end
  end
  strippedChildren = stripChildren(children, link)
  return {
    type = tree.type,
    children = strippedChildren,
    from = strippedChildren[0].from,
    to = strippedChildren[#strippedChildren].to
  }
end

---Extracts the section of the page containing the given link that the link is relevant for, i. e. all sub items of the item containing the link and all its ancestor items.
local function extractLinkSection(link)
  local page = space.readPage(link.page)
  local tree = markdown.parseMarkdown(page)
  local strippedTree = stripTree(tree, link)
  return markdown.renderParseTree(strippedTree)
end

function widgets.journalEntries(pageName)
  pageName = pageName or editor.getCurrentPage()
  local journalMentions = query[[
    from index.tag "link"
    where _.page.startsWith("Journals/")
      and _.toPage == pageName
    order by page desc
  ]]
  if #journalMentions > 0 then
    local pinnedEntries = query[[
      from journalMentions
      where _.snippet:match("#pinned")
    ]]
    local otherEntries = query[[
      from journalMentions
      where not _.snippet:match("#pinned")
    ]]
    local markdown = "# Journal Entries\n"
    for _, item in ipairs(pinnedEntries) do
      markdown = markdown .. mentionTemplate({
        ref = item.ref,
        section = extractLinkSection(link)
      })
    end
    for _, item in ipairs(otherEntries) do
      -- TODO: skip entries that are already included in the context of other entries. Take care to show the top-most entry in this case.
      markdown = markdown .. mentionTemplate({
        ref = item.ref,
        section = extractLinkSection(link)
      })
    end
    return widget.new {
      markdown = markdown,
    }
  end
end
```

### Bottom widget

Add the journal-entries widget to every page.

```space-lua
event.listen {
  name = "hooks:renderBottomWidgets",
  run = function(e)
    return widgets.journalEntries()
  end
}
```

### Styling

Override the bottom-widget style to expand instead of scrolling.

```space-style
#sb-main .cm-editor .sb-lua-bottom-widget .content {
  max-height: unset;
}
```

## Linked mentions

Override the built-in "Linked Mentions" widget to exclude journal entries.

```space-lua
-- priority: 2
widgets = widgets or {}

local mentionTemplate = template.new [==[
**[[${_.ref}]]**
> ${_.snippet}

]==]

function widgets.linkedMentions(pageName)
  pageName = pageName or editor.getCurrentPage()
  local linkedMentions = query[[
    from index.tag "link"
    where _.page != pageName
      and not _.page.startsWith("Journals/")
      and _.toPage == pageName
    order by page
  ]]
  if #linkedMentions > 0 then
    return widget.new {
      markdown = "# Linked Mentions\n"
        .. template.each(linkedMentions, mentionTemplate)
    }
  end
end
```

## Tasks

Define a function to display all undone tasks on journal pages.

```space-lua
function journalTasks()
  local tasks = query[[
    from index.tag "task"
    where not _.done and _.page:startsWith("Journals/")
    order by page desc
  ]]
  local md = ""
  if #tasks > 0 then
    return "# Tasks\n"
       .. template.each(tasks, templates.taskItem)
  else
    return ""
  end
end
```
