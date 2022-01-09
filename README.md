# How to use service mesh

To consider it, it is necessary to understand independent dimensions of configuration for an Istio deployment as below:

- single or multiple cluster  --> But what I'm interested in is a behavior of multi-cluster case only.
- single or multiple network
- single or multiple control plane
- single or multiple mesh

# 1. Without Istio service mesh

- If EndPoint-A has a logic accessing to Container-B in the same cluster, the access goes through svc-B because it needs to have the stable IP not by a ephemeral IP.
- If EndPoint-A has a logic accessing to Container-B in the diffrent cluster, it is not reachable because ClusterIP is not shared each other even though clusters are located in same network.

Kube-proxy creates an iptables rule for each of the backend Pods in the Service as you can see below:
```
Cluster1                                              Cluster2
+-------------------------------------------------+    +-------------------------------------------------+
|           ClusterIP (10.105.235.xxx)            |    |           ClusterIP (10.108.214.xxx)            |
|           -----+--------+---------+--------+--  |    |           -----+--------+---------+--------+--  |
|                |        |         |        |    |    |                |        |         |        |    |
| +-----+   +----+-----+  |    +----+-----+  |    |    | +-----+   +----+-----+  |    +----+-----+  |    |
| |Kube-|   |  svc-A   |  |    |  svc-B   |  |    |    | |Kube |   |  svc-B   |  |    |  svc-C   |  |    |
| |Proxy|-->|(iptable) |  |    |(iptable) |  |    |    | |Proxy|-->|(iptable) |  |    |(iptable) |  |    |
| +-----+   +----------+  |    +----------+  |    |    | +-----+   +----------+  |    +----------+  |    |
|                         |                  |    |    |                         |                  |    |
|           +-Pod------+  |    +-Pod------+  |    |    |           +-Pod------+  |    +-Pod------+  |    |
|           | EndPnt-A +--+    | EndPnt-B +--+    |    |           | EndPnt-B +--+    | EndPnt-C +--+    |
|           +----------+       +----------+       |    |           +----------+       +----------+       |
+-------------------------------------------------+    +-------------------------------------------------+
```

Of cource you can expose services to outside thru LoadBalancer and Ingress so that comunicate between clusters as followings:
```
Cluster1                                               Cluster2
     +--------------+                                       +--------------+                            
+----| LoadBalancer |-----------------------------+    +----| LoadBalancer |-----------------------------+
|    +----+---------+                             |    |    +----+---------+                             |
|         |                                       |    |         |                                       | 
|    +----+----+                                  |    |    +----+----+                                  | 
|    | Ingress |                                  |    |    | Ingress |                                  |
|    +----+----+                                  |    |    +----+----+                                  |
|         | ClusterIP (10.105.235.xxx)            |    |         | ClusterIP (10.108.214.xxx)            |
|    -----+------+--------+---------+--------+--  |    |    -----+------+--------+---------+--------+--  |
|                |        |         |        |    |    |                |        |         |        |    |
| +-----+   +----+-----+  |    +----+-----+  |    |    | +-----+   +----+-----+  |    +----+-----+  |    |
| |Kube-|   |  svc-A   |  |    |  svc-B   |  |    |    | |Kube |   |  svc-B   |  |    |  svc-C   |  |    |
| |Proxy|-->|(iptable) |  |    |(iptable) |  |    |    | |Proxy|-->|(iptable) |  |    |(iptable) |  |    |
| +-----+   +----------+  |    +----------+  |    |    | +-----+   +----------+  |    +----------+  |    |
|                         |                  |    |    |                         |                  |    |
|           +-Pod------+  |    +-Pod------+  |    |    |           +-Pod------+  |    +-Pod------+  |    |
|           | EndPnt-A +--+    | EndPnt-B +--+    |    |           | EndPnt-B +--+    | EndPnt-C +--+    |
|           +----------+       +----------+       |    |           +----------+       +----------+       |
+-------------------------------------------------+    +-------------------------------------------------+
```

# 2. With a single Istio service mesh in the same network

- If EndPoint-A has a logic accessing to Service-B, the access is hi-jacked by Envoy proxy in the pod. The access travels along on service mesh. 
- There is an endpoint of Service-B in each cluster (both Cluster1 and Cluster2). So, the access from EndPoint-A will go to either EndPoint-B in Cluster1 or in Cluster2.

The following is a case of 
- multiple cluster
- single network
- single control plane
- single mesh

```
Cluster1                                               Cluster2
     +--------------+                                       +--------------+                            
+----| LoadBalancer |-----------------------------+    +----| LoadBalancer |-----------------------------+
|    +----+---------+                             |    |    +----+---------+                             |
|         |                                       |    |         |                                       | 
|    +----+----+                                  |    |    +----+----+                                  | 
|    | Ingress |                                  |    |    | Ingress |                                  |
|    +----+----+                                  |    |    +----+----+                                  |
|         | ClusterIP (10.105.235.xxx)            |    |         | ClusterIP (10.108.214.xxx)            |
|    -----+------+--------+---------+--------+--  |    |    -----+------+--------+---------+--------+--  |
|                |        |         |        |    |    |                |        |         |        |    |
| +-----+   +----+-----+  |    +----+-----+  |    |    | +-----+   +----+-----+  |    +----+-----+  |    |
| |Kube-|   |  svc-A   |  |    |  svc-B   |  |    |    | |Kube |   |  svc-B   |  |    |  svc-C   |  |    |
| |Proxy|-->|(iptable) |  |    |(iptable) |  |    |    | |Proxy|-->|(iptable) |  |    |(iptable) |  |    |
| +-----+   +----------+  |    +----------+  |    |    | +-----+   +----------+  |    +----------+  |    |
|                         |                  |    |    |                         |                  |    |
|           +-Pod------+  |    +-Pod------+  |    |    |           +-Pod------+  |    +-Pod------+  |    |
| +------+  | +-----+  |  |    | +-----+  |  |    |    |           | +-----+  |  |    | +-----+  |  |    |
| |Istiod|--->|Proxy|============|Proxy|============+  |           | |Proxy|============|Proxy|============+
| +------+  | +-----+  |  |    | +-----+  |  |    | |  |           | +-----+  |  |    | +-----+  |  |    | |
|           | EndPnt-A +--+    | EndPnt-B +--+    | |  |           | EndPnt-B +--+    | EndPnt-C +--+    | |
|           +----------+       +----------+       | |  |           +----------+       +----------+       | |
+-------------------------------------------------+ |  +-------------------------------------------------+ |
                                                    | Mesh Network                                         |
                                                    +------------------------------------------------------+
```

# 3. With multiple Istio service mesh in the same network
- To enable communication between two meshes with different CAs, you must exchange the trust bundles of the meshes. 

The following is a case of 
- multiple cluster
- single network
- multiple control plane
- multiple mesh

```
Cluster1                                               Cluster2
     +--------------+                                       +--------------+                            
+----| LoadBalancer |-----------------------------+    +----| LoadBalancer |-----------------------------+
|    +----+---------+                             |    |    +----+---------+                             |
|         |                                       |    |         |                                       | 
|    +----+----+                                  |    |    +----+----+                                  | 
|    | Ingress |                                  |    |    | Ingress |                                  |
|    +----+----+                                  |    |    +----+----+                                  |
|         | ClusterIP (10.105.235.xxx)            |    |         | ClusterIP (10.108.214.xxx)            |
|    -----+------+--------+---------+--------+--  |    |    -----+------+--------+---------+--------+--  |
|                |        |         |        |    |    |                |        |         |        |    |
| +-----+   +----+-----+  |    +----+-----+  |    |    | +-----+   +----+-----+  |    +----+-----+  |    |
| |Kube-|   |  svc-A   |  |    |  svc-B   |  |    |    | |Kube |   |  svc-B   |  |    |  svc-C   |  |    |
| |Proxy|-->|(iptable) |  |    |(iptable) |  |    |    | |Proxy|-->|(iptable) |  |    |(iptable) |  |    |
| +-----+   +----------+  |    +----------+  |    |    | +-----+   +----------+  |    +----------+  |    |
|                         |                  |    |    |                         |                  |    |
|           +-Pod------+  |    +-Pod------+  |    |    |           +-Pod------+  |    +-Pod------+  |    |
| +------+  | +-----+  |  |    | +-----+  |  |    |    | +------+  | +-----+  |  |    | +-----+  |  |    |
| |Istiod|--->|Proxy|============|Proxy|============+  |Istiod|--->|Proxy|============|Proxy|==============+
| +--+---+  | +-----+  |  |    | +-----+  |  |    | |  | +--+---+  | +-----+  |  |    | +-----+  |  |    | |
|    |      | EndPnt-A +--+    | EndPnt-B +--+    | |  |    |      | EndPnt-B +--+    | EndPnt-C +--+    | |
| +--+---+  +----------+       +----------+       | |  | +--+---+  +----------+       +----------+       | |
| |  CA  |                                        | |  | |  CA  |                                        | |
| +------+                                        | |  | +------+                                        | |
+-------------------------------------------------+ |  +-------------------------------------------------+ |
                                                    | mTLS                                                 |
                                                    +******************************************************+ 
```

