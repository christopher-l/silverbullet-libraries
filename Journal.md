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

---Returns the position part of a reference.
---@param ref string
---@return integer
local function refPos(ref)
  local posString = ref:match("@(%d+)$")
  return tonumber(posString)
end

---@param line string
local function isItem(line)
  if line:match("^%s*[%-%*]%s") ~= nil then
    return true
  elseif line:match("^%s*%d%.%s") ~= nil then
    return true
  else
    return false
  end
end

---@param line string
local function isEmpty(line)
  return string.len(string.trim(line)) == 0
end

---@param line string
---@return integer
local function indentLevel(line)
  match = line:match("^(%s*)")
  return match:len()
end

---Extracts an item with all ancestor and all sub items.
---Expects refLineIndex to indicate a line with an item.
---@param lines { line: string, startIndex: integer }[]
---@param refLineIndex integer
---@return { line: string, startIndex: integer }[]
local function extractItem(lines, refLineIndex)
  local idx = refLineIndex
  local level = indentLevel(lines[idx].line)
  -- Include referenced item.
  local extractedLines = { lines[refLineIndex] }
  -- Include ancestor items.
  while level > 0 and idx > 1 do
    idx = idx - 1
    if isEmpty(lines[idx].line) then
      -- continue
    elseif isItem(lines[idx].line)
      and indentLevel(lines[idx].line) < level then
      level = indentLevel(lines[idx].line)
      table.insert(extractedLines, 1, lines[idx])
    elseif not isItem(lines[idx].line)
      and indentLevel(lines[idx].line) == 0 then
      break
    end
  end
  -- Include sub items.
  idx = refLineIndex
  level = indentLevel(lines[idx].line)
  while idx < #lines do
    idx = idx + 1
    if isEmpty(lines[idx].line) then
      -- continue
    elseif indentLevel(lines[idx].line) > level then
      table.insert(extractedLines, lines[idx])
    else
      break
    end
  end
  return extractedLines
end

---Extracts a paragraph from and to an empty line.
---Expects refLineIndex to indicate a line within a paragraph.
---@param lines { line: string, startIndex: integer }[]
---@param refLineIndex integer
---@return { line: string, startIndex: integer }[]
local function extractParagraph(lines, refLineIndex)
  local idx = refLineIndex
  -- Include referenced line.
  local extractedLines = { lines[refLineIndex] }
  -- Include lines above.
  while idx > 1 do
    idx = idx - 1
    if isEmpty(lines[idx].line) then
      break
    elseif isItem(lines[idx].line) then
      break
    else
      table.insert(extractedLines, 1, lines[idx])
    end
  end
  -- Include lines below.
  idx = refLineIndex
  while idx < #lines do
    idx = idx + 1
    if isEmpty(lines[idx].line) then
      break
    elseif isItem(lines[idx].line) then
      break
    else
      table.insert(extractedLines, lines[idx])
    end
  end
  return extractedLines
end

---@param item { ref: string, page: string }
---@return { line: string, startIndex: integer }[]
local function extractLines(item)
  local pos = refPos(item.ref)
  local page = space.readPage(item.page)
  local lines = {}
  local lineChars = 0
  local refLineIndex = -1
  for line in page:gmatch("([^\n]*)\n?") do
    table.insert(lines, {
      line = line,
      startIndex = lineChars,
    })
    lineChars = lineChars + string.len(line) + 1
    if refLineIndex < 0 and lineChars > pos then
      refLineIndex = #lines
    end
  end
  local refLine = lines[refLineIndex].line
  if isItem(refLine) then
    return extractItem(lines, refLineIndex)
  else
    return extractParagraph(lines, refLineIndex)
  end
end

---Returns true if any of the given lines includes the given ref.
---Expects the given ref and lines to belong to the same page.
---@param lines { line: string, startIndex: integer }[]
---@param ref string
---@return boolean
local function includeRef(lines, ref)
  local pos = refPos(ref)
  for _, line in ipairs(lines) do
    local endIndex = line.startIndex + string.len(line.line)
    if line.startIndex <= pos and endIndex >= pos then
      return true
    end
  end
  return false
end

---Builds the snippet string from an array of line objects.
---@param lines { line: string, startIndex: integer }[]
---@return string
local function buildSnippet(lines)
  local snippet = ""
  for _, line in ipairs(lines) do
    snippet = snippet .. line.line .. "\n"
  end
  return string.trimEnd(snippet)
end

local mentionTemplate = template.new [==[
**[[${_.ref}]]**
${_.snippet}

]==]

function widgets.journalEntries(pageName)
  pageName = pageName or editor.getCurrentPage()
  local journalMentions = query[[
    from index.tag "link"
    where _.page.startsWith("Journals/")
      and _.toPage == pageName
    order by page desc
  ]]
  if #journalMentions > 0 then
    local markdown = "# Journal Entries\n"
    local page = ""
    local linesOnPage = {}
    for _, item in ipairs(journalMentions) do
      if item.page == page and includeRef(linesOnPage, item.ref) then
        -- continue
      else
        if page != item.page then
          page = item.page
          linesOnPage = {}
        end
        extractedLines = extractLines(item)
        for _, line in ipairs(extractedLines) do
          table.insert(linesOnPage, line)
        end
        markdown = markdown .. mentionTemplate({
          ref = item.ref,
          snippet = buildSnippet(extractedLines)
        })
      end
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
