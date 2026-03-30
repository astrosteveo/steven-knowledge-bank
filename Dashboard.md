---
tags:
  - MOC
aliases:
  - Home
---

# Dashboard

## Recently Modified
```dataview
TABLE file.mtime AS "Last Modified", topic AS "Topic"
FROM ""
WHERE file.name != "Dashboard"
SORT file.mtime DESC
LIMIT 10
```

## Fleeting Notes Inbox
```dataview
TABLE created AS "Captured"
FROM #fleeting
WHERE file.name != "Welcome"
SORT created DESC
```

## Knowledge Areas

| Area | Notes |
|---|---|
| [[Ansible/Ansible\|Ansible]] | Automation & configuration management |
| [[Kubernetes/Kubernetes\|Kubernetes]] | Container orchestration |
| [[Helm/Helm\|Helm]] | Kubernetes package management |

## All Tags
```dataview
TABLE length(rows) AS "Count"
FROM ""
WHERE file.tags
FLATTEN file.tags AS tag
GROUP BY tag
SORT length(rows) DESC
```
