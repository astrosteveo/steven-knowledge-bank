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

The risk here is including large subsets of dependencies in the base image also means the initial image pull is longer. If you're delivering the image on physical media then the risk is lower. If you're pulling this image down over the network (whether local or internet), it will increase the initial pull.
## Build as a separate OCI image

The `.env` environment itself can be built as an OCI image and then pushed to a centralized OCI repository such as Nexus. Then this image is mounted to the pod using `pod.spec.volumes.image`. The image can 

## Mount as NFS

Works just as it appears. You mount an NFS share. This assumes the cluster will have access to the server hosting the NFS shared. In air-gapped networks this could work long as there is a connection back to the NFS share. This would likely not be suitable in bandwidth-constrained environments.

---
*Captured at 14:49 — Monday, March 30th 2026*
