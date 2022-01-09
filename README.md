# How to use service mesh

To consider it, it is necessary to understand independent dimensions of configuration for an Istio deployment as below:

- single or multiple cluster
- single or multiple network
- single or multiple control plane
- single or multiple mesh

# 1. Without Istio service mesh

```

Cluster1                                       Cluster2
+-----------------------------------------+    +-----------------------------------------+
|  ClusterIP (10.105.235.xxx)             |    |  ClusterIP (10.108.214.xxx)             |
|  -----+-------+---------+-------+-----  |    |  -----+-------+---------+-------+-----  |
|       |       |         |       |       |    |       |       |         |       |       |
|  +----+----+  |    +----+----+  |       |    |  +----+----+  |    +----+----+  |       |
|  |  svc-A  |  |    |  svc-B  |  |       |    |  |  svc-C  |  |    |  svc-D  |  |       |
|  +---------+  |    +---------+  |       |    |  +---------+  |    +---------+  |       |
|               |                 |       |    |               |                 |       |
|  +-Pod-----+  |    +-Pod-----+  |       |    |  +-Pod-----+  |    +-Pod-----+  |       |
|  |  cnt-A  +--+    |  cnt-B  +--+       |    |  |  cnt-C  +--+    |  cnt-D  +--+       |
|  +---------+       +---------+          |    |  +---------+       +---------+          |
+-----------------------------------------+    +-----------------------------------------+



```
