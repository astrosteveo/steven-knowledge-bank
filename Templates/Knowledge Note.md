---
tags:
  - <% tp.system.suggester(["Ansible", "Kubernetes", "Helm"], ["ansible", "kubernetes", "helm"]) %>
  - <% tp.system.prompt("Sub-tag (e.g. ansible/playbooks, kubernetes/networking)") %>
topic: <% tp.system.suggester(["Getting Started", "Architecture", "Charts", "Configuration", "Development", "Execution", "Infrastructure", "Networking", "Observability", "Operations", "Organization", "Playbooks", "Quick Reference", "Scheduling", "Security", "Storage", "Variables and Facts", "Workloads"], ["Getting Started", "Architecture", "Charts", "Configuration", "Development", "Execution", "Infrastructure", "Networking", "Observability", "Operations", "Organization", "Playbooks", "Quick Reference", "Scheduling", "Security", "Storage", "Variables and Facts", "Workloads"]) %>
created: <% tp.date.now("YYYY-MM-DD") %>
---

# <% tp.file.title %>

## Overview

<% tp.file.cursor(1) %>
