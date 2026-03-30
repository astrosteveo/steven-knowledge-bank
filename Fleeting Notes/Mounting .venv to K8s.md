---
aliases: 
tags:
  - fleeting
  - kubernetes
created: 2026-03-30
status: inbox
title: Mounting .venv to K8s
date created: Monday, March 30th 2026, 2:49:19 pm
date modified: Monday, March 30th 2026, 3:31:16 pm
---

Sometimes you need to run Kubernetes in an air-gapped or a bandwidth-constrained environment. Simply pulling large amounts of dependencies over the internet is often not possible (in the case of a truly air-gapped network), or if it is bandwidth-constrained, waiting hours for 10+ gigs worth of dependencies to download is not an option.

There are some options for accessing large dependencies.

## Build into the container image

This is the old school standard way of doing things. Simply include the dependencies as part of the container image. If running older versions of K8s, it is the gold-standard, tried and tested way of including data and dependencies.

The risk here is an increase in the initial container image.

## Build as a separate OCI image

This is a recent advancement in Kubernetes, first entering Alpha stage in 1.31, Beta in 1.33, and becomes a default-on feature in 1.35 Beta. That likely means it is going to mature even more once 1.35 is GA and we're probably looking at a production-ready feature soon thereafter.

Any directory can be built as an OCI image. This would involve simply building the Python virtual environment (`./.env`) and then pushing that image to Nexus and mounting it as an Image Volume on the Pod.

Here is an example what that looks like in practice:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: clearml-venv
spec:
// ... Rest of spec
  volumes:
  - name: clearml-venv
    image:
      reference: 10.22.73.66:32600/clearml-venv:testing
      pullPolicy: IfNotPresent
// ... Rest of spec
```

## Mount as NFS

This assumes the cluster will have access to the server hosting the NFS shared. This would likely not be suitable in bandwidth-constrained environments. In the context of a 10+ GB virtual environment, it's not an avenue I would use.

---

*Captured at 14:49 — Monday, March 30th 2026*
