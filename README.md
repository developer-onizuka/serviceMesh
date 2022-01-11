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
- Each cluster must have a DNS entry for the service in order for the DNS lookup to succeed, and a request to be successfully sent. This is true even if there are no instances of that service’s pods running in the cluster which makes requests.
- For clusters spanning networks, this can be achieved by exposing the control plane through an Istio gateway which goes thru such as internal load balancers. (But I don't draw the Istio gateway in the figure below, because of no space to draw it and it also gets so complicating to understand.)
- Istio Citadel is for CA (Certificate Authority). See also https://www.youtube.com/watch?v=LmZz8KpSY74.

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
- In order for consumers to reliably call your workload, it’s recommended to declare a Service association. (But I don't draw it in the figure below.)

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

# 5-1. How to join virtual machines into the mesh
https://istio.io/latest/docs/setup/install/virtual-machine/

# 5-1-1. Master node in kubernetes cluster

(1) Set some environment valuse.
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
```
```
$ kubectl get services -n istio-system
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.100.219.251   192.168.121.231   15021:32740/TCP,15443:32261/TCP,15012:30627/TCP,15017:31259/TCP   103s
istio-ingressgateway    LoadBalancer   10.111.23.101    192.168.121.230   15021:32569/TCP,80:30858/TCP,443:31696/TCP                        4m33s
istiod                  ClusterIP      10.99.43.125     <none>            15010/TCP,15012/TCP,443/TCP,15014/TCP                             5m37s

```
```
$ kubectl apply -n istio-system -f istio-1.12.1/samples/multicluster/expose-istiod.yaml
gateway.networking.istio.io/istiod-gateway created
virtualservice.networking.istio.io/istiod-vs created
```
```
$ kubectl apply -n istio-system -f istio-1.12.1/samples/multicluster/expose-services.yaml
gateway.networking.istio.io/cross-network-gateway created
```
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

(4) Check if third party tokens are enabled in your cluster.
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
(5) Make a Yaml file for workloadgroup.
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
(6) Generate the istio-token and Copy it to the VM which you want to join into the cluster.
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

# 5-1-2. Virtual Machine

```
$ tar xvfz vmintegration.tar.gz
$ cd vmintegration
$ sudo mkdir -p /etc/certs
$ sudo cp root-cert.pem /etc/certs/
$ sudo mkdir -p /var/run/secrets/tokens
$ sudo cp istio-token /var/run/secrets/tokens/
```

```
$ curl -LO https://storage.googleapis.com/istio-release/releases/1.12.1/deb/istio-sidecar.deb
$ sudo dpkg -i istio-sidecar.deb
```

```
$ sudo cp cluster.env /var/lib/istio/envoy/
$ sudo cp mesh.yaml /etc/istio/config/mesh
$ sudo sh -c 'cat hosts >> /etc/hosts'
```
```
$ cat /etc/hosts |grep istiod
192.168.121.221 istiod.istio-system.svc
```

```
$ sudo mkdir -p /etc/istio/proxy
$ sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
```

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


