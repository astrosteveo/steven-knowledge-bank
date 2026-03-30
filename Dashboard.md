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
TABLE created AS "Captured", status AS "Status"
FROM #fleeting
WHERE file.name != "Welcome"
SORT created DESC
```

## Troubleshooting Runbooks
```dataview
TABLE severity AS "Severity", created AS "Created"
FROM #runbook
SORT created DESC
```

## Knowledge Areas

| Area | Notes |
|---|---|
| [[Ansible/Ansible\|Ansible]] | Automation & configuration management |
| [[Kubernetes/Kubernetes\|Kubernetes]] | Container orchestration |
| [[Helm/Helm\|Helm]] | Kubernetes package management |

## Recent Daily Notes
```dataview
LIST
FROM #daily
SORT created DESC
LIMIT 7
```

## All Tags
```dataview
TABLE length(rows) AS "Count"
FROM ""
WHERE file.tags
FLATTEN file.tags AS tag
GROUP BY tag
SORT length(rows) DESC
```
