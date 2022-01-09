# How to use service mesh

To consider it, it is necessary to understand independent dimensions of configuration for an Istio deployment as below:

- single or multiple cluster
- single or multiple network
- single or multiple control plane
- single or multiple mesh

# 1. Without Istio service mesh

- If Container-A has a logic accessing to Container-B in the same cluster, the access goes through svc-B because it needs to have the stable IP not by a ephemeral IP.
- If Container-A has a logic accessing to Container-C in the diffrent cluster, it is not reachable because ClusterIP is not shared each other even though clusters are located in same network.

Kube-proxy creates an iptables rule for each of the backend Pods in the Service as you can see below:
```
Cluster1                                             Cluster2
+-----------------------------------------------+    +-----------------------------------------------+
|           ClusterIP (10.105.235.xxx)          |    |           ClusterIP (10.108.214.xxx)          |
|           -----+-------+---------+-------+--  |    |           -----+-------+---------+-------+--  |
|                |       |         |       |    |    |                |       |         |       |    |
| +-----+   +----+----+  |    +----+----+  |    |    | +-----+   +----+----+  |    +----+----+  |    |
| |Kube-|   |  svc-A  |  |    |  svc-B  |  |    |    | |Kube |   |  svc-C  |  |    |  svc-D  |  |    |
| |Proxy|-->|(iptable)|  |    |(iptable)|  |    |    | |Proxy|-->|(iptable)|  |    |(iptable)|  |    |
| +-----+   +---------+  |    +---------+  |    |    | +-----+   +---------+  |    +---------+  |    |
|                        |                 |    |    |                        |                 |    |
|           +-Pod-----+  |    +-Pod-----+  |    |    |           +-Pod-----+  |    +-Pod-----+  |    |
|           |  cnt-A  +--+    |  cnt-B  +--+    |    |           |  cnt-C  +--+    |  cnt-D  +--+    |
|           +---------+       +---------+       |    |           +---------+       +---------+       |
+-----------------------------------------------+    +-----------------------------------------------+
```

Of cource you can expose services to outside thru LoadBalancer and Ingress so that comunicate between clusters as followings:
```
Cluster1                                             Cluster2
     +--------------+                                     +--------------+                            
+----| LoadBalancer |---------------------------+    +----| LoadBalancer |---------------------------+
|    +----+---------+                           |    |    +----+---------+                           |
|         |                                     |    |         |                                     | 
|    +----+----+                                |    |    +----+----+                                | 
|    | Ingress |                                |    |    | Ingress |                                |
|    +----+----+                                |    |    +----+----+                                |
|         | ClusterIP (10.105.235.xxx)          |    |         | ClusterIP (10.108.214.xxx)          |
|    -----+------+-------+---------+-------+--  |    |    -----+------+-------+---------+-------+--  |
|                |       |         |       |    |    |                |       |         |       |    |
| +-----+   +----+----+  |    +----+----+  |    |    | +-----+   +----+----+  |    +----+----+  |    |
| |Kube-|   |  svc-A  |  |    |  svc-B  |  |    |    | |Kube |   |  svc-C  |  |    |  svc-D  |  |    |
| |Proxy|-->|(iptable)|  |    |(iptable)|  |    |    | |Proxy|-->|(iptable)|  |    |(iptable)|  |    |
| +-----+   +---------+  |    +---------+  |    |    | +-----+   +---------+  |    +---------+  |    |
|                        |                 |    |    |                        |                 |    |
|           +-Pod-----+  |    +-Pod-----+  |    |    |           +-Pod-----+  |    +-Pod-----+  |    |
|           |  cnt-A  +--+    |  cnt-B  +--+    |    |           |  cnt-C  +--+    |  cnt-D  +--+    |
|           +---------+       +---------+       |    |           +---------+       +---------+       |
+-----------------------------------------------+    +-----------------------------------------------+
```

