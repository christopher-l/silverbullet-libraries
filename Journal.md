```space-lua
local function isRootItem(line)
  if line:match("^[%-%*]%s+") ~= nil then
    return true
  elseif line:match("^%d%.%s+") ~= nil then
    return true
  else
    return false
  end
end

local function isSubItem(line)
  if line:match("^%s+[%-%*]%s+") ~= nil then
    return true
  elseif line:match("^%s+%d%.%s+") ~= nil then
    return true
  else
    return false
  end
end

local function extractRootItem(lines, refLineIndex)
  local idx = refLineIndex
  local snippet = lines[refLineIndex].line
  while #lines > idx + 1 do
    if isSubItem(lines[idx + 1].line) then
      idx = idx + 1
      snippet = snippet .."\n" .. lines[idx].line
    else
      break
    end
  end
  return {
    snippet = snippet,
    startIndex = lines[refLineIndex].startIndex,
    endIndex = lines[idx].startIndex + string.len(lines[idx].line),
  }
end

local function extractSnippet(item)
  local posString = item.ref:match("@(%d+)$")
  local pos = tonumber(posString)
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
  local m = refLine:match('^%s*[%-%*]%s+')
  if refLine:match("^[%-%*]%s+") ~= nil then
    return extractRootItem(lines, refLineIndex)
  elseif refLine:match("%s+^[%-%*]%s+") ~= nil then
    -- mode = "sub-item"
  else
    --mode = "other"
  end
  return page
end

local mentionTemplate = template.new [==[
**[[${_.ref}]]**
${_.snippet}

]==]

function foo(pageName)
  pageName = pageName or editor.getCurrentPage()
  local journalMentions = query[[
    from index.tag "link"
    where _.page.startsWith("Journals/")
      and _.toPage == pageName
    order by page desc
    limit 9
  ]]
  if #journalMentions > 0 then
    local markdown = "# Linked References\n"
    for _, item in ipairs(journalMentions) do
      extractedSnippet = extractSnippet(item)
      markdown = markdown .. mentionTemplate({
        ref = item.ref,
        snippet = extractedSnippet.snippet
      })
    end
    return widget.new {
      markdown = markdown
    }
  end
end
```
