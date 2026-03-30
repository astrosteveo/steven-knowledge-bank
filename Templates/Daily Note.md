---
tags:
  - daily
created: <% tp.date.now("YYYY-MM-DD") %>
---

# <% tp.date.now("dddd, MMMM Do YYYY") %>

## What I'm Working On

- <% tp.file.cursor(1) %>

## Notes & Observations

- 

## Captured Today

```dataview
LIST
FROM #fleeting
WHERE created = date("<% tp.date.now("YYYY-MM-DD") %>")
SORT file.ctime ASC
```

## Links

**Yesterday:** [[<% tp.date.now("YYYY-MM-DD", -1) %>]] | **Tomorrow:** [[<% tp.date.now("YYYY-MM-DD", 1) %>]]
