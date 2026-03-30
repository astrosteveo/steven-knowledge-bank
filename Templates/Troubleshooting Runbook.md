---
tags:
  - runbook
  - <% tp.system.suggester(["Ansible", "Kubernetes", "Helm", "Infrastructure"], ["ansible", "kubernetes", "helm", "infrastructure"]) %>
created: <% tp.date.now("YYYY-MM-DD") %>
severity: <% tp.system.suggester(["Critical — service down", "High — degraded", "Medium — workaround exists", "Low — cosmetic"], ["critical", "high", "medium", "low"]) %>
---

# <% tp.file.title %>

## Symptoms

<% tp.file.cursor(1) %>

## Root Cause

<% tp.file.cursor(2) %>

## Resolution Steps

1. <% tp.file.cursor(3) %>

## Verification

```bash
# Commands to verify the fix
```

## Prevention

- 

---
*Last used: <% tp.date.now("YYYY-MM-DD") %>*
