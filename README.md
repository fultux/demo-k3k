# demo-k3k
This repo contains the different steps and manifests to demonstrate the various aspects of k3k and SUSE virtual clusters 

## 1. Basics

### Enable SUSE Virtual cluster
A Rancher Prime cluster is required to have access to the SUSE Virtual Cluster integration. On the community version you don't have the integration but k3k can be installed on the host cluster with helm. 
* Enable the SUSE virtual cluster extension in Rancher
* Enable Virtual cluster in the downstream host cluster.

### Deploy a k3k cluster from the CLI

* Install the [k3k cli ](https://github.com/rancher/k3k#install-the-k3kcli)
* Access the cluster from the kubeconfig generated through the Rancher AP and try to Create a cluster. 
	```
	k3kcli cluster create <cluster_name>
	```
* Check that when cluster is when connected to a Rancher it tries to use the Rancher API. 
* Delete the cluster
 ```
	k3kcli cluster delete <cluster_name>
 ```
* Access the cluster directly with ssh and use kubeconfig directly from the node. See that you can access the k3k cluster from here. 
* Notice that if you don't provide a namespace it creates one with k3k-[cluster_name]
* the command ``` kubectl cluster-info ``` should return something. but if you try ```kubectl get nodes``` should only return a worker node. 
* Create a namespace and a pod by applying the manifest manifests/demo-pod-nginx.yaml
* Verify that the pod is running on the k3k cluster



 ## 2. The different modes of k3k

k3k supports three operational modes depending on the desired level of isolation, resource footprint, and infrastructure topology:

### 1. Shared Mode (`shared.yaml`)
In **shared** mode, the virtual control plane runs as a pod on the host cluster, but virtual workloads schedule directly onto the host cluster's nodes within the tenant namespace.

* **Deploy the cluster**:
  ```bash
  kubectl create namespace k3k-shared
  kubectl apply -f manifests/shared.yaml
  ```
* **Retrieve the kubeconfig**:
  ```bash
  kubectl get secret -n k3k-shared k3k-k3k-shared-kubeconfig -o jsonpath='{.data.kubeconfig\.yaml}' | base64 -d > kubeconfig-shared.yaml
  ```
* **Behavior**: Workloads scheduled in the virtual cluster will directly translate to pods running in the `k3k-shared` namespace on the host cluster.
---


### 2. Virtual Mode (`virtual.yaml`)
In **virtual** mode, both the control plane and worker agents run in isolated pods on the host cluster. Workloads deployed inside the virtual cluster execute inside the k3k agent pods rather than directly on host nodes.

* **Deploy the cluster**:
  ```bash
  kubectl create namespace k3k-virtual
  kubectl apply -f manifests/virtual.yaml
  ```
* **Retrieve the kubeconfig**:
  ```bash
  kubectl get secret -n k3k-virtual k3k-k3k-virtual-kubeconfig -o jsonpath='{.data.kubeconfig\.yaml}' | base64 -d > kubeconfig-virtual.yaml
  ```

* **Behavior**: Virtual nodes appear in `kubectl --kubeconfig kubeconfig-virtual.yaml get nodes` corresponding to the `agents` count configured in the manifest.

* **Hint**: The cluster might not be able to deploy. Can you troubleshoot the issue ?  


---

### 3. Hosted Control Plane (HCP) Mode (`hcp.yaml`)
In **hcp** mode, only the control plane runs as pods on the host cluster. The API server endpoint is exposed externally (via `LoadBalancer`, `NodePort`, or `Ingress`) so external nodes (VMs, bare-metal machines) can join the cluster.

* **Deploy the cluster**:
  ```bash
  kubectl create namespace k3k-hcp
  kubectl apply -f manifests/hcp.yaml
  ```
* **Retrieve the kubeconfig**:
  ```bash
  kubectl get secret -n k3k-hcp k3k-k3k-hcp-kubeconfig -o jsonpath='{.data.kubeconfig\.yaml}' | base64 -d > kubeconfig-hcp.yaml
  ```
* **Join external nodes**: Retrieve the node token from the cluster secret:
  ```bash
  kubectl get secret -n k3k-hcp k3k-k3k-hcp-token -o jsonpath='{.data.token}' | base64 -d
  ```
  Then launch K3s agent on external machines pointing to the exposed endpoint and token.


### 4. Expose apps outside the cluster

In **shared mode**, workloads running in the virtual cluster are physically scheduled onto the host nodes inside the tenant namespace (`k3k-shared`). Because virtual cluster services are decoupled from the host networking layer, exposing tenant applications externally is accomplished by applying Service or Ingress resources on the **host cluster** that match the pod selectors of the tenant workload.

#### Step 1: Deploy a workload in the virtual cluster
Deploy an application inside the virtual cluster using its kubeconfig:
```bash
kubectl --kubeconfig kubeconfig-shared.yaml create deployment nginx --image=nginx -n external
```

Verify on the host cluster that the pod is running with label `app=nginx` in the tenant namespace:
```bash
kubectl get pods -n k3k-shared -l app=nginx
```

#### Step 2: Expose the workload from the host cluster

Choose one of the exposure strategies using the provided manifests:

* **Option A: NodePort (`expose-shared-nodeport.yaml`)**
  Exposes the workload on static port `30080` across all host cluster nodes.
  ```bash
  kubectl apply -f manifests/expose-shared-nodeport.yaml
  ```
  Test connectivity:
  ```bash
  curl http://<host-node-ip>:30080
  ```

* **Option B: LoadBalancer (`expose-shared-loadbalancer.yaml`)**
  Provisions an external IP via the host cluster's LoadBalancer provider or ServiceLB (Klipper).
  ```bash
  kubectl apply -f manifests/expose-shared-loadbalancer.yaml
  ```
  Test connectivity:
  ```bash
  EXTERNAL_IP=$(kubectl get svc nginx-loadbalancer -n k3k-shared -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  curl http://${EXTERNAL_IP}
  ```

* **Option C: Ingress (`expose-shared-ingress.yaml`)**
  Creates a `ClusterIP` service and routes HTTP requests through the host cluster's Ingress controller using domain-based routing (`nginx-shared.example.com`).
  ```bash
  kubectl apply -f manifests/expose-shared-ingress.yaml
  ```
  Test connectivity:
  ```bash
  curl -H "Host: nginx-shared.example.com" http://<host-ingress-ip>
  ```


## 3. Advanced usage

### 1. Cluster customization
You can configure a shared k3k cluster to use a specific StorageClass for its persistence by defining `spec.persistence.storageClassName`.

To deploy the dedicated StorageClass and the shared cluster:
```bash
kubectl create namespace k3k-custom
kubectl apply -f manifests/storage-class.yaml
kubectl apply -f manifests/k3k-custom.yaml
```

Get the cluster's Kubeconfig
```bash
kubectl get secret -n k3k-custom k3k-k3k-custom-kubeconfig -o jsonpath='{.data.kubeconfig\.yaml}' | base64 -d > kubeconfig-custom.yaml
```

Verify you have access to the selected storage class
``` bash
kubectl --kubeconfig=kubeconfig-custom.yaml get sc
```

In the custom cluster deploy the nginx pod and deploy the ingress
``` bash
kubectl --kubeconfig=kubeconfig-custom.yaml apply -f manifests/demo-pod-nginx.yaml
kubectl --kubeconfig=kubeconfig-custom.yaml apply -f manifests/expose-custom-ingress.yaml
```

 Test connectivity:
  ```bash
  curl -H "Host: nginx-shared.example.com" http://<host-ingress-ip>
  ```

Verify you have an ingress on the custom cluster 
 Test connectivity:
  ```bash
  kubectl get ingress -n demo
  ```

Verify the ingress is present in the host cluster
```bash
kubectl get ing -n k3k-custom
```



### 2. Virtual cluster policies
k3k provides a dedicated `VirtualClusterPolicy` custom resource (`k3k.io/v1beta1`) to govern virtual cluster boundaries. A single `VirtualClusterPolicy` applied in the virtual cluster's namespace configures:

* **Resource Quotas & Limits**: Sets boundaries (`resourceQuota`) and container defaults (`limitRange`).
* **Network Isolation**: Governs automated network policy generation via `disableNetworkPolicy` (set to `false` by default to enforce isolation and prevent cross-tenant/host network access, or `true` to disable network policy creation).
* **Pod Security Standards (PSA)**: Automatically applies `enforce`, `warn`, and `audit` Pod Security Admission standards (`podSecurityStandards`) to workloads within the virtual cluster.

Deploy the complete policy:

```bash
kubectl apply -f manifests/virtualclusterpolicy.yaml
```

### 3. Custom runtime (kata-containers)
Kata Containers provides hardware-isolated microVMs for container workloads. With k3k, Kata Containers is used with **`shared` mode** clusters because virtual workloads are scheduled directly as pods onto host nodes, allowing the host's container runtime to invoke the Kata hypervisor shim.

#### Prerequisites
* Kata Containers installed and configured on the host cluster nodes (e.g. via `kata-deploy`).
* A k3k cluster running in **`shared` mode** (e.g., `manifests/shared.yaml`).

#### Step-by-Step Procedure

1. **Create the `RuntimeClass` on the host cluster**:
   The host cluster must recognize the `kata` runtime handler:
   ```bash
   kubectl apply -f manifests/kata-runtimeclass.yaml
   ```

2. **Deploy the shared k3k cluster and retrieve kubeconfig**:
   ```bash
   kubectl create namespace k3k-shared
   kubectl apply -f manifests/shared.yaml
   kubectl get secret -n k3k-shared k3k-k3k-shared-kubeconfig -o jsonpath='{.data.kubeconfig}' | base64 -d > kubeconfig-shared.yaml
   ```

3. **Create the `RuntimeClass` inside the virtual cluster**:
   The identical `RuntimeClass` must also exist within the virtual cluster:
   ```bash
   kubectl --kubeconfig kubeconfig-shared.yaml apply -f manifests/kata-runtimeclass.yaml
   ```

4. **Deploy the workload inside the virtual cluster**:
   ```bash
   kubectl --kubeconfig kubeconfig-shared.yaml apply -f manifests/kata-pod.yaml
   ```

5. **Verify execution**:
   * In the virtual cluster: `kubectl --kubeconfig kubeconfig-shared.yaml get pods`
   * On the host cluster: check the pod in the `k3k-shared` namespace:
     ```bash
     kubectl get pods -n k3k-shared -o wide
     ```
   * On the host worker node: verify the workload is running inside a Kata microVM process:
     ```bash
     ps aux | grep -E "qemu|cloud-hypervisor|kata"
     ```
