# interview-kiln

## I) How would you deploy a Kubernetes cluster ? How would you handle scalability and auto-scalability ?
<br>

### I.a Deploying a Bare Metal Kubernetes Cluster

To deploy a bare metal Kubernetes cluster, I would utilize kubeadm for its simplicity.
The implementation steps would include:<br>
**Step to deploy bare metal kubernetes cluster:**<br>
- **Requirement:** Ensure all bare metal servers have a compatible OS, resources (RAM and memory) and necessary dependencies installed (Docker, kubectl and kubeadm).
- **Network Configuration:** Creating a flat network for pod communication and configuring firewall rules to allow necessary traffic (e.g., ports 6443 for the API server, 10250 for kubelet).
- **Initialize Control Plane:** Use kubeadm init on the master node to set up the control plane.
- **Join Worker Nodes:** Use kubeadm join to add worker nodes to the cluster.
- **CNI Network:** Deploy a CNI plugin like Calico for networking. It provides network policies and is well-suited for security in a bare metal environment.\
<br>


### I.b Scalability and Auto-Scalability
#### Scalability for Nodes
__Node Sizing and Types:__ Start by categorizing the available bare metal servers into different classes (small, medium, large, CPU optimized, Memory optimized etc.) hence create taint and tolerations based on those node specifications. This allows flexible scaling based on workload requirements.


__Scaler with monitoring:__ Implement robust monitoring solutions (such as Prometheus) to collect metrics related to resource usage (CPU, memory, network I/O). Use these metrics to forecast capacity needs and identify when to scale up or down. Regularly review resource utilization patterns to make informed decisions.
Monitor workloads and manually add or remove nodes based on demand. This can be manually by a script that alerts administrators when certain resource thresholds are met or by adapting tools like cluster autoscaler or Karpenter to work on bare metal.


#### Auto-Scalability for Workloads
For auto-scalability of workloads (pods), Kubernetes provides several mechanisms that can be leveraged:<br>
__Resource Requests and Limits:__ Properly configure resource requests and limits for all containers to ensure the Kubernetes scheduler can make informed decisions about where to place pods. This is crucial for effective auto-scaling, as it allows the Scheduler, HPA or other tools to operate optimally and avoid performance issue such as throtling.<br>
__Custom Metrics:__ The cluster API can be expanded with CRDs to handle the workload and for improve the scaling decisions. Custom metrics tools like Keda can scale beyond the HPA standard metrics, ie, CPU and memory (e.g., request counts, latency, etc ...).

```yaml
apiVersion: kedacore.io/v1alpha1
kind: ScaledObject
metadata:
  name: my-app-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: my-app
    kind: Deployment
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-server.monitoring.svc.cluster.local
        metricName: http_requests_total
        threshold: '100'
        query: sum(rate(http_requests_total[1m]))
```
<br><br><br>

## II) Can you provide persistent storage for your application ? How ?

When deploying a Kubernetes cluster on bare metal environment, providing persistent storage is critical for stateful applications.
According to my opinion an NFS server is the best solution for persistent storage on a bare metal kubernetes cluster as the storage is shared betwen pod as its support `ReadWriteMany`, its straithforward and its cost effective. However challenges might be face concerning scalability and throughput performance.

Here’s how to approach this:

- __NFS Storage Integration:__ Leverage the infrastructure provider's block storage solutions such as NFS if there is any, otherwise crete an NFS server from one of the provider VM.

- __Dynamic Provisioning:__ Utilize Kubernetes Storage Classes to enable dynamic provisioning of persistent volumes. Storage Class can automatically create persistent volumes when needed. This reduces the overhead of managing storage manually and ensures that the storage is correctly configured based on the application’s requirements. Applications can request persistent storage via Persistent Volume Claims. By specifying the desired storage size and class in the PVC, The Kubernetes Container Storage Interface (CSI) will handle the provisioning of the required Persistent Volume.


**Storage class**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage-class
provisioner: nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "false"
reclaimPolicy: Delete
mountOptions:
  - vers=4
```
**PVC**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-dynamic-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-storage-class
  resources:
    requests:
      storage: 5Gi
```



## III) How would you handle connectivity across the cluster ? Will your application be reachable from the Internet ? How will the communication between Kubernetes nodes will be secured ?
<br>

__1. Connectivity across the cluster:__
To ensure seamless communication between pods in a Kubernetes cluster, I would leverage the capabilities of a Container Network Interface (CNI) plugin, such as Calico or Flannel. These plugins allow pods to communicate with each other directly using their IP addresses, creating a flat networking model.

__2. External Accessibility:__
The applications deployed on the cluster can be reachable from internet via the Ingress controller which manages external access to the services within the cluster via routing rules specified in its CRD.

__3. Securing Communication Between Nodes:__

To ensure that communication between Kubernetes nodes is secure, you can implement several strategies:

- __TLS Encryption:__ Enable TLS for all communication within the cluster. Kubernetes supports TLS for API server communication and can be configured to enforce it for all node-to-node traffic. If tls is enabled for node, it also make sens to enabled it for communicating with the ETCD and API server also.


- __Network Policies:__ Use Kubernetes Network Policies to control traffic flow between pods. This allows you to enforce rules that restrict which pods can communicate with each other, minimizing the potential attack surface.

- __VPN for External Access:__ VPN help to access the cluster from an external network securely. This would encrypt all traffic between your external clients and the Kubernetes nodes, ensuring that sensitive data is protected.

- __Istio__ can significantly enhances the security of the cluster by enforcing mutual tls between worklods. I will go with istio only if there is not hight latency requirement since the encryption and decryption of each message increase the latency.

## IV) How would you deploy your applications ?

### Templating vs Patching:
For the deployment I will suggest a gitops approach to avoid manual interaction with the cluster as much as possible.
I will go with helm if there few application or the application specifications are pretty similar betwen enironment. If the application is not templatizable, I'll go with kustomize.\

### CI/CD Integration:
I'll rather use argoCD if there few application to deploy, otherwie Ill go with Fluxcd for its sharding capability.
The github or gitlab runner will be on kubernetes running as pod.
<br>

Here’s a step-by-step process to deploy applications using Kustomize with FluxCD

**Create Base and Overlay Directories:**<br>
Structure your repository with a base directory for common configurations and overlay directories for each environment.

```yaml
kiln-app/
├── base/
│   ├── frontend-deployment.yaml
│   ├── backend-deployment.yaml
│   ├── ingress.yaml
│   └── service.yaml
├── overlays/
│   ├── production/
│   │   └── kustomization.yaml
│   └── staging/
│       └── kustomization.yaml
```
**Define Kustomizations:**<br>
Create a kustomization.yaml file in each overlay directory to customize the base configurations.

**Push to Git:**<br>
Commit your Kustomize configuration to your Git repository.

**Install FluxCD:**<br>
Install FluxCD in your cluster and configure it to watch the Git repository.
```yaml
flux bootstrap github \
  --owner=<GITHUB_USER> \
  --repository=<KILN_REPO> \
  --branch=<BRANCH> \
  --path=./clusters/kiln-cluster \
  --personal
```


**Deploy:**<br>
FluxCD will automatically apply the changes from your Git repository, ensuring that the cluster state matches the desired state.

