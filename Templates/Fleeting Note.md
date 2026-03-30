---
tags:
  - fleeting
  - <% tp.system.suggester(["Ansible", "Kubernetes", "Helm", "General", "Idea", "Question", "Todo"], ["ansible", "kubernetes", "helm", "general", "idea", "question", "todo"]) %>
created: <% tp.date.now("YYYY-MM-DD") %>
status: <% tp.system.suggester(["Inbox", "Processed", "Promoted"], ["inbox", "processed", "promoted"]) %>
---

# <% tp.file.title %>

<% tp.file.cursor(1) %>

---
*Captured at <% tp.date.now("HH:mm") %> — <% tp.date.now("dddd, MMMM Do YYYY") %>*
