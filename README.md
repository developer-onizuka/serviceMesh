# How to use service mesh

To consider it, it is necessary to understand independent dimensions of configuration for an Istio deployment as below:

- single or multiple cluster  --> But what I'm interested in is a behavior of multi-cluster case only.
- single or multiple network
- single or multiple control plane
- single or multiple mesh

See also https://istio.io/latest/docs/ops/deployment/deployment-models.

# 1. Without Istio service mesh

- If EndPoint-A has a logic accessing to EndPoint-B in the same cluster, the access goes through Service-B because it needs to have the stable IP not by an ephemeral IP.
- If EndPoint-A has a logic accessing to EndPoint-B in the diffrent cluster, it is not reachable because ClusterIP is not shared with each other even though clusters are located in same network.
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
      consumers                                               consumers
          |                                                       |
     +----+---------+                                        +----+---------+                            
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
- An Istio control plane manages traffic within the mesh by providing each proxy with the list of service endpoints.
- Each cluster must have a DNS entry for the service in order for the DNS lookup to succeed, and a request to be successfully sent. This is true even if there are no instances of that service???s pods running in the cluster which makes requests.
- For clusters spanning networks, this can be achieved by exposing the control plane through an Istio gateway which goes thru such as internal load balancers. (But I don't draw the Istio gateway in the figure below, because of no space to draw it and it also gets so complicating to understand.)
- Istio Citadel is for CA (Certificate Authority). See also https://www.youtube.com/watch?v=LmZz8KpSY74.
- See also the video about mTLS (https://www.youtube.com/watch?v=7_O58efytvM)

The following is a case of 
- multiple cluster
- single network
- single control plane
- single mesh

```
Cluster1                                                Cluster2
      consumers                                               consumers
          |                                                       |
     +----+---------+                                        +----+---------+                            
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
                                                     | mTLS between Istio proxies                            |
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
      consumers                                               consumers
          |                                                       |
     +----+---------+                                        +----+---------+                            
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
                                                     | mTLS between Istio proxies                            |
                                                     +=======================================================+ 
```

# 4. With multiple Istio service mesh in different network
- In order to ensure secure communications in a multi-network scenario, Istio only supports cross-network communication to workloads with an Istio proxy. This is due to the fact that Istio exposes services at the Ingress Gateway with TLS pass-through, which enables mTLS directly to the workload.
- IngressGateway consumes "VirtualService", while IngressController consumes "Ingress" as a resouce.
- You may use Intel QAT for acceleration of TLS process in Istio proxy using Intel QAT device plugin. (https://01.org/kubernetes/solutions/QAT-envoy-solution)

The following is a case of 
- multiple cluster
- multiple network
- multiple control plane
- multiple mesh

```
                                      mTLS between Istio proxies with TLS pass-through
                                    +=======================================================+ 
      consumers                     |                         consumers                     | 
          |                         |                             |                         |
     +----+---------+          +----+---------+              +----+---------+          +----+---------+ 
+----| LoadBalancer |----------| LoadBalancer |----+    +----| LoadBalancer |----------| LoadBalancer |----+
|    +----+---------+          +----+---------+    |    |    +----+---------+          +----+---------+    | 
|         |                         |              |    |         |                         |              | 
|    +----+--------------+     +----+------------+ |    |    +----+--------------+     +----+------------+ |  
|    | IngressController |     | IngressGateway  | |    |    | IngressController |     | IngressGateway  | |
|    | routing between   |     | routing between | |    |    | routing between   |     | routing between | | 
|    | services          |     | virtualservices | |    |    | services          |     | virtualservices | |
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

# 5. Joining virtual machines into the mesh 
Istio provides two mechanisms to represent virtual machine workloads:
- WorkloadGroup represents a logical group of virtual machine workloads that share common properties. This is similar to a Deployment in Kubernetes.
- WorkloadEntry represents a single instance of a virtual machine workload. This is similar to a Pod in Kubernetes.
- In order for consumers to reliably call your workload, it???s recommended to declare a Service association. (But I don't draw it in the figure below.)

With this configuration, requests to product would be load-balanced across both the pod and virtual machine workload instances.

See also https://istio.io/latest/docs/ops/deployment/vm-architecture/ and https://www.youtube.com/watch?v=0B8maYcjq_c.

The following is a case of 
- multiple cluster       (2 Clusters)
- multiple network       (3 Networks)
- multiple control plane (2 Controle Planes)
- multiple mesh          (2 Meshes, But 3 Meshes if VirtualMachine has an independent CA rather than Cluster1 and2.)
```
                                      mTLS between Istio proxies with TLS pass-through
                                    +=======================================================+========================================+
      consumers                     |                          consumers                    |                                        |
          |                         |                             |                         |                                        |
     +----+---------+          +----+---------+              +----+---------+          +----+---------+                              |
+----| LoadBalancer |----------| LoadBalancer |----+    +----| LoadBalancer |----------| LoadBalancer |----+    +--VirtualMachine--+ |
|    +----+---------+          +----+---------+    |    |    +----+---------+          +----+---------+    |    |                  | |
|         |                         |              |    |         |                         |              |    |                  | |
|    +----+--------------+     +----+------------+ |    |    +----+--------------+     +----+------------+ |    |                  | |
|    | IngressController |     | IngressGateway  | |    |    | IngressController |     | IngressGateway  | |    |                  | |
|    +----+--------------+     +---------------+-+ |    |    +----+--------------+     +---------------+-+ |    |                  | |
|         | ClusterIP (10.105.235.xxx)         |   |    |         | ClusterIP (10.108.214.xxx)         |   |    |                  | |
|    -----+------+--------+---------+--------+ |   |    |    -----+------+--------+---------+--------+ |   |    |                  | |
|                |        |         |        | |   |    |                |        |         |        | |   |    |                  | |
| +------+  +----+-----+  |    +----+-----+  | |   |    | +------+  +----+-----+  |    +----+-----+  | |   |    |                  | |
| | Kube |  |  svc-A   |  |    |  svc-B   |  | |   |    | | Kube |  |  svc-B   |  |    |  svc-C   |  | |   |    |                  | |
| | Prxy |-->(iptable) |  |  -->(iptable) |  | |   |    | | Prxy |-->(iptable) |  |  -->(iptable) |  | |   |    |                  | |
| +------+  +----------+  |    +----------+  | |   |    | +------+  +----------+  |    +----------+  | |   |    |                  | |
|                         |                  | |   |    |                         |                  | |   |    |   WorkloadEntry  | |
|           +-Pod------+  |    +-Pod------+  | |   |    |           +-Pod------+  |    +-Pod------+  | |   |    |   +----------+   | |
| +------+  | +------+ |  |    | +------+ |  | |   |    | +------+  | +------+ |  |    | +------+ |  | |   |    |   | +------+ |   | |
| |Istiod|--->| Prxy |===========| Prxy |======+   |    | |Istiod|--->| Prxy |===========| Prxy |======+   |  +------>| Prxy |=======+
| +--+---+  | +------+ |  |    | +------+ |  |     |    | +--+---+  | +------+ |  |    | +------+ |  |     |  | |   | +------+ |   |
|    |      | EndPnt-A +--+    | EndPnt-B +--+     |    |    |      | EndPnt-B +--+    | EndPnt-C +--+     |  | |   | EndPnt-B |   | 
| +--+---+  +----------+       +----------+        |    | +--+---+  +----------+       +----------+        |  | |   +----------+   | 
| |  CA  |                                         |    | |  CA  |                                         |  | |                  | 
| +------+ with Cluster2's Trust Bundles           |    | +------+ with Cluster1's Trust Bundles           |  | |                  | 
+--------------------------------------------------+    +--------------------------------------------------+  | +------------------+
Cluster1                                                Cluster2                                              |
                                                                                                             From Istiod
                                                                                                             in the cluster1 or2
```

# 5-1. How to join virtual machines into the mesh in different network
> https://istio.io/latest/docs/setup/install/virtual-machine/

# 5-1-1. Create Secrets in kubernetes cluster

(1) Set some environment values.
```
VM_APP="vmapp"
VM_NAMESPACE="vmnamespace"
WORK_DIR="/home/vagrant/vmintegration"
SERVICE_ACCOUNT="mysvcaccount"
CLUSTER_NETWORK="kube-network"
VM_NETWORK="vm-network"
CLUSTER="cluster1"
```
(2) Make a work directory.
```
$ mkdir -p $WORK_DIR
$ kubectl create namespace "${VM_NAMESPACE}"
$ kubectl create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"
```
(3) Install the Istio control plane
```
$ curl -L https://istio.io/downloadIstio | sh -
$ export PATH=$PATH:/home/vagrant/istio-1.12.1/bin
$ cat <<EOF > ./vm-cluster.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: "${CLUSTER}"
      network: "${CLUSTER_NETWORK}"
EOF
```
```
$ istioctl install -f vm-cluster.yaml
$ istio-1.12.1/samples/multicluster/gen-eastwest-gateway.sh \
--mesh mesh1 --cluster "${CLUSTER}" --network "${CLUSTER_NETWORK}" | \
istioctl install -y -f -
```
(4) Check the gateways
```
$ kubectl get services -n istio-system
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.109.178.196   192.168.33.221   15021:30600/TCP,15443:31534/TCP,15012:31242/TCP,15017:30426/TCP   49m
istio-ingressgateway    LoadBalancer   10.110.212.70    192.168.33.220   15021:31932/TCP,80:30217/TCP,443:31930/TCP                        51m
istiod                  ClusterIP      10.111.13.175    <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP 
```
(5) Create the Virtualservices for Control Plane and  Data Plane
- Control Plane (for istiod)
```
$ kubectl apply -n istio-system -f istio-1.12.1/samples/multicluster/expose-istiod.yaml
gateway.networking.istio.io/istiod-gateway created
virtualservice.networking.istio.io/istiod-vs created
```
- Data Plane (for helloworld you run in next step)
```
$ kubectl apply -n istio-system -f istio-1.12.1/samples/multicluster/expose-services.yaml
gateway.networking.istio.io/cross-network-gateway created
```

(6) Check all of them
```
$ kubectl get -n istio-system gateway
NAME                    AGE
cross-network-gateway   5m11s
istiod-gateway          35m
```
```
$ kubectl describe gateway -n istio-system cross-network-gateway 
Name:         cross-network-gateway
Namespace:    istio-system
Labels:       <none>
Annotations:  <none>
API Version:  networking.istio.io/v1beta1
Kind:         Gateway
Metadata:
  Creation Timestamp:  2022-01-11T00:23:32Z
  Generation:          1
  Managed Fields:
    API Version:  networking.istio.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:selector:
          .:
          f:istio:
        f:servers:
    Manager:         kubectl-client-side-apply
    Operation:       Update
    Time:            2022-01-11T00:23:32Z
  Resource Version:  10242
  UID:               7c721d93-d386-4b97-a988-a0f7cbd8845b
Spec:
  Selector:
    Istio:  eastwestgateway
  Servers:
    Hosts:
      *.local
    Port:
      Name:      tls
      Number:    15443
      Protocol:  TLS
    Tls:
      Mode:  AUTO_PASSTHROUGH
Events:      <none>
```
```
$ kubectl describe gateway -n istio-system istiod-gateway 
Name:         istiod-gateway
Namespace:    istio-system
Labels:       <none>
Annotations:  <none>
API Version:  networking.istio.io/v1beta1
Kind:         Gateway
Metadata:
  Creation Timestamp:  2022-01-10T23:52:59Z
  Generation:          1
  Managed Fields:
    API Version:  networking.istio.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:selector:
          .:
          f:istio:
        f:servers:
    Manager:         kubectl-client-side-apply
    Operation:       Update
    Time:            2022-01-10T23:52:59Z
  Resource Version:  6108
  UID:               2c064853-0f9b-486f-96a4-30f15771cab1
Spec:
  Selector:
    Istio:  eastwestgateway
  Servers:
    Hosts:
      *
    Port:
      Name:      tls-istiod
      Number:    15012
      Protocol:  tls
    Tls:
      Mode:  PASSTHROUGH
    Hosts:
      *
    Port:
      Name:      tls-istiodwebhook
      Number:    15017
      Protocol:  tls
    Tls:
      Mode:  PASSTHROUGH
Events:      <none>
```

(7) Check if third party tokens are enabled in your cluster.
- See also https://istio.io/latest/docs/ops/best-practices/security/#configure-third-party-service-account-tokens
```
$ kubectl get --raw /api/v1 | jq '.resources[] | select(.name | index("serviceaccounts/token"))'
{
  "name": "serviceaccounts/token",
  "singularName": "",
  "namespaced": true,
  "group": "authentication.k8s.io",
  "version": "v1",
  "kind": "TokenRequest",
  "verbs": [
    "create"
  ]
}
```
(8) Make a Yaml file for workloadgroup.
```
$ cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
  metadata:
    labels:
      app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
    network: "${VM_NETWORK}"
EOF
```
(9) Generate the istio-token and Copy it to the VM which you want to join into the cluster.
```
$ istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}"
Warning: a security token for namespace "vmnamespace" and service account "mysvcaccount" has been generated and stored at "/home/vagrant/vmintegration/istio-token"
Configuration generation into directory /home/vagrant/vmintegration was successful
```
```
$ cd /home/vagrant
$ tar cvfz vmintegration.tar.gz vmintegration/
$ scp -p vmintegration.tar.gz vagrant@<ipaddress of vm>:/home/vagrant/
```

# 5-1-2. Put the secrets in Virtual Machine
(1) Copy them to the certain directry
```
$ tar xvfz vmintegration.tar.gz
$ cd vmintegration
$ sudo mkdir -p /etc/certs
$ sudo cp root-cert.pem /etc/certs/
$ sudo mkdir -p /var/run/secrets/tokens
$ sudo cp istio-token /var/run/secrets/tokens/
```
(2) Install Istio Agent
```
$ curl -LO https://storage.googleapis.com/istio-release/releases/1.12.1/deb/istio-sidecar.deb
$ sudo dpkg -i istio-sidecar.deb
$ sudo cp cluster.env /var/lib/istio/envoy/
$ sudo cp mesh.yaml /etc/istio/config/mesh
```
(3) Add the host and IP to resolve 
```
$ sudo sh -c 'cat hosts >> /etc/hosts'
$ cat /etc/hosts |grep istiod
192.168.33.221 istiod.istio-system.svc
```
(5) Change owner for each directry
```
$ sudo mkdir -p /etc/istio/proxy
$ sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
```
(6) Run istio Agent
```
$ sudo systemctl start istio
$ tail -f /var/log/istio/istio.log
2022-01-10T12:29:12.168300Z	info	cache	generated new workload certificate	latency=176.531403ms ttl=23h59m59.831707486s
2022-01-10T12:29:12.168330Z	info	cache	Root cert has changed, start rotating root cert
2022-01-10T12:29:12.168341Z	info	ads	XDS: Incremental Pushing:0 ConnectedEndpoints:2 Version:
2022-01-10T12:29:12.168432Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.831593702s
2022-01-10T12:29:12.168466Z	info	cache	returned workload certificate from cache	ttl=23h59m59.831536073s
2022-01-10T12:29:12.168535Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.831467947s
2022-01-10T12:29:12.168940Z	info	ads	SDS: PUSH request for node:mvc.vmnamespace resources:1 size:1.1kB resource:ROOTCA
2022-01-10T12:29:12.169019Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.830996231s
2022-01-10T12:29:12.169044Z	info	ads	SDS: PUSH request for node:mvc.vmnamespace resources:1 size:4.0kB resource:default
2022-01-10T12:29:12.169084Z	info	ads	SDS: PUSH for node:mvc.vmnamespace resources:1 size:1.1kB resource:ROOTCA
```

# 5-1-3. Run services in kubernetes cluster
```
$ kubectl label namespace vmnamespace istio-injection=enabled
$ kubectl apply -n vmnamespace -f istio-1.12.1/samples/helloworld/helloworld.yaml 
service/helloworld created
deployment.apps/helloworld-v1 created
deployment.apps/helloworld-v2 created
```
```
$ cat <<EOF | sudo kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu
  namespace: vmnamespace
  labels:
    name: ubuntu
spec:
  containers:
  - name: ubuntu
    image: ubuntu:20.04
    command:
    - sleep
    - "360000"
EOF

$ kubectl exec -n vmnamespace -it ubuntu -- apt-get update
$ kubectl exec -n vmnamespace -it ubuntu -- apt-get install -y curl dnsutils
$ kubectl exec -n vmnamespace -it ubuntu -- nslookup helloworld
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	helloworld.vmnamespace.svc.cluster.local
Address: 10.98.163.121

$ kubectl exec -n vmnamespace -it ubuntu -- curl helloworld:5000/hello
Hello version: v2, instance: helloworld-v2-5b46bc9f84-bd6qj
```

# 5-1-4. Access to the service from Virtual Machine
```
$ nslookup helloworld.vmnamespace.svc
Server:		127.0.0.53
Address:	127.0.0.53#53

Name:	helloworld.vmnamespace.svc
Address: 10.98.163.121

$ curl helloworld.vmnamespace.svc:5000/hello
Hello version: v2, instance: helloworld-v2-5b46bc9f84-bd6qj
```
```
$ curl helloworld.vmnamespace.svc:5000/hello -v
*   Trying 10.98.163.121:5000...
* TCP_NODELAY set
* Connected to helloworld.vmnamespace.svc (10.98.163.121) port 5000 (#0)
> GET /hello HTTP/1.1
> Host: helloworld.vmnamespace.svc:5000
> User-Agent: curl/7.68.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-type: text/html; charset=utf-8
< content-length: 60
< server: envoy
< date: Fri, 14 Jan 2022 03:56:35 GMT
< x-envoy-upstream-service-time: 98
< 
Hello version: v2, instance: helloworld-v2-5b46bc9f84-bd6qj
* Connection #0 to host helloworld.vmnamespace.svc left intact
```

# 5-2. A Non-Kubernetes Endpoint
> https://istio.io/latest/blog/2020/workload-entry/

# 5-2-1. Run nginx workload on Virtual Machine as a non-kubernetes Endpoint
```
$ sudo docker pull nginx:1.16.1
$ sudo docker run --rm --name nginx -d -p 80:80 nginx:1.16.1
$ sudo docker exec -it nginx sed -id s/Welcome\ to/Welcome\ to\ VM\'s/g /usr/share/nginx/html/index.html
$ curl -s 127.0.0.1 |grep "<title>.*</title>"
<title>Welcome to VM's nginx!</title>
```
# 5-2-2. Create the service for the non-kubernetes Endpoint in kubernetes cluster
```
$ cat <<EOF | kubectl apply -n vmnamespace -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-vm-svc
  labels:
    app: nginx-vm
spec:
  ports:
  - port: 8080
    targetPort: 80
    name: tcp
  selector:
    app: nginx-vm
EOF
```
```
$ kubectl exec -n vmnamespace -it ubuntu -- nslookup nginx-vm-svc
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	nginx-vm-svc.vmnamespace.svc.cluster.local
Address: 10.108.92.68
```

But at this moment, You can not access to the FQDN of "nginx-vm-svc.vmnamespace.svc.cluster.local". You should bind between the service and IP address of Virtual Machine.
```
$ kubectl exec -n vmnamespace -it ubuntu -- curl nginx-vm-svc.vmnamespace.svc:8080 |grep "<title>.*</title>"
command terminated with exit code 56
```

# 5-2-3. Create WorkloadEntry in kubernetes cluster
WorkloadEntry allows you to describe non-Pod endpoints that should still be part of the mesh, and treat them the same as a Pod. Specify the IP address of the Virtual Machine running nginx container on, so that the "nginx-vm" defined in label of app can be bound for the IP address of 192.168.33.112.
```
$ cat <<EOF | kubectl apply -n vmnamespace -f -
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadEntry
metadata:
  name: nginx-vm-wkle
spec:
  #serviceAccount: mysvcaccount
  address: 192.168.33.112
  labels:
    app: nginx-vm
    instance-id: ubuntu-vm-nginx
EOF
```
```
$ kubectl get -n vmnamespace workloadentry
NAME            AGE   ADDRESS
nginx-vm-wkle   62m   192.168.33.112
```
```
$ kubectl exec -n vmnamespace -it ubuntu -- curl nginx-vm-svc.vmnamespace.svc:8080 |grep "<title>.*</title>"
<title>Welcome to VM's nginx!</title>
```
![kiali](https://github.com/developer-onizuka/serviceMesh/blob/main/kiali.png)

# 5-2-4. LoadBalancing between Pod and WorkloadEntry
First of all, Create deployment which runs nginx in the kubernetes cluster. Please note label used by selecting endpoints is defined as "app" and "nginx-vm".
```
$ cat <<EOF | sudo kubectl apply -n vmnamespace -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-vm-test
spec:
  selector:
    matchLabels:
      app: nginx-vm
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-vm
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1
        ports:
        - containerPort: 80
EOF
```
```
$ kubectl exec -n vmnamespace -it ubuntu -- curl nginx-vm-svc.vmnamespace.svc:8080 |grep "<title>.*</title>"
<title>Welcome to nginx!</title>

$ kubectl exec -n vmnamespace -it ubuntu -- curl nginx-vm-svc.vmnamespace.svc:8080 |grep "<title>.*</title>"
<title>Welcome to VM's nginx!</title>
```


# Option) Create ServiceEntry in kubernetes cluster
You might need ServiceEntry in addition to WorkloadEntry.
ServiceEntry enables adding additional entries into Istio???s internal service registry. "nginx-vm-svc.vmnamespace.svc.cluster.local" in yaml is FQDN which is created by kind of Service in 5-2-2 above. But it might be not accessible as is due to some reasons (by default, older version of Istio locks down outbound traffic), then you could specify the destination FQDN as a Service Entry so that it can access internally inside of mesh.
> https://blog.1q77.com/2020/03/istio-part7/
```
$ cat <<EOF | kubectl apply -n vmnamespace -f -
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: nginx-vm-svce
spec:
  hosts:
  - nginx-vm-svc.vmnamespace.svc.cluster.local
  location: MESH_INTERNAL
  ports:
  - number: 8080
    name: http
    protocol: HTTP
  resolution: STATIC
  workloadSelector:
    labels:
      app: nginx-vm
EOF
```
```
$ kubectl get -n vmnamespace serviceentry
NAME            HOSTS                                            LOCATION        RESOLUTION   AGE
nginx-vm-svce   ["nginx-vm-svc.vmnamespace.svc.cluster.local"]   MESH_INTERNAL   STATIC       62m
```

# 5-2-5. For the Access thru IngressGateway from outside of Mesh
```
$ cat <<EOF | kubectl apply -n vmnamespace -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: vm-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 8080
      name: vm-gateway
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: nginx-vm-vsvc
spec:
  hosts:
  - "*"
  gateways:
  - vm-gateway
  http:
  - match:
    - uri:
        prefix: "/"
    route:
    - destination:
        port:
          number: 8080
        host: nginx-vm-svc
EOF
```
Green Line is the access of http://192.168.33.220 that is thru IngressGateway. Blue Line is the access of curl to VM or Pod with loadbalancing.
![kiali2](https://github.com/developer-onizuka/serviceMesh/blob/main/kiali2.png)




