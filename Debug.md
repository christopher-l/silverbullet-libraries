#meta/library

```space-lua
-- priority: 100

-- Adapted from https://stackoverflow.com/a/27028488
function dump(o)
   if type(o) == 'table' then
      local s = '{ '
      for k,v in pairs(o) do
         if type(k) ~= 'number' then k = '"'..k..'"' end
         s = s .. '['..k..'] = ' .. dump(v) .. ','
      end
      return s .. '} '
   elseif type(o) == 'string' then
     return "'" ..o:gsub("\n", "\\n").. "'"
   else
      return tostring(o)
   end
end
```
