# How to use service mesh

To consider it, it is necessary to understand independent dimensions of configuration for an Istio deployment as below:

- single or multiple cluster  --> But what I'm interested in is a behavior of multi-cluster case only.
- single or multiple network
- single or multiple control plane
- single or multiple mesh

See also https://istio.io/latest/docs/ops/deployment/deployment-models.

# 1. Without Istio service mesh

- If EndPoint-A has a logic accessing to Container-B in the same cluster, the access goes through svc-B because it needs to have the stable IP not by a ephemeral IP.
- If EndPoint-A has a logic accessing to Container-B in the diffrent cluster, it is not reachable because ClusterIP is not shared each other even though clusters are located in same network.
- Kube-proxy creates an iptables rule for each of the backend Pods in the Service.
- After catching the traffic sent to the ClusterIP, iptables forwards that traffic directly to one of the backend Pod using DNAT. 

The following is a case of

- multiple cluster
- single network
- no control plane
- no mesh
```
Cluster1                                                Cluster2
+--------------------------------------------------+    +--------------------------------------------------+
|           ClusterIP (10.105.235.xxx)             |    |           ClusterIP (10.108.214.xxx)             |
|           -----+--------+---------+--------+     |    |           -----+--------+---------+--------+     |
|                |        |         |        |     |    |                |        |         |        |     |
| +------+  +----+-----+  |    +----+-----+  |     |    | +------+  +----+-----+  |    +----+-----+  |     |
| | Kube |  |  svc-A   |  |    |  svc-B   |  |     |    | | Kube |  |  svc-B   |  |    |  svc-C   |  |     |
| | Prxy |-->(iptable) |  |  -->(iptable) |  |     |    | | Prxy |-->(iptable) |  |  -->(iptable) |  |     |
| +------+  +----------+  |    +----------+  |     |    | +------+  +----------+  |    +----------+  |     |
|                         |                  |     |    |                         |                  |     |
|           +-Pod------+  |    +-Pod------+  |     |    |           +-Pod------+  |    +-Pod------+  |     |
|           | EndPnt-A +--+    | EndPnt-B +--+     |    |           | EndPnt-B +--+    | EndPnt-C +--+     |
|           +----------+       +----------+        |    |           +----------+       +----------+        |
+--------------------------------------------------+    +--------------------------------------------------+
```

Of cource you can expose services to outside thru LoadBalancer and Ingress so that comunicate between clusters as followings:
```
Cluster1                                                Cluster2
     +--------------+                                        +--------------+                            
+----| LoadBalancer |------------------------------+    +----| LoadBalancer |------------------------------+
|    +----+---------+                              |    |    +----+---------+                              |
|         |                                        |    |         |                                        | 
|    +----+--------------+                         |    |    +----+--------------+                         | 
|    | IngressController |                         |    |    | IngressController |                         |
|    +----+--------------+                         |    |    +----+--------------+                         |
|         | ClusterIP (10.105.235.xxx)             |    |         | ClusterIP (10.108.214.xxx)             |
|    -----+------+--------+---------+--------+     |    |    -----+------+--------+---------+--------+     |
|                |        |         |        |     |    |                |        |         |        |     |
| +------+  +----+-----+  |    +----+-----+  |     |    | +------+  +----+-----+  |    +----+-----+  |     |
| | Kube |  |  svc-A   |  |    |  svc-B   |  |     |    | | Kube |  |  svc-B   |  |    |  svc-C   |  |     |
| | Prxy |-->(iptable) |  |  -->(iptable) |  |     |    | | Prxy |-->(iptable) |  |  -->(iptable) |  |     |
| +------+  +----------+  |    +----------+  |     |    | +------+  +----------+  |    +----------+  |     |
|                         |                  |     |    |                         |                  |     |
|           +-Pod------+  |    +-Pod------+  |     |    |           +-Pod------+  |    +-Pod------+  |     |
|           | EndPnt-A +--+    | EndPnt-B +--+     |    |           | EndPnt-B +--+    | EndPnt-C +--+     |
|           +----------+       +----------+        |    |           +----------+       +----------+        |
+--------------------------------------------------+    +--------------------------------------------------+
```

# 2. With a single Istio service mesh in the same network

- If EndPoint-A has a logic accessing to Service-B, the access is hi-jacked by Envoy proxy in the pod. The access travels along on service mesh. 
- There is an endpoint of Service-B in each cluster (both Cluster1 and Cluster2). So, the access from EndPoint-A will go to either EndPoint-B in Cluster1 or in Cluster2.
- Each cluster must have a DNS entry for the service in order for the DNS lookup to succeed, and a request to be successfully sent. This is true even if there are no instances of that service’s pods running in the cluster which makes requests.

The following is a case of 
- multiple cluster
- single network
- single control plane
- single mesh

```
Cluster1                                                Cluster2
     +--------------+                                        +--------------+                            
+----| LoadBalancer |------------------------------+    +----| LoadBalancer |------------------------------+
|    +----+---------+                              |    |    +----+---------+                              |
|         |                                        |    |         |                                        | 
|    +----+--------------+                         |    |    +----+--------------+                         | 
|    | IngressController |                         |    |    | IngressController |                         |
|    +----+--------------+                         |    |    +----+--------------+                         |
|         | ClusterIP (10.105.235.xxx)             |    |         | ClusterIP (10.108.214.xxx)             |
|    -----+------+--------+---------+--------+     |    |    -----+------+--------+---------+--------+     |
|                |        |         |        |     |    |                |        |         |        |     |
| +------+  +----+-----+  |    +----+-----+  |     |    | +------+  +----+-----+  |    +----+-----+  |     |
| | Kube |  |  svc-A   |  |    |  svc-B   |  |     |    | | Kube |  |  svc-B   |  |    |  svc-C   |  |     |
| | Prxy |-->(iptable) |  |  -->(iptable) |  |     |    | | Prxy |-->(iptable) |  |  -->(iptable) |  |     |
| +------+  +----------+  |    +----------+  |     |    | +------+  +----------+  |    +----------+  |     |
|                         |                  |     |    |                         |                  |     |
|           +-Pod------+  |    +-Pod------+  |     |    |           +-Pod------+  |    +-Pod------+  |     |
| +------+  | +------+ |  |    | +------+ |  |     |    |  From     | +------+ |  |    | +------+ |  |     |
| |Istiod|--->| Prxy |===========| Prxy |============+  |  Istiod --->| Prxy |===========| Prxy |============+
| +--+---+  | +------+ |  |    | +------+ |  |     | |  |  in the   | +------+ |  |    | +------+ |  |     | |
|    |      | EndPnt-A +--+    | EndPnt-B +--+     | |  |  Cluster1 | EndPnt-B +--+    | EndPnt-C +--+     | |
| +--+---+  +----------+       +----------+        | |  |           +----------+       +----------+        | |
| |  CA  |                                         | |  |                                                  | |
| +------+                                         | |  |                                                  | |
+--------------------------------------------------+ |  +--------------------------------------------------+ |
                                                     | mTLS between Istio Proxys                             |
                                                     +=======================================================+ 
```

# 3. With multiple Istio service mesh in the same network
- To enable communication between two meshes with different CAs, you must exchange the trust bundles of the meshes. 

The following is a case of 
- multiple cluster
- single network
- multiple control plane
- multiple mesh

```
Cluster1                                                Cluster2
     +--------------+                                        +--------------+                            
+----| LoadBalancer |------------------------------+    +----| LoadBalancer |------------------------------+
|    +----+---------+                              |    |    +----+---------+                              |
|         |                                        |    |         |                                        | 
|    +----+--------------+                         |    |    +----+--------------+                         | 
|    | IngressController |                         |    |    | IngressController |                         |
|    +----+--------------+                         |    |    +----+--------------+                         |
|         | ClusterIP (10.105.235.xxx)             |    |         | ClusterIP (10.108.214.xxx)             |
|    -----+------+--------+---------+--------+     |    |    -----+------+--------+---------+--------+     |
|                |        |         |        |     |    |                |        |         |        |     |
| +------+  +----+-----+  |    +----+-----+  |     |    | +------+  +----+-----+  |    +----+-----+  |     |
| | Kube |  |  svc-A   |  |    |  svc-B   |  |     |    | | Kube |  |  svc-B   |  |    |  svc-C   |  |     |
| | Prxy |-->(iptable) |  |  -->(iptable) |  |     |    | | Prxy |-->(iptable) |  |  -->(iptable) |  |     |
| +------+  +----------+  |    +----------+  |     |    | +------+  +----------+  |    +----------+  |     |
|                         |                  |     |    |                         |                  |     |
|           +-Pod------+  |    +-Pod------+  |     |    |           +-Pod------+  |    +-Pod------+  |     |
| +------+  | +------+ |  |    | +------+ |  |     |    | +------+  | +------+ |  |    | +------+ |  |     |
| |Istiod|--->| Prxy |===========| Prxy |============+  | |Istiod|--->| Prxy |===========| Prxy |============+
| +--+---+  | +------+ |  |    | +------+ |  |     | |  | +--+---+  | +------+ |  |    | +------+ |  |     | |
|    |      | EndPnt-A +--+    | EndPnt-B +--+     | |  |    |      | EndPnt-B +--+    | EndPnt-C +--+     | |
| +--+---+  +----------+       +----------+        | |  | +--+---+  +----------+       +----------+        | |
| |  CA  |                                         | |  | |  CA  |                                         | |
| +------+ with Cluster2's Trust Bundles           | |  | +------+ with Cluster1's Trust Bundles           | |
+--------------------------------------------------+ |  +--------------------------------------------------+ |
                                                     | mTLS between Istio Proxys                             |
                                                     +=======================================================+ 
```

# 4. With multiple Istio service mesh in different network
- In order to ensure secure communications in a multi-network scenario, Istio only supports cross-network communication to workloads with an Istio proxy. This is due to the fact that Istio exposes services at the Ingress Gateway with TLS pass-through, which enables mTLS directly to the workload.
- IngressGateway consumes "virtualservice", while IngressController consumes "Ingress" as a resouce.
- You may use Intel QAT for acceleration of TLS process in Istio proxy using Intel QAT device plugin. (https://01.org/kubernetes/solutions/QAT-envoy-solution)

The following is a case of 
- multiple cluster
- multiple network
- multiple control plane
- multiple mesh

```
                                      mTLS between Istio Proxys with TLS pass-through
                                    +=======================================================+ 
                                    |                                                       |
     +--------------+          +----+---------+              +--------------+          +----+---------+ 
+----| LoadBalancer |----------| LoadBalancer |----+    +----| LoadBalancer |----------| LoadBalancer |----+
|    +----+---------+          +----+---------+    |    |    +----+---------+          +----+---------+    | 
|         |                         |              |    |         |                         |              | 
|    +----+--------------+     +----+------------+ |    |    +----+--------------+     +----+------------+ |  
|    | IngressController |     | IngressGateway  | |    |    | IngressController |     | IngressGateway  | |
|    +----+--------------+     +---------------+-+ |    |    +----+--------------+     +---------------+-+ |
|         | ClusterIP (10.105.235.xxx)         |   |    |         | ClusterIP (10.108.214.xxx)         |   |
|    -----+------+--------+---------+--------+ |   |    |    -----+------+--------+---------+--------+ |   |
|                |        |         |        | |   |    |                |        |         |        | |   |
| +------+  +----+-----+  |    +----+-----+  | |   |    | +------+  +----+-----+  |    +----+-----+  | |   |
| | Kube |  |  svc-A   |  |    |  svc-B   |  | |   |    | | Kube |  |  svc-B   |  |    |  svc-C   |  | |   |
| | Prxy |-->(iptable) |  |  -->(iptable) |  | |   |    | | Prxy |-->(iptable) |  |  -->(iptable) |  | |   |
| +------+  +----------+  |    +----------+  | |   |    | +------+  +----------+  |    +----------+  | |   |
|                         |                  | |   |    |                         |                  | |   |
|           +-Pod------+  |    +-Pod------+  | |   |    |           +-Pod------+  |    +-Pod------+  | |   |
| +------+  | +------+ |  |    | +------+ |  | |   |    | +------+  | +------+ |  |    | +------+ |  | |   |
| |Istiod|--->| Prxy |===========| Prxy |======+   |    | |Istiod|--->| Prxy |===========| Prxy |======+   |
| +--+---+  | +------+ |  |    | +------+ |  |     |    | +--+---+  | +------+ |  |    | +------+ |  |     | 
|    |      | EndPnt-A +--+    | EndPnt-B +--+     |    |    |      | EndPnt-B +--+    | EndPnt-C +--+     | 
| +--+---+  +----------+       +----------+        |    | +--+---+  +----------+       +----------+        | 
| |  CA  |                                         |    | |  CA  |                                         | 
| +------+ with Cluster2's Trust Bundles           |    | +------+ with Cluster1's Trust Bundles           | 
+--------------------------------------------------+    +--------------------------------------------------+ 
Cluster1                                                Cluster2
```
