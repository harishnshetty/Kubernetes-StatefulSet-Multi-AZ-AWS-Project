# Kubernetes-StatefulSet-Multi-AZ-AWS-Project


Launch the cluster:

```bash
eksctl create cluster -f eks-config.yaml
```

> Although we specify **3 AZs**, we request **4 worker nodes**. EKS ensures each AZ gets at least one node, with one AZ receiving a second node.

Check node distribution:

```bash
kubectl get nodes --show-labels | grep topology.kubernetes.io/zone
```

---

### 4. Install the Amazon EBS CSI Driver

#### Why Is the CSI Plugin Required?

Kubernetes uses the **Container Storage Interface (CSI)** standard to interact with external storage providers. Without the **Amazon EBS CSI driver**:

* EBS volumes **cannot be provisioned or attached** dynamically
* Stateful workloads like MySQL or Redis will remain **in `Pending` state**
* Volume expansion, lifecycle control, and recovery features won't work



Install the CSI plugin:

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.44"
```

Verify successful deployment:

```bash
kubectl get daemonset ebs-csi-node -n kube-system
kubectl get deploy ebs-csi-controller -n kube-system
```

#### Plugin Components

* **`ebs-csi-node`**
  A **DaemonSet** that runs on every worker node. It handles **attaching and mounting** EBS volumes so pods can access data locally.

* **`ebs-csi-controller`**
  A **central Deployment** that manages volume **provisioning and lifecycle**, coordinating with the AWS API.



## Step 0: Create Namespace

```bash
kubectl create namespace mysql-ha
```

```bash
kubectl config set-context --current --namespace=mysql-ha
```

> If you ever want to switch back to the default (`default`) namespace, run:
>
> ```bash
> kubectl config set-context --current --namespace=default
 ```


  ## Step 1: Create a StorageClass

We use a custom EBS-backed StorageClass that provisions **gp3 volumes** and waits for the pod to be scheduled before selecting the AZ.

Apply it:

```bash
kubectl apply -f 01-sc.yaml
```


---

## Step 3: Create a Headless Service


Apply it:

```bash
kubectl apply -f 02-mysql-hs.yaml
```
## Step 4: Deploy the MySQL StatefulSet


Apply the StatefulSet:

```bash
kubectl apply -f 03-mysql-sts.yaml
```


## Step 6: Observe StatefulSet Behavior

```bash
kubectl get pods -o wide
kubectl get pvc
kubectl get pv
```


    **If You Want the 4th Pod to Be Scheduled (Soft Anti-Affinity Rule)**

    By default, using `requiredDuringSchedulingIgnoredDuringExecution` (a **hard rule**) ensures pods are strictly distributed — one per AZ. If your StatefulSet has **more pods than AZs**, the extras will remain **pending** due to this constraint.

    To allow scheduling of the extra pod(s) even if anti-affinity **cannot** be perfectly satisfied, use a **soft rule** instead:

    ```yaml
    affinity:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100  # Preference weight (1–100). Higher means stronger preference.
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: mysql
              topologyKey: topology.kubernetes.io/zone
    ```

    ---


    ### Single AZ Scenario:

If you're in a **single AZ** environment, replace:

```yaml
topologyKey: topology.kubernetes.io/zone
```

with:

```yaml
topologyKey: kubernetes.io/hostname
```