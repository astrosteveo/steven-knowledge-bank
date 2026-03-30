---
tags:
  - fleeting
  - kubernetes
created: 2026-03-30
status: inbox
---

Sometimes you need to run Kubernetes in an air-gapped or a bandwidth-constrained environment. Simply pulling large amounts of dependencies over the internet is often not possible (in the case of a truly air-gapped network), or if it is bandwidth-constrained, waiting hours for 10+ gigs worth of dependencies to download is not an option.

There are some options for accessing large dependencies.

## Build into the container image

This is the old school standard way of doing things. Simply include the dependencies as part of the container image. If running older versions of K8s, it is the gold-standard, tried and tested way of including data and dependencies.

The risk here is an increase in the initial container image.
## Build as a separate OCI image

This is a recent advancement in Kubernetes over the past year and a half, advancing to beta state in Kubernetes 1.33 (previously introduced as an Alpha feature in 1.31), and comes auto-enabled with Kubernetes 1.35 Beta. That likely means it is going to mature even more once 1.35 is GA.

Any directory can be built as an OCI image. This would involve simply building the Python virtual environment (`./.env`) and then pushing that image to Nexus and mounting it as an Image Volume on the Pod.

Here is an example what that looks like in practice:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: image-volume
spec:
  containers:
  - name: shell
    command: ["sleep", "infinity"]
    image: debian
    volumeMounts:
    - name: volume
      mountPath: /volume
  volumes:
  - name: volume
    image:
      reference: quay.io/crio/artifact:v2
      pullPolicy: IfNotPresent
```

## Mount as NFS

This assumes the cluster will have access to the server hosting the NFS shared. This would likely not be suitable in bandwidth-constrained environments.

---
*Captured at 14:49 — Monday, March 30th 2026*
