# 🚀 100 Kubernetes Assignments: Beginner Hands-on Guide

This guide provides detailed, step-by-step execution and verification instructions for the first five Kubernetes assignments.

---

## #1: Deploying Your First Application
**Objective:** Learn to provision an AKS cluster and deploy a simple Nginx web server using `kubectl`.

### Step-by-Step Instructions
1.  **Create Resource Group:** Establish a logical container for your Azure resources.
    ```bash
    az group create --name myK8sResourceGroup --location centralus
    ```
2.  **Provision AKS Cluster:** (Using B-Series for Free Trial compatibility).
    ```bash
    az aks create --resource-group myK8sResourceGroup --name my-first-aks-cluster --node-count 2 --node-vm-size Standard_B2s --generate-ssh-keys
    ```
3.  **Merge Credentials:** Download the `kubeconfig` to your local machine.
    ```bash
    az aks get-credentials --resource-group myK8sResourceGroup --name my-first-aks-cluster
    ```
4.  **Create Deployment:** Deploy the Nginx image.
    ```bash
    kubectl create deployment my-nginx --image=nginx:latest
    ```
5.  **Expose Application:** Create a NodePort service to allow traffic.
    ```bash
    kubectl expose deployment my-nginx --port=80 --type=NodePort
    ```
6.  **Verify:** ```bash
    kubectl get pods    # Status should be 'Running'
    kubectl get svc     # Note the assigned NodePort (e.g., 3xxxx)
    ```

---

## #2: Understanding Kubernetes Pods
**Objective:** Define a Pod via YAML and learn how to "exec" into it for troubleshooting.

### Step-by-Step Instructions
1.  **Create Manifest:** Generate the `pod.yaml` file with labels for future service discovery.
    ```bash
    cat > pod.yaml << 'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-pod
      labels:
        app: busybox-app
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo Hello Kubernetes && sleep 3600']
    EOF
    ```
2.  **Deploy:** Apply the YAML to the cluster.
    ```bash
    kubectl apply -f pod.yaml
    ```
3.  **Inspect Logs:** View the "Hello Kubernetes" output.
    ```bash
    kubectl logs my-pod
    ```
4.  **Interactive Debugging:** Access the container shell.
    ```bash
    kubectl exec -it my-pod -- sh
    # Inside the pod: run 'ls' or 'whoami', then type 'exit'
    ```

---

## #3: Exploring Kubernetes Namespaces
**Objective:** Isolate resources logically within the same physical cluster.

### Step-by-Step Instructions
1.  **Create Namespace:**
    ```bash
    kubectl create namespace dev
    ```
2.  **Targeted Deployment:** Run a pod specifically in the new namespace.
    ```bash
    kubectl run nginx-dev --image=nginx -n dev
    ```
3.  **Verification:** Compare default vs dev namespaces.
    ```bash
    kubectl get pods          # Should NOT show nginx-dev
    kubectl get pods -n dev   # Should show nginx-dev
    ```
4.  **Switch Context:** Set `dev` as the default to save typing.
    ```bash
    kubectl config set-context --current --namespace=dev
    ```

---

## #4: Working with Kubernetes Services
**Objective:** Understand how Services provide stable network endpoints (ClusterIP vs NodePort).

### Step-by-Step Instructions
1.  **Prepare Replicas:** Create 2 instances of an app.
    ```bash
    kubectl create deployment web --image=nginx --replicas=2
    ```
2.  **Expose Internally:** Create a `ClusterIP` service (accessible only inside the cluster).
    ```bash
    kubectl expose deployment web --port=80 --target-port=80 --name=web-svc
    ```
3.  **Connectivity Test:** Use a temporary "test" pod to hit the service.
    ```bash
    kubectl run test --image=busybox --rm -it --restart=Never -- wget -qO- http://web-svc
    ```
4.  **Patch for External Access:** Change the service type to `NodePort`.
    ```bash
    kubectl patch svc web-svc -p '{"spec":{"type":"NodePort"}}'
    ```

---

This runbook provides a practical guide to managing application settings using Kubernetes **ConfigMaps**. By separating configuration from your container images, you make your applications portable across different environments (development, staging, production) without needing to rebuild.

---

## 🛠️ Runbook: Kubernetes ConfigMap Management

### 1. Create ConfigMaps
ConfigMaps store non-confidential data in key-value pairs. You can create them imperatively using `kubectl`.

**A. From Literals**
Useful for simple environment variables.
```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_PORT=8080
```

**B. From Files**
Useful for configuration files like `.properties`, `.ini`, or `.yaml`.
```bash
echo 'LOG_LEVEL=info' > app.properties
kubectl create configmap file-config --from-file=app.properties
```

---

### 2. Inspecting Your Configuration
Before deploying a Pod, verify that the data was stored correctly.

* **Summary view:** `kubectl get configmap`
* **Detailed view:** `kubectl describe configmap app-config`

---

### 3. Consume ConfigMaps in a Pod


#### **Option A: Environment Variables**
This method injects specific keys into the container's shell environment. Save this as `configmap-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
    - name: test-container
      image: nginx
      env:
        - name: ENV_SETTING
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
```

#### **Option B: Volume Mounts**
This method mounts the ConfigMap as a file inside the container. Save this as `volume-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
    - name: test-container
      image: nginx
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: file-config
```

---

### 4. Verification
To ensure your application can actually "see" the data, use the `exec` command to look inside the running Pod.

**Check Environment Variables:**
```bash
kubectl exec -it env-pod -- env | grep APP
```

**Check Mounted Files:**
```bash
kubectl exec -it volume-pod -- ls /etc/config
kubectl exec -it volume-pod -- cat /etc/config/app.properties
```

---

### 💡 Key Takeaways
| Feature | Environment Variables | Volume Mounts |
| :--- | :--- | :--- |
| **Best For** | Simple flags, ports, and IDs | Large config files, certificates |
| **Updates** | Requires Pod restart to update | Updates automatically (with delay) |
| **Visibility** | Visible in `env` output | Visible as a file in the filesystem |

> **Note:** ConfigMaps are **not** encrypted. For sensitive data like passwords or API keys, always use **Kubernetes Secrets** instead.


-------------------------------------------
## Runbook #6: Managing Secrets in Kubernetes (Beginner)

This runbook guides you through the fundamental operations of managing sensitive data within a Kubernetes cluster. While ConfigMaps are for non-sensitive configuration, **Secrets** are specifically designed to hold small amounts of sensitive data, such as passwords, tokens, or keys.

---

### 🟢 Objectives
* Distinguish between **ConfigMaps** (non-sensitive) and **Secrets** (sensitive).
* Create, decode, and consume Kubernetes Secrets.
* Implement best practices to avoid plain-text exposure in YAML files.

### 🛠 Prerequisites
* `kubectl` CLI installed and configured.
* Basic understanding of Kubernetes Pods and ConfigMaps.

---

### 📋 Step-by-Step Instructions

#### 1. Create a Secret
Unlike ConfigMaps, Secrets are often created via the CLI to avoid putting passwords directly into a file on your disk. Use the `generic` type for Opaque secrets (the most common type).

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=s3cr3t
```

#### 2. Inspect the Secret
When you view a Secret's YAML, you will notice the values are not human-readable. They are **Base64 encoded**.

```bash
kubectl get secret db-secret -o yaml
```

> [!IMPORTANT]  
> **Base64 is NOT encryption.** It is a transformation format. Anyone with access to the YAML can easily decode these values.

#### 3. Decode a Value
To verify the content, you can extract a specific key and pipe it to the base64 decoder.

```bash
kubectl get secret db-secret -o jsonpath='{.data.DB_PASS}' | base64 --decode
```

#### 4. Inject as Environment Variables
To use these secrets in an application, reference them in your Pod manifest. This keeps credentials out of your container image.

**File:** `secret-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: app-container
    image: nginx
    envFrom:
    - secretRef:
        name: db-secret
```
**Apply:** `kubectl apply -f secret-pod.yaml`

#### 5. Mount as Volume
For better security, mount secrets as files. This allows the application to read them from the filesystem, and the files are automatically updated if the Secret changes.

**File:** `secret-volume-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol-pod
spec:
  containers:
  - name: app-container
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: "/etc/secrets"
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
```
**Apply:** `kubectl apply -f secret-volume-pod.yaml`

#### 6. Restrict Access
Because Secrets are easily decoded, you must use **Role-Based Access Control (RBAC)** to limit who can "get" or "describe" secrets. You can test permissions using the `auth can-i` command:

```bash
# Check if an anonymous user can see secrets (Should be 'no')
kubectl auth can-i get secrets --as=system:anonymous
```

---

### 💡 Summary Table: ConfigMap vs. Secret

| Feature | ConfigMap | Secret |
| :--- | :--- | :--- |
| **Data Type** | Non-sensitive (URLs, feature flags) | Sensitive (Passwords, API keys) |
| **Storage** | Plain text in etcd | Base64 encoded (and ideally encrypted at rest) |
| **Max Size** | 1MiB | 1MiB |

---

### ✅ Conclusion
By using Kubernetes Secrets, you decouple sensitive credentials from your application code and container images. In a production environment, always ensure:
1.  **RBAC** is strictly configured.
2.  **Encryption at Rest** is enabled for the Kubernetes underlying data store (etcd).
3.  You consider external secret managers (like HashiCorp Vault or AWS Secrets Manager) for advanced needs.


---------------------------------------

Scaling applications manually is one of the most fundamental skills in Kubernetes. It allows you to respond to sudden traffic spikes or reduce resource consumption when demand is low.

Here is a breakdown of your run book with a bit of extra context to help you understand what’s happening behind the scenes.

---

## 🏗️ Core Concept: The Hierarchy
Before running the commands, it is helpful to visualize how Kubernetes organizes these resources. When you scale a **Deployment**, you aren't talking to the Pods directly. You are talking to a **ReplicaSet**.



* **Deployment:** Manages the rollout and strategy.
* **ReplicaSet:** Ensures the exact number of specified Pods are running.
* **Pod:** The actual container running your application.

---

## 🛠️ Step-by-Step Execution

### 1. Create the Initial Deployment
Start by creating a single instance of the Nginx web server.
```bash
kubectl create deployment scale-demo --image=nginx
```

### 2. Scale Up
Need more power? Increase the count to 5. The control plane will immediately notice the "desired state" has changed and start spinning up new Pods.
```bash
kubectl scale deployment scale-demo --replicas=5
```

### 3. Monitor the Changes
Use the `-w` (watch) flag to see the Pods transition from `Pending` to `ContainerCreating` to `Running`.
```bash
kubectl get pods -w
```

### 4. Inspect the Controller
The **ReplicaSet (RS)** is the "middleman" that does the heavy lifting of scaling. You’ll see the `DESIRED`, `CURRENT`, and `READY` counts here.
```bash
kubectl get rs
```

### 5. Test Self-Healing
Kubernetes is declarative. If you manually delete a Pod, the ReplicaSet will notice the current count is 4, but the desired count is 5, and it will **immediately** trigger a new Pod creation.
```bash
# Get the pod name from 'kubectl get pods'
kubectl delete pod <pod-name>
```
 
To save resources, bring the count back down to 2. Kubernetes will gracefully terminate the oldest or most redundant Pods.
```bash
kubectl scale deployment scale-demo --replicas=2
```

---

## 💡 Pro-Tips
* **Declarative Scaling:** While `kubectl scale` is great for quick changes, in a real-world GitOps workflow, you would usually update the `replicas:` field in your **YAML manifest** and run `kubectl apply -f deployment.yaml`.
* **The "Why":** Manual scaling is perfect for scheduled events (like a planned sale), but for unpredictable traffic, you'll eventually want to look into the **Horizontal Pod Autoscaler (HPA)**.

Would you like to see how to perform these same steps using a YAML configuration file instead of the command line?

---------------------------------------------------

 Here’s a clear run book for “Rolling Updates and Rollbacks” in Kubernetes:

---

# 8.lling Updates and Rollbacks – Run Book

## Objectives
- Perform a rolling update on a Deployment
- Monitor the rollout status
- Roll back to a previous version

## Prerequisites
- A Deployment running on your cluster
- kubectl CLI configured for your cluster

---

## Step-by-Step Instructions

### 1. Create Initial Deployment
Deploy nginx version 1.21:
```sh
kubectl create deployment rollout-demo --image=nginx:1.21
```

### 2. Update the Image
Update to nginx 1.25 (triggers a rolling update):
```sh
kubectl set image deployment/rollout-demo nginx=nginx:1.25
```

### 3. Watch Rollout Status
Monitor the progress of the rolling update:
```sh
kubectl rollout status deployment/rollout-demo
```

### 4. View Rollout History
List the revision history of the deployment:
```sh
kubectl rollout history deployment/rollout-demo
```

### 5. Rollback to Previous Version
Undo the last rollout:
```sh
kubectl rollout undo deployment/rollout-demo
```

### 6. Verify Version
Confirm the Pod is running the previous image version:
```sh
kubectl describe deployment rollout-demo | grep Image
```

---

## ✅ Conclusion
Rolling updates and rollbacks provide zero-downtime deployments and a safety net for bad releases—core capabilities for production workloads.

This is a solid, functional run book. It covers the essential "What" and "How" of resource management in Kubernetes. To make this truly "production-ready," I’ve polished the formatting and added a few critical commands that help in real-world troubleshooting, such as monitoring actual usage versus defined limits.

---

Here is the complete, code-ready Run Book. This version includes the exact YAML manifests we used, so you can re-run this entire simulation in any new environment or namespace.

---

# 🚀 Run Book: Kubernetes Resource Management (with Code)

## 1. Prerequisites
* **Namespace:** `default` (or your active namespace)
* **Tooling:** `kubectl` and a running K8s cluster (minikube, kind, or cloud-based)

---

## 2. Simulation A: Resource Enforcement (OOMKilled)
This demonstrates how Kubernetes protects a node by killing a container that exceeds its memory limit.

### File: `oom-demo.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
  - name: oom-container
    image: polinux/stress
    resources:
      limits:
        memory: "100Mi"
      requests:
        memory: "50Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

### Execution & Evaluation
```bash
# 1. Apply the pod
kubectl apply -f oom-demo.yaml

# 2. Watch the crash happen
kubectl get pod oom-demo -w

# 3. Prove it was an OOM Kill
kubectl describe pod oom-demo | grep -A 3 "Last State"
```
**Outcome:** `Last State: Terminated`, `Reason: OOMKilled`.

---

## 3. Simulation B: Namespace Governance (LimitRange)
This ensures every pod has a "safety net" even if the developer doesn't include one in their YAML.

### File: `limitrange.yaml`
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-cpu-limit-range
spec:
  limits:
  - type: Container
    max: 
      cpu: "800m"
      memory: "1Gi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "200m"
      memory: "256Mi"
```

### File: `no-resources-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-resources-pod
spec:
  containers:
  - name: nginx-container
    image: nginx:alpine
```

### Execution & Evaluation
```bash
# 1. Apply the policy
kubectl apply -f limitrange.yaml

# 2. Deploy the "naked" pod
kubectl apply -f no-resources-pod.yaml

# 3. Verify the injection
kubectl describe pod no-resources-pod | grep -E "Requests:|Limits:|cpu:|memory:"
```
**Outcome:** The pod will show `512Mi` limits despite the YAML being empty.

---

## 4. Simulation C: Rejecting Over-Provisioning
This demonstrates the "Bouncer" effect where Kubernetes rejects a manifest that violates the `max` constraint.

### File: `too-big-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: too-big-pod
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    resources:
      limits:
        memory: "2Gi" # Violation: Max is 1Gi
```

### Execution & Evaluation
```bash
# Attempt to create
kubectl apply -f too-big-pod.yaml
```
**Outcome:** `Error from server (Forbidden): ... maximum memory usage per Container is 1Gi, but limit is 2Gi.`

---

## 5. Cheat Sheet: Operational Commands

| Command | Purpose |
| :--- | :--- |
| `kubectl top nodes` | Check real-time CPU/RAM pressure on hardware. |
| `kubectl top pods` | Identify which pods are currently consuming the most. |
| `kubectl get pods -A` | Look for `CrashLoopBackOff` status (common symptom of resource issues). |
| `kubectl describe node | grep -A 10 "Allocated"` | View total "Bank Balance" of requests vs limits. |

---

## ✅ Final Conclusion
You have now implemented a **three-tier defense** for your cluster:
1.  **Requests:** Predictable scheduling.
2.  **Limits:** Hard enforcement to prevent node-wide crashes.
3.  **LimitRanges:** Global policies to ensure no pod is left unmanaged.

-------------------------------------


## Runbook: Mastering Kubernetes Labels and Selectors

Labels are key-value pairs attached to objects (like Pods). They don't provide direct functionality to the core system but are crucial for **organizing** and **selecting** subsets of objects.

---

### 1. Apply Labels via YAML and CLI
While `kubectl run` is quick, using YAML is the standard for production environments.

**The YAML Approach (`pod.yaml`):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
  labels:
    app: web
    env: dev
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.25
```

**Execution:**
* **Create via YAML:** `kubectl apply -f pod.yaml`
* **Create via CLI:** `kubectl run labeled-pod --image=nginx --labels=app=web,env=dev,tier=frontend`

---

### 2. Filtering Resources with Selectors
Selectors allow you to query specific resources based on their labels.

* **Equality-based:** Find pods where `app` is `web`.
    `kubectl get pods -l app=web`
* **Set-based:** Find pods where `env` is either `dev` or `staging`.
    `kubectl get pods -l 'env in (dev, staging)'`
* **Multiple Selectors:** Filter by both `app` and `env`.
    `kubectl get pods -l app=web,env=dev`



---

### 3. Modifying Labels at Runtime
You can update labels on the fly without restarting the Pod.

* **Add/Update a label:**
    `kubectl label pod labeled-pod version=v1`
* **Overwrite an existing label:**
    `kubectl label pod labeled-pod env=prod --overwrite`
* **Remove a label:** (Note the dash `-` after the key)
    `kubectl label pod labeled-pod version-`

---

### 4. Service Label Matching
Services use selectors to define which Pods should receive traffic. This decoupling allows you to swap backend Pods seamlessly.

**The Service YAML (`service.yaml`):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web    # This must match the label on your Pod
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

**Execution:**
* **Create via YAML:** `kubectl apply -f service.yaml`
* **Create via CLI:** `kubectl expose pod labeled-pod --port=80 --name=web-service --selector=app=web`

---

### 5. Verification and Troubleshooting
To ensure the Service has successfully "found" your Pods, check the **Endpoints**.

* **Check Endpoints:**
    `kubectl get endpoints web-service`
    > **Note:** If the `ENDPOINTS` column is `<none>`, your Service selector does not match any Pod labels.

* **Show all labels in a list:**
    `kubectl get pods --show-labels`

---

### Summary Checklist
| Action | Command / Logic |
| :--- | :--- |
| **Assign** | Use `metadata.labels` in YAML. |
| **Filter** | Use `-l key=value` with `kubectl get`. |
| **Link** | Ensure `spec.selector` in Service matches Pod labels. |
| **Debug** | Check `kubectl get endpoints` to verify connectivity. |

> **Pro Tip:** Use labels for versioning (e.g., `version: v1`, `version: v2`) to perform **Canary Deployments** by simply updating the Service selector to point to the new version.


----------------------------------------

## Runbook: Managing Persistent Storage in Kubernetes

This runbook guides you through the lifecycle of **PersistentVolumes (PV)** and **PersistentVolumeClaims (PVC)**. By the end, you will have a Pod writing to storage that persists even if the Pod is deleted.

---

### 1. Create the PersistentVolume (PV)
The PV is the actual "physical" storage resource in the cluster. In this example, we use `hostPath`, which uses a directory on the worker node.

**`pv.yaml`**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

**Action:**
```bash
kubectl apply -f pv.yaml
```

---

### 2. Create the PersistentVolumeClaim (PVC)
The PVC is a request for storage. Kubernetes looks for a PV that matches the claim's requirements and binds them together.

**`pvc.yaml`**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

**Action:**
```bash
kubectl apply -f pvc.yaml
```

---

### 3. Verify the Binding
Before moving forward, ensure the claim has successfully found and "bound" to the volume.

**Action:**
```bash
kubectl get pv,pvc
```
> **What to look for:** The status for both should be `Bound`.

---

### 4. Mount PVC in a Pod
Now we create a Pod that uses the PVC. We mount it to the directory `/data` inside the container.

**`pvc-pod.yaml`**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/data"
          name: task-pv-storage
```

**Action:**
```bash
kubectl apply -f pvc-pod.yaml
```

---

### 5. Write and Verify Data Persistence
To prove the storage is persistent, we will write data, kill the Pod, and check if the data is still there.

**Step 5a: Write the file**
```bash
kubectl exec -it pvc-pod -- sh -c 'echo "Storage is working!" > /data/test.txt'
```

**Step 5b: Delete and Recreate the Pod**
```bash
kubectl delete pod pvc-pod
kubectl apply -f pvc-pod.yaml
```

**Step 5c: Verify the file survives**
```bash
kubectl exec -it pvc-pod -- cat /data/test.txt
```
> **Expected Output:** `Storage is working!`

---

### Summary Table: PV vs. PVC

| Feature | PersistentVolume (PV) | PersistentVolumeClaim (PVC) |
| :--- | :--- | :--- |
| **Role** | The Resource (The "Hard Drive") | The Request (The "Ticket") |
| **Scope** | Cluster-level | Namespace-level |
| **Created by** | Administrator (or Provisioner) | Developer / User |
| **Lifecycle** | Exists independently of Pods | Bound to a specific PV |

---

### Troubleshooting Tips
* **Pending PVC:** If your PVC stays in `Pending` state, check if the `accessModes` and `storageClassName` match exactly between the PV and PVC.
* **Permissions:** If using `hostPath`, ensure the worker node has permissions to write to the specified directory (e.g., `/mnt/data`).

----------------------------------------------

This runbook provides the necessary YAML manifests and a structured guide to mastering **Jobs** and **CronJobs** in Kubernetes.

---

## 🏗️ Part 1: Manifest Files

Before executing the steps, create these two files in your working directory.

### `pi-job.yaml`
This Job calculates $\pi$ to 2000 places using a Perl container.
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

### `date-cronjob.yaml`
This CronJob runs every minute to print the current timestamp.
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: date-job
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: date-container
            image: busybox:1.28
            command:
            - /bin/sh
            - -c
            - date; echo "Hello from Kubernetes CronJob"
          restartPolicy: OnFailure
```

---

## 🚀 Part 2: Step-by-Step Execution

### 1. Create a One-Shot Job
Deploy the Pi calculation job. Unlike a standard Pod, a Job is designed to terminate successfully rather than run forever.
```bash
kubectl apply -f pi-job.yaml
```

### 2. Monitor Job Progress
Observe the status transitions from `0/1` (Running) to `1/1` (Completed).
```bash
kubectl get job pi -w
```

### 3. View Results (Logs)
Since the Pod has finished its task, we check the logs to see the calculated value of $\pi$.
```bash
kubectl logs -l job-name=pi
```

### 4. Schedule a Recurring CronJob
Apply the CronJob to the cluster. This acts as a controller that creates a new Job object based on the schedule provided (`* * * * *`).
```bash
kubectl apply -f date-cronjob.yaml
```

### 5. Verify & Manual Trigger
Check if the CronJob is active. If you don't want to wait a full minute for the next cycle, you can trigger a manual run using the CronJob as a template.
```bash
# List active schedules
kubectl get cronjobs

# Trigger a manual execution
kubectl create job --from=cronjob/date-job manual-run
```

### 6. Clean Up
Kubernetes does not automatically delete completed Jobs by default (to allow you to inspect logs). Use the following commands to keep your cluster tidy.
```bash
kubectl delete job pi
kubectl delete cronjob date-job
kubectl delete job manual-run
```

---

## 💡 Key Differences Summary

| Feature | Job | CronJob |
| :--- | :--- | :--- |
| **Usage** | One-off tasks (e.g., database migration). | Repeated tasks (e.g., backups, reports). |
| **Lifecycle** | Runs once until completion. | Creates Jobs on a time-based schedule. |
| **Restart Policy** | Usually `Never` or `OnFailure`. | Inherited by the Jobs it creates. |

> **Note:** If a Job fails, Kubernetes will retry the execution until it hits the `backoffLimit` defined in your YAML.

------------------------------------------------

This runbook outlines the process for setting up the **Kubernetes Dashboard**, providing both the execution steps and the necessary YAML configuration to grant administrative access.

-----

## 🛠️ Configuration File: `dashboard-admin.yaml`

To access the dashboard with full permissions, you need to create a **ServiceAccount** and bind it to the **cluster-admin** role. Save the following content as `dashboard-admin.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

-----

## 🚀 Execution Steps

### 1\. Deploy the Dashboard

Run the official deployment script provided by the Kubernetes maintainers:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### 2\. Apply Admin Permissions

Create the service account and role binding using the YAML provided above:

```bash
kubectl apply -f dashboard-admin.yaml
```

### 3\. Generate Access Token

The dashboard requires a bearer token for login. Generate one for the `admin-user`:

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

> **Note:** Copy and save this token string; you will need it to log in.

### 4\. Start the Proxy

Since the dashboard is not exposed to the public internet by default, use `kubectl proxy` to create a secure tunnel:

```bash
kubectl proxy
```

*Keep this terminal window open.*

### 5\. Access the UI

Open your web browser and navigate to the following URL:
[http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/](https://www.google.com/search?q=http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)

  * Select **Token** as the login method.
  * Paste the token generated in **Step 3**.

-----

## 📈 Post-Deployment Checklist

Once logged in, you should verify you can see the following resources:

  * **Workloads:** Check that your Pods and Deployments are visible.
  * **Services:** Ensure internal networking is correctly mapped.
  * **Logs:** Click on a specific Pod to view real-time log output directly in the browser.

-----

**Warning:** Giving `cluster-admin` access to a dashboard user is fine for a local learning environment (like Minikube), but for production environments, you should follow the principle of least privilege and use more restrictive roles.

------------------------------------------------

This runbook provides a clear path for managing non-identifying metadata in Kubernetes. Below, I have refined the instructions and provided the equivalent **YAML manifests** so you can manage these resources declaratively, which is the standard for production environments.

---

## 🛠️ Annotations vs. Labels: The Quick Difference

| Feature | Labels | Annotations |
| :--- | :--- | :--- |
| **Purpose** | Identifying & Grouping (Selection) | Metadata & Tooling (Information) |
| **Queryable?** | **Yes** (via `-l` selectors) | **No** |
| **Max Size** | 63 characters | 256 KB (Much larger) |
| **Example** | `env: production`, `tier: frontend` | `build-id: 456`, `contact: dev-ops@org.com` |

---

## 📄 YAML Implementation
Instead of using multiple `kubectl` commands, you can define your annotations directly in a manifest.

### `deployment-annotated.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: annotated-app
  namespace: default
  # Metadata for the Deployment object itself
  annotations:
    build-version: "1.2.3"
    git-commit: "abc123"
    kubernetes.io/change-cause: "Initial deployment"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: annotated-app
  template:
    metadata:
      labels:
        app: annotated-app
      # You can also add annotations to the Pods created by this deployment
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "80"
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

---

## 🚀 Step-by-Step Execution

### 1. Apply the Manifest
```bash
kubectl apply -f deployment-annotated.yaml
```

### 2. Add or Update Annotations Imperatively
If you need to add a "last modified by" note without editing the YAML:
```bash
kubectl annotate deployment annotated-app last-modified-by='Janedoe'
```

### 3. Verify the Metadata
To see the full list of annotations applied to your resource:
```bash
kubectl get deployment annotated-app -o jsonpath='{.metadata.annotations}'
```

### 4. Record a Change Cause
The `kubernetes.io/change-cause` annotation is special. It populates the **CHANGE-CAUSE** column in your rollout history.
```bash
kubectl annotate deployment annotated-app kubernetes.io/change-cause="Scaling up for traffic"
kubectl rollout history deployment/annotated-app
```

### 5. Cleanup (Remove) an Annotation
To remove an annotation, use the key followed by a dash (`-`):
```bash
kubectl annotate deployment annotated-app git-commit-
```

---

> **Note on `--record`**: You might see the `--record` flag in older tutorials (Step 4 in your list). Be aware that this flag is **deprecated** in newer Kubernetes versions. The best practice now is to manually add the `kubernetes.io/change-cause` annotation or use a CI/CD tool to inject it.

------------------------------------------------

## Runbook: Configuring Kubernetes Health Probes

This runbook guides you through implementing self-healing and traffic management using **Liveness**, **Readiness**, and **Startup** probes.

---

### 1. Create Pod with Liveness Probe
A **Liveness Probe** determines if a container is running. If the probe fails, Kubernetes kills the container and starts a new one based on the `restartPolicy`.

**`liveness-pod.yaml`**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
  labels:
    test: liveness
spec:
  containers:
  - name: liveness-container
    image: nginx
    ports:
    - containerPort: 80
    # Command to create a dummy health file on startup
    command: ["/bin/sh", "-c", "touch /usr/share/nginx/html/healthz; nginx -g 'daemon off;'"]
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
```

**Action:**
```bash
kubectl apply -f liveness-pod.yaml
```

---

### 2. Simulate Failure & Watch Restarts
By removing the file the probe expects, we trigger a container "unhealthy" state.

**Action:**
1. **Delete the health endpoint:**
   ```bash
   kubectl exec -it probe-demo -- rm /usr/share/nginx/html/healthz
   ```
2. **Watch the recovery:**
   ```bash
   kubectl get pod probe-demo -w
   ```
   *Observation: You will see the "RESTARTS" count increment as the probe fails and Kubernetes restarts the container to restore health.*

---

### 3. Add Readiness Probe
A **Readiness Probe** determines if a container is ready to service requests. If it fails, the Pod’s IP address is removed from all Services.

**`readiness-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: readiness-test
  template:
    metadata:
      labels:
        app: readiness-test
    spec:
      containers:
      - name: web-container
        image: nginx
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Action:**
```bash
kubectl apply -f readiness-deployment.yaml
```

---

### 4. Check Endpoints
Verify that Kubernetes only routes traffic to Pods that have passed their readiness check.

**Action:**
```bash
# Check the status (Look for 1/1 READY)
kubectl get pods -l app=readiness-test

# View the Service endpoints
kubectl get endpoints readiness-demo
```
> **Note:** If you haven't created a Service yet, the endpoints list will be empty. The probe ensures that as soon as a Service is created, only "Ready" pods receive traffic.


It looks like your **Pods** are healthy and "Ready" (1/1), but the **Endpoints** are missing because you haven't created a **Service** to group them yet.

In Kubernetes, an `Endpoints` object is only created automatically when a `Service` is defined with a selector that matches your Pod labels. Without a Service, there is no "entry point," so there are no endpoints to track.

### The Fix: Create a Service
To see the endpoints in action, you need to apply a Service manifest. Create a file named `readiness-svc.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: readiness-demo
spec:
  selector:
    app: readiness-test
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

### Steps to Verify
1. **Apply the Service:**
   ```bash
   kubectl apply -f readiness-svc.yaml
   ```

2. **Check Endpoints again:**
   ```bash
   kubectl get endpoints readiness-demo
   ```



### Why the "NotFound" error happened:
* **The Deployment** created the Pods.
* **The Readiness Probe** inside the Pods is passing (that's why you see `1/1` in your `get pods` output).
* **The Missing Link:** A Service acts like a load balancer. It looks for Pods with the label `app: readiness-test`. Once it finds them **and** confirms they are "Ready," it adds their internal IP addresses to the `Endpoints` list. 

Since you are following the runbook, notice that step 5 in your original instructions mentioned checking `readiness-svc`. Make sure the name in your `kubectl get` command matches the `metadata.name` in your Service YAML!

Does the `get endpoints` command show the Pod IPs now that you've added the Service?

---

### 5. Configure Startup Probe
**Startup Probes** are vital for legacy or slow-starting apps. They disable liveness and readiness checks until the container has successfully started, preventing premature restarts.

**`startup-probe-pod.yaml`**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-demo
spec:
  containers:
  - name: slow-app
    image: nginx
    startupProbe:
      httpGet:
        path: /
        port: 80
      # Allow up to 30 seconds (10 tries * 3s) for startup
      failureThreshold: 10
      periodSeconds: 3
    livenessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 10
```

**Action:**
```bash
kubectl apply -f startup-probe-pod.yaml
```

---

### Summary Table: Which Probe to Use?

| Probe | Purpose | Action on Failure |
| :--- | :--- | :--- |
| **Liveness** | Checks if the app is dead/stuck. | Restarts the container. |
| **Readiness** | Checks if the app can handle traffic. | Removes Pod from Service endpoints. |
| **Startup** | Protects slow-starting apps during boot. | Disables other probes; restarts if boot times out. |


-----------------------------------------

This runbook provides a comprehensive guide to mastering **Kubernetes DaemonSets**. Unlike a standard Deployment, a DaemonSet ensures that a copy of a specific Pod runs on all (or selected) nodes in your cluster.

---

## 📘 DaemonSet Runbook

### 1. Concept Overview
A **DaemonSet** is used for "background" tasks that need to happen on every node. 
* **Common use cases:** Log collection (Fluentd, Logstash), Monitoring agents (Prometheus Node Exporter), and Network plugins (Calico, Flannel).
* **Behavior:** When a new node is added to the cluster, the DaemonSet automatically adds the Pod to it. When a node is removed, the Pod is garbage collected.



---

### 2. Implementation Scripts

#### File: `log-daemonset.yaml`
This manifest defines a simple log collector using a lightweight BusyBox image.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: default
  labels:
    app: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: log-collector
        image: busybox:1.28
        args:
        - /bin/sh
        - -c
        - "while true; do echo 'Collecting logs...'; sleep 30; done"
```

---

### 3. Step-by-Step Execution Guide

#### Step 1: Deploy the DaemonSet
Apply the manifest to the cluster.
```bash
kubectl apply -f log-daemonset.yaml
```

#### Step 2: Verify Node Coverage
Check that the number of Pods matches the number of nodes in your cluster.
```bash
kubectl get pods -o wide -l app=log-collector
```

#### Step 3: Target Specific Nodes
If you only want the DaemonSet on specific nodes (e.g., worker nodes but not edge nodes), use **Node Selectors**.

1.  **Label your target node:**
    ```bash
    # Replace <node-name> with your actual node name
    kubectl label node <node-name> role=worker
    ```

2.  **Patch the DaemonSet:**
    ```bash
    kubectl patch daemonset log-collector -p \
    '{"spec":{"template":{"spec":{"nodeSelector":{"role":"worker"}}}}}'
    ```

#### Step 4: Handle Taints (Optional)
If some nodes are tainted (like the control plane), the DaemonSet won't run there by default. To force it to run everywhere, add a **toleration** to the spec:
```yaml
# Add this inside spec.template.spec
tolerations:
- effect: NoSchedule
  operator: Exists
```

#### Step 5: Rolling Updates
Update the image version. Kubernetes will replace the Pods one node at a time.
```bash
kubectl set image daemonset/log-collector log-collector=busybox:1.36
```

---

### 4. Troubleshooting Cheat Sheet

| Command | Purpose |
| :--- | :--- |
| `kubectl describe ds log-collector` | Check desired vs. available status and event logs. |
| `kubectl rollout status ds log-collector` | Monitor the progress of an update. |
| `kubectl get nodes --show-labels` | Verify if node labels match your `nodeSelector`. |

---

### 5. Conclusion
You have successfully deployed, labeled, and updated a DaemonSet. You now have a robust mechanism for ensuring system-level agents are consistently present across your infrastructure.

-------------------------------------------

This runbook provides the necessary configuration files (declarative) and commands (imperative) to deploy a StatefulSet in Kubernetes.

---

## 📘 Runbook: Deploying a StatefulSet

### 1. The Declarative Configs (YAML)
Before running commands, create these two files in your working directory.

#### `headless-svc.yaml`
A **Headless Service** is required to control the network domain of the pods. It does not have a ClusterIP.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-svc
  labels:
    app: stateful-demo
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None # This makes it "headless"
  selector:
    app: stateful-demo
```

#### `statefulset.yaml`
This defines the workload. Note the `volumeClaimTemplates`, which ensures each Pod gets its own unique disk.
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-demo
spec:
  serviceName: "stateful-svc"
  replicas: 3
  selector:
    matchLabels:
      app: stateful-demo
  template:
    metadata:
      labels:
        app: stateful-demo
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

---

### 2. Execution Steps (Imperative)

Follow these steps to deploy and verify the setup.

**Step 1: Create the Headless Service**
```bash
kubectl apply -f headless-svc.yaml
```

**Step 2: Deploy the StatefulSet**
```bash
kubectl apply -f statefulset.yaml
```

**Step 3: Verify Stable Network Identities**
Unlike Deployments (which use random suffixes like `web-7fh21`), StatefulSets use ordinal indexes.
```bash
# Watch the pods being created one-by-one (0, then 1, then 2)
kubectl get pods -l app=stateful-demo -w
```

**Step 4: Verify Persistent Volumes**
Check that each Pod has a unique PersistentVolumeClaim (PVC) bound to it.
```bash
kubectl get pvc
```

**Step 5: Test DNS Resolution**
Each Pod gets a predictable DNS name: `<pod-name>.<service-name>.default.svc.cluster.local`.
```bash
kubectl exec -it stateful-demo-0 -- nslookup stateful-demo-0.stateful-svc
```

**Step 6: Scaling the StatefulSet**
When scaling up, Kubernetes adds pods in order (`3`, then `4`). When scaling down, it removes them from highest to lowest.
```bash
kubectl scale statefulset stateful-demo --replicas=5
```

---

### 💡 Summary Table: Deployment vs. StatefulSet

| Feature | Deployment | StatefulSet |
| :--- | :--- | :--- |
| **Pod Names** | Random (e.g., `web-xy12`) | Ordered/Fixed (e.g., `web-0`) |
| **Storage** | Shared or Ephemeral | Individual/Dedicated per Pod |
| **Scaling** | Parallel/Concurrent | Sequential (Ordered) |
| **Use Case** | Stateless Apps (Web Servers) | Databases (MySQL, Mongo, Redis) |

-------------------------------------

This runbook provides the necessary YAML manifests and execution steps to master **Node Affinity** and **Anti-Affinity** in Kubernetes.

---

## 🛠️ Prerequisites & Setup
Before starting, identify your worker nodes to select a target for labeling.
```bash
kubectl get nodes
```
*Choose one worker node name to replace `<node-name>` in the steps below.*

---

## 1. Label a Node
We will simulate a node having high-performance storage.
```bash
# Apply the label
kubectl label node <node-name> disktype=ssd

# Verify the label exists
kubectl get nodes --show-labels | grep disktype
```

---

## 2. Required Affinity (Hard Constraint)
The `requiredDuringSchedulingIgnoredDuringExecution` rule acts as a strict filter. If no node matches the criteria, the Pod will remain in a **Pending** state.

### `required-affinity-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-demo
spec:
  containers:
  - name: nginx-ssd
    image: nginx:latest
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

**Action:**
```bash
kubectl apply -f required-affinity-pod.yaml
```

---

## 3. Preferred Affinity (Soft Constraint)
The `preferredDuringSchedulingIgnoredDuringExecution` rule tells the scheduler to *try* to place the Pod on specific nodes, but to ignore the rule if those nodes are unavailable or over-utilized.

### `preferred-affinity.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: preferred-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: low-latency
  template:
    metadata:
      labels:
        app: low-latency
    spec:
      containers:
      - name: web
        image: nginx
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: region
                operator: In
                values:
                - us-east-1
```

**Action:**
```bash
kubectl apply -f preferred-affinity.yaml
```

---

## 4. Pod Anti-Affinity
This ensures that Pods are spread across different nodes (e.g., for High Availability) by instructing the scheduler not to place a Pod on a node that already runs a Pod with a specific label.

### `anti-affinity.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: anti-affinity-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      containers:
      - name: nginx
        image: nginx
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-server
            topologyKey: "kubernetes.io/hostname"
```

---

## 5. Verification & Testing

### Verify Placement
Check which nodes the pods landed on:
```bash
kubectl get pods -o wide
```

### Test Failure Mode (Required Affinity)
If you remove the label and restart the pod, the scheduler will fail to find a valid home for it.
```bash
# Remove the label (the '-' at the end deletes it)
kubectl label node <node-name> disktype-

# Delete the pod so it attempts to reschedule
kubectl delete pod affinity-demo

# Check status (it should stay Pending)
kubectl get pod affinity-demo
kubectl describe pod affinity-demo | grep Events -A 5
```

---

## 📋 Summary Table of Operators

| Operator | Description |
| :--- | :--- |
| **In** | Label value must be in the provided list. |
| **NotIn** | Label value must NOT be in the provided list. |
| **Exists** | The label key must exist (value doesn't matter). |
| **DoesNotExist** | The label key must NOT exist on the node. |
| **Gt / Lt** | Greater than / Less than (for integer values). |


--------------------------------------------

## Runbook: Managing Workloads with Taints and Tolerations

This runbook guides you through the process of controlling Pod placement by "repelling" certain workloads from specific nodes. Think of **Taints** as a "Keep Out" sign on a node, and **Tolerations** as the special key a Pod needs to enter.

---

### 🛠️ Prerequisites
* A running Kubernetes cluster.
* `kubectl` configured with cluster-admin access.
* Completion of the Node Affinity module (preferred).

---

### 📝 Step-by-Step Implementation

#### 1. Apply a Taint to a Node
We will mark a node as "dedicated" for GPU workloads. This ensures that standard Pods don't accidentally take up resources meant for high-performance tasks.

```bash
# Replace <node-name> with your actual node name (e.g., minikube-m02)
kubectl taint node <node-name> dedicated=gpu:NoSchedule
```
* **Key**: `dedicated`
* **Value**: `gpu`
* **Effect**: `NoSchedule` (New Pods will not be scheduled unless they tolerate this taint).

#### 2. Deploy a Standard Pod (No Toleration)
Let’s see what happens when a normal Pod tries to find a home.

**Script: `no-toleration-pod.yaml`**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: standard-nginx
spec:
  containers:
  - name: nginx
    image: nginx
```

**Action:**
```bash
kubectl apply -f no-toleration-pod.yaml
# Check the status - it will likely land on a different node or stay Pending if no other nodes exist
kubectl get pods -o wide
```

#### 3. Add Toleration to a Pod
Now, let’s create a Pod that is "brave" enough to handle the taint we applied.

**Script: `gpu-toleration-pod.yaml`**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  containers:
  - name: cuda-container
    image: nvidia/cuda:11.0-base
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

**Action:**
```bash
kubectl apply -f gpu-toleration-pod.yaml
```

#### 4. Verify Placement
Check if the `gpu-workload` Pod successfully scheduled onto the tainted node.

```bash
kubectl get pod gpu-workload -o wide
```

#### 5. Evicting Pods with `NoExecute`
The `NoExecute` effect is more aggressive; it evicts Pods already running on the node if they don't have a matching toleration.

```bash
# This will immediately kick off any pod that doesn't tolerate 'maintenance'
kubectl taint node <node-name> maintenance=true:NoExecute
```

#### 6. Cleanup (Remove Taint)
To return the node to a "neutral" state, remove the taint by appending a minus sign (`-`) to the end of the key.

```bash
kubectl taint node <node-name> dedicated-
kubectl taint node <node-name> maintenance-
```

---

### 💡 Key Concepts Summary

| Effect | Behavior |
| :--- | :--- |
| **NoSchedule** | New Pods won't be placed here, but existing ones stay. |
| **PreferNoSchedule** | System tries to avoid the node, but it's not a hard requirement. |
| **NoExecute** | New Pods won't be placed here AND existing Pods are evicted immediately. |

> [!TIP]
> **Taints** are applied to **Nodes**.
> **Tolerations** are applied to **Pods**.

---------------------------------------------


## Runbook: Managing Kubernetes Resource Quotas

This guide walks you through setting up and testing **Resource Quotas** to ensure fair resource distribution in a multi-tenant cluster. Resource Quotas are essential for preventing "noisy neighbor" scenarios where one application consumes all available CPU or Memory.



---

### 1. Prerequisites & Environment Setup
Before starting, ensure your `kubectl` context is pointing to the correct cluster.

**Create the Test Namespace:**
```bash
kubectl create namespace quota-demo
```

---

### 2. Define the Resource Quota
You need a YAML manifest to define the hard limits for the namespace. This script limits total CPU, Memory, and the maximum number of Pods.

**File:** `resource-quota.yaml`
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: quota-demo
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    pods: "10"
```

**Apply the Quota:**
```bash
kubectl apply -f resource-quota.yaml -n quota-demo
```

---

### 3. Monitoring Quota Usage
To see how much of your "budget" you have spent, use the `describe` command. This is your primary tool for debugging quota issues.

```bash
kubectl describe quota compute-resources -n quota-demo
```

> **Note:** Initially, "Used" values will be 0 or near 0 until you deploy resources.

---

### 4. Deploying Within Limits
When a quota is active, Kubernetes **requires** you to specify requests and limits for every container in your Pods. If you don't, the Pod will be rejected.

**File:** `limited-app.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quota-app
  namespace: quota-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

**Apply the deployment:**
```bash
kubectl apply -f limited-app.yaml
```

---

### 5. Testing Quota Enforcement (Exceeding Limits)
Now, let's intentionally break the rules by scaling the deployment beyond the pod limit (set to 10).

**Attempt to scale to 15 replicas:**
```bash
kubectl scale deployment quota-app --replicas=15 -n quota-demo
```

**Verify the Failure:**
Since the quota only allows 10 pods, the ReplicaSet will fail to create the last 5 pods. Check the ReplicaSet events to see the error message:

```bash
kubectl get rs -n quota-demo
kubectl describe rs <replica-set-name> -n quota-demo
```

**Expected Error Message:**
> `Error creating: pods "quota-app-xxx" is forbidden: exceeded quota: compute-resources, requested: pods=1, used: pods=10, limited: pods=10`

---

### 6. Cleanup
To avoid leaving orphan resources in your cluster, delete the namespace:

```bash
kubectl delete namespace quota-demo
```

---

### Summary Table: Quota Scopes
| Resource Type | Description |
| :--- | :--- |
| **requests.cpu** | The sum of CPU requests across all pods cannot exceed this value. |
| **limits.memory** | The sum of memory limits across all pods cannot exceed this value. |
| **pods** | Total count of Pods allowed in the namespace. |
| **services** | Total number of Services allowed (prevents LoadBalancer cost spikes). |

--------------------------------------------

This is the complete, consolidated runbook for **Kubernetes Playbook #21: Init Containers**. It combines the conceptual notes, the successful implementation, the failure simulation, and the recovery steps into one structured document.

---

# 📔 Playbook #21: Init Containers in Kubernetes
**Level:** Beginner  
**Focus:** Lifecycle Management & Pre-runtime Logic

---

## 🏗️ 1. Concept Overview
**Init Containers** are specialized containers that run **before** the app containers in a Pod. They are used to separate setup logic from application code.

### Key Rules
*   **Sequential Execution:** They run one after another. Container B won't start until Container A finishes successfully.
*   **Blocking Nature:** The main application container will **never** start if an init container fails.
*   **Immutability:** Once a Pod is created, you cannot easily change the `initContainers` field; you must delete and recreate the Pod.



---

## 🛠️ 2. Step-by-Step Implementation

### Step 1: Create the Standard Init Pod
This YAML creates a shared volume where the init container writes data for the main Nginx container to use.

**File:** `init-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: main-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  initContainers:
  - name: init-config
    image: busybox
    command: ['sh', '-c', "echo 'Init Complete!' > /work-dir/index.html"]
    volumeMounts:
    - name: shared-data
      mountPath: /work-dir
```

**Run:**
```bash
kubectl apply -f init-pod.yaml
```

### Steps 2-4: Monitor & Verify
```bash
# 2. Watch the phase change from Init:0/1 to Running
kubectl get pod init-demo -w

# 3. Check what the init container did
kubectl logs init-demo -c init-config

# 4. Verify the main container can see the data
kubectl exec -it init-demo -- cat /usr/share/nginx/html/index.html
```

---

## ⚠️ 3. Handling Failures (The "Fail & Fix" Cycle)

### Step 5: Simulate Init Failure
We create a Pod where the init container explicitly exits with an error code.

**File:** `failing-init-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: failing-init
spec:
  containers:
  - name: main-app
    image: nginx
  initContainers:
  - name: init-fail
    image: busybox
    command: ['sh', '-c', 'echo "Checking DB connection... Failed!"; exit 1']
```

**Run & Observe:**
```bash
kubectl apply -f failing-init-pod.yaml
kubectl get pod failing-init
# STATUS will show: Init:CrashLoopBackOff

# Describe to see the "Back-off" events
kubectl describe pod failing-init
```

### Step 6: Fix and Redeploy
Since we cannot edit the existing Pod's command, we delete and apply a corrected version.

**File:** `fixed-init-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fixed-init
spec:
  containers:
  - name: main-app
    image: nginx
  initContainers:
  - name: init-success
    image: busybox
    command: ['sh', '-c', 'echo "Checking DB connection... Success!"; exit 0']
```

**Run:**
```bash
kubectl delete pod failing-init
kubectl apply -f fixed-init-pod.yaml
```

---

## 📝 4. Final Summary Notes

| Feature | Details |
| :--- | :--- |
| **Termination** | Init containers must exit/terminate. They cannot be long-running like the app. |
| **Status Codes** | `Exit 0` is required to move to the next stage. |
| **Use Case 1** | **Wait-for:** Use a loop to wait for a Database Service to be "Ready". |
| **Use Case 2** | **Security:** Download secrets or certificates into a volume without putting the tools in the main image. |
| **Logging** | Always use `kubectl logs <pod> -c <init-container-name>` to debug. |

---
✅ **Conclusion:** Init Containers allow for a "clean" application image by offloading environment preparation to a separate, temporary container.

-------------------------------------------------------


Here is the complete, consolidated runbook and set of technical notes for **Module #22: Kubernetes Events and Debugging**. 

This guide is designed to be used as a "cheat sheet" for both learning and practical on-the-job troubleshooting.

---

## 📘 Technical Notes: Kubernetes Events
Events are not just logs; they are **API objects** within Kubernetes that record state changes and errors.

### Why Events Matter
*   **Audit Trail:** They explain *why* a Pod is stuck in `Pending` or `ImagePullBackOff`.
*   **Lifecycle Tracking:** They show when a node was added, when a container started, and when the scheduler made a decision.
*   **Ephemeral Nature:** By default, events are deleted after **1 hour** to prevent overloading the etcd database.

### The Lifecycle of a Pod Failure


1.  **Pending:** The Scheduler cannot find a node (check `FailedScheduling`).
2.  **ContainerCreating:** Issues with pulling images or mounting volumes (check `FailedMount` or `ErrImagePull`).
3.  **Running (CrashLoop):** The app starts but crashes (check `BackOff`).
4.  **Terminated:** The app was killed by the system (check `OOMKilled`).

---

## 🛠 Runbook: Troubleshooting Kubernetes
Follow these steps to diagnose and resolve common cluster issues.

### Phase 1: Generating the "Lab" Environment
To practice debugging, create these two manifests on your local machine.

**File:** `bad-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
spec:
  containers:
  - name: error-container
    image: doesnotexist:latest
```

**File:** `oom-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
spec:
  containers:
  - name: stress-container
    image: polinux/stress
    resources:
      limits:
        memory: "50Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

---

### Phase 2: Execution & Investigation

#### 1. Deployment
```bash
kubectl apply -f bad-pod.yaml
kubectl apply -f oom-pod.yaml
```

#### 2. Deep Dive with Describe
When a specific pod fails, this is your first line of defense.
```bash
kubectl describe pod bad-pod
```
*   **Action:** Scroll to the bottom. Look for `Pulling`, then `Failed`. The message will tell you exactly which registry it couldn't find.

#### 3. Analyzing Memory Issues (OOM)
If a pod disappears or restarts without a clear error, check the termination state.
```bash
kubectl get pod oom-pod -o yaml | grep -A 5 "lastState"
```
*   **Evidence:** Look for `reason: OOMKilled`. This confirms your resource limits are too low for the application's needs.

#### 4. The "Global View" Debugging
If multiple things are breaking at once, use field selectors to cut through the noise.
```bash
# Get only Warning events across all namespaces
kubectl get events -A --field-selector type=Warning

# Sort by time to see the most recent catastrophic events
kubectl get events --sort-by='.lastTimestamp'
```

---

### Phase 3: Resolution Summary
| Symptom | Event Reason | Resolution |
| :--- | :--- | :--- |
| **ImagePullBackOff** | `Failed` / `InspectFailed` | Fix image name or add `imagePullSecrets`. |
| **Pod stuck in Pending** | `FailedScheduling` | Check node capacity; check `Taints` and `Tolerations`. |
| **CrashLoopBackOff** | `BackOff` | Check application logs: `kubectl logs <pod>`. |
| **OOMKilled** | `Evicted` or `OOMKilled` | Increase `resources.limits.memory` in the YAML. |

---

### ✅ Conclusion
Debugging in Kubernetes is the art of **matching symptoms to events**. If the pod isn't doing what you expect, the answer is almost always hidden in `kubectl describe` or the event stream.

> **Note:** For real-world production environments, consider using a tool like **Kubewatch** or **EventRouter** to forward these events to Slack or an ELK stack before they expire after 60 minutes.


----------------------------------------------

## Runbook: Deploying Applications with Helm

**Project ID:** #23

**Level:** Beginner

**Focus:** Package Management for Kubernetes

---

### 📘 Overview & Concepts

Helm is essentially the **"App Store"** for Kubernetes. It allows you to manage complex applications through **Charts**, which are packages of pre-configured Kubernetes resources. Instead of managing dozens of individual YAML files for a single app, you use Helm to deploy everything as a single unit.

**Key Terminology:**

* **Chart:** A bundle of information necessary to create an instance of a Kubernetes application.
* **Repo:** A place where charts can be collected and shared (like GitHub for Helm).
* **Release:** An instance of a chart running in a Kubernetes cluster.

---

### 🛠 Prerequisites

Before starting, ensure your environment meets these requirements:

* **Kubernetes Cluster:** A running cluster (Minikube, Kind, or Cloud-based).
* **kubectl CLI:** Installed and configured to communicate with your cluster.
* **Permissions:** Sufficient RBAC permissions to create deployments and services.

---

### 🚀 Execution Steps

#### 1. Install Helm

The following command fetches the official installation script and executes it locally.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

```

*Note: Always verify the version after installation using `helm version`.*

#### 2. Configure Repositories

Helm doesn't come with charts built-in. You must point it to a repository. Bitnami is the industry standard for well-maintained, secure charts.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

```

#### 3. Locate Your Application

Search the repository to ensure the package exists and to see the available versions.

```bash
helm search repo bitnami/nginx

```

#### 4. Deploy (The "Install" Phase)

Deploy the Nginx web server. The name `my-nginx` is your **Release Name**, which allows you to track this specific instance.

```bash
helm install my-nginx bitnami/nginx

```

#### 5. Verification

Check that Helm recognizes the release and that Kubernetes has spun up the physical Pods.

```bash
helm list
kubectl get pods

```

#### 6. Cleanup

To avoid resource waste, uninstall the release. This removes all associated services, deployments, and pods created by the chart.

```bash
helm uninstall my-nginx

```

---

### 📝 Post-Deployment Notes

* **Customization:** While this runbook uses default settings, Helm’s power lies in the `values.yaml` file. You can override settings (like port numbers or replica counts) using the `--set` flag or by providing your own YAML file.
* **Idempotency:** Helm allows you to run `helm upgrade --install`, which will either install the chart if it’s missing or update it if it already exists.
* **Rollbacks:** If a deployment goes south, Helm keeps a history. You can revert to a previous working state using `helm rollback <release-name> <revision-number>`.

> **Pro-Tip:** Use `helm status my-nginx` to see the specific instructions (like IP addresses or passwords) generated by the chart author after installation.
  
-----------------------------------------------------

## 🛡️ Kubernetes Network Policy Lab: Zero-Trust Microsegmentation

This lab covers the transition from a default "open" network to a secured "Zero-Trust" environment. These notes are structured for production readiness, covering basic, intermediate, and troubleshooting use cases.

---

### 📋 Lab Environment Context

* **Cluster Type:** AKS with Azure NPM (`network-policy: azure`)
* **Namespace:** `default` (or your chosen lab namespace)
* **Logic:** Ingress rules are **additive**. If a pod is targeted by a policy, it enters "Isolation Mode."

---

### 🛠️ Phase 1: The Setup

Deploy three pods to test different communication paths: a **Frontend**, a **Backend**, and an **Unauthorized** pod.

```bash
# 1. Deploy pods with specific labels
kubectl run frontend --image=busybox --labels="tier=frontend" -- sleep 3600
kubectl run backend --image=nginx --labels="tier=backend"
kubectl run unauthorized --image=busybox --labels="tier=other" -- sleep 3600

# 2. Expose Backend via ClusterIP Service
kubectl expose pod backend --port=80 --name=backend

```

---

### 🚀 Phase 2: Use Cases

#### Use Case 1: The "Security Baseline" (Default Deny)

**Objective:** Block all lateral movement by default. This ensures that no pod can talk to another unless explicitly permitted.

**`01-default-deny.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {} # Targets ALL pods in the namespace
  policyTypes:
  - Ingress

```

* **Verification:** `kubectl exec frontend -- wget -qO- --timeout=2 http://backend`
* **Result:** `timed out` (Correct).

#### Use Case 2: "Service-to-Service" (Allow Specific Tier)

**Objective:** Allow the Frontend to access the Backend, but keep the Unauthorized pod blocked.

**`02-allow-frontend.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      tier: backend # The "Target"
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend # The "Source"
    ports:
    - protocol: TCP
      port: 80

```

* **Verification (Frontend):** `kubectl exec frontend -- wget -qO- --timeout=2 http://backend` -> **Success (200 OK)**.
* **Verification (Unauthorized):** `kubectl exec unauthorized -- wget -qO- --timeout=2 http://backend` -> **Timed Out**.

#### Use Case 3: "Namespace Isolation"

**Objective:** Allow traffic only from pods within a specific namespace (e.g., allow `monitoring` namespace to scrape `app` namespace).

**`03-namespace-allow.yaml`**

```yaml
spec:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring

```

#### Use Case 4: "Egress Control" (External API Access)

**Objective:** Prevent pods from calling the public internet, except for a specific IP (e.g., an external database).

**`04-egress-limit.yaml`**

```yaml
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 20.30.40.50/32
    ports:
    - protocol: TCP
      port: 443

```

---

### 📝 Expert Troubleshooting & Best Practices

| Issue | Root Cause | Verification Command |
| --- | --- | --- |
| **"Bad Address"** | DNS Failure (UDP 53) | `kubectl exec ... -- nslookup google.com` |
| **Policy Ignored** | CNI doesn't support NetPol | `az aks show ... --query networkProfile.networkPolicy` |
| **Traffic Still Flows** | Label Mismatch | `kubectl get pods --show-labels` |
| **Everything Breaks** | Egress Deny-All applied | Check if `policyTypes` includes `Egress` without rules. |

---

### 💡 Key Design Principles for your Notes

1. **Implicit Deny:** Once a pod is selected by a policy, it is "isolated." Unselected pods remain "non-isolated" (open) unless a `Default Deny` exists.
2. **DNS is Egress:** If you implement a global **Egress Deny**, you **MUST** add a rule to allow traffic to `kube-system` on Port 53 (UDP), otherwise pods cannot resolve service names.
3. **Namespace Labels:** Modern K8s automatically labels namespaces with `kubernetes.io/metadata.name`. Use this for easy `namespaceSelector` rules.
4. **Blast Radius:** By isolating tiers, a compromised Frontend pod cannot be used to port-scan your internal DB or other internal services.

---

### ✅ Conclusion

By moving from the first cluster (`none`) to the second cluster (`azure`), you observed that Network Policy is a **software-defined firewall** managed by the CNI/NPM agent. Without that agent, the YAML is just metadata; with it, it is a powerful security enforcement tool.


-----------------------------------------------------------------

## Runbook #25: Local Access via `kubectl port-forward`

This runbook provides a streamlined guide for using port-forwarding to securely access internal Kubernetes resources from your local machine.

---

### 📋 Overview & Purpose

`kubectl port-forward` allows you to bridge a local port on your workstation to a port on a specific Pod or Service within your cluster. It is primarily used for **debugging**, **testing**, and **database administration** without the need to configure complex Ingress rules or expensive LoadBalancers.

### 🛠 Prerequisites

* A running Kubernetes cluster.
* `kubectl` CLI configured and authenticated.
* At least one running Pod or Service to target.

---

### 🚀 Step-by-Step Execution

#### 1. Deploy the Demo Application

Create a simple Nginx deployment and expose it via an internal `ClusterIP` service (this service is not accessible from the public internet).

```bash
kubectl create deployment pf-demo --image=nginx
kubectl expose deployment pf-demo --port=80

```

#### 2. Forward to a Pod

Map your local port **8080** to the deployment's port **80**. Kubernetes will automatically select a pod managed by the deployment.

```bash
kubectl port-forward deployment/pf-demo 8080:80

```

> **Note:** The process will stay active in your terminal. If you close the terminal or press `Ctrl+C`, the connection will drop.

#### 3. Verify Connectivity

While the forward is active, open a new terminal tab or a web browser:

* **Browser:** Navigate to `http://localhost:8080`
* **Terminal:**
```bash
curl http://localhost:8080

```



#### 4. Forward to a Service

Alternatively, you can forward traffic to the Service level. This is often preferred as it abstracts away specific pod names.

```bash
kubectl port-forward svc/pf-demo 9090:80

```

#### 5. Manage as a Background Process

To keep your terminal free while maintaining the connection, run the command in the background.

```bash
kubectl port-forward svc/pf-demo 9090:80 &

```

#### 6. Terminate the Connection

To stop a background port-forwarding session:

```bash
# To kill the most recent background job
kill %1

```

---

### 📝 Strategic Notes

| Feature | `port-forward` | `LoadBalancer` / `Ingress` |
| --- | --- | --- |
| **Primary Use** | Temporary debugging / Dev | Production traffic |
| **Access** | Local machine only | Public or Private Network |
| **Security** | High (uses kubeconfig auth) | Depends on firewall/IAM |
| **Cost** | Free | Usually incurs cloud provider fees |

#### ⚠️ Critical Reminders

* **Security:** `port-forward` is only as secure as your `kubeconfig` file. Never leave sensitive ports forwarded on shared machines.
* **Stability:** This is a tunnel, not a permanent networking solution. It may timeout or disconnect during network hiccups.
* **Targeting:** If you forward to a **Deployment**, `kubectl` will pick one pod. If that pod dies, the port-forward session usually terminates. Forwarding to a **Service** provides slightly more resilience.

```

```
-------------------------------------------------

## Kubernetes Context Management: Beginner’s Guide & Runbook

Managing multiple Kubernetes clusters doesn't have to feel like juggling chainsaws. By mastering the `kubeconfig` file, you can move between environments (Dev, Staging, Prod) with confidence and speed.

---

### 🧠 Key Concepts: The "Kubeconfig" Hierarchy

A `kubeconfig` file is essentially a YAML map that tells `kubectl` how to find and authenticate with your clusters. It is built on three main pillars:

1. **Clusters:** The API server URL and certificate authority (Where am I going?).
2. **Users:** Your credentials—tokens, passwords, or client certificates (Who am I?).
3. **Contexts:** A named shortcut that links a **User** to a **Cluster** and optionally sets a default **Namespace** (How am I connecting?).

---

### 🛠 Practical Runbook

Follow these steps to audit and manage your environments.

#### 1. Audit Your Current Setup

Before changing anything, see what clusters your machine currently "knows" about.

* **View Full Config:** `kubectl config view`
* **List All Contexts:** `kubectl config get-contexts`
> *Note: The asterisk (*) indicates your currently active context.*



#### 2. Switching Environments

Moving from one cluster to another is the most common task.

* **Switch Context:**
```bash
kubectl config use-context <context-name>

```



#### 3. Namespace Optimization

Stop typing `-n production` every time. You can "pin" a context to a specific namespace.

* **Set Default Namespace:**

```bash
    kubectl config set-context --current --namespace=<your-namespace>
    ```

#### 4. Merging Multiple Configs
If a cloud provider gives you a new `.yaml` file, don't just replace your old config. Merge them into one master file.
*   **The Merge Command:**
    
```bash
    KUBECONFIG=~/.kube/config:~/new-cluster.yaml kubectl config view --flatten > ~/.kube/config_new
    mv ~/.kube/config_new ~/.kube/config
    ```

---

### 💡 Pro-Tips for Efficiency

#### 🚀 Use Helper Tools
Manual `kubectl` commands can be wordy. Most pros use these lightweight wrappers:
*   **kubectx:** Switch contexts instantly.
*   **kubens:** Switch namespaces instantly.
*   **Installation:** `brew install kubectx`

#### 🛡 Context Hygiene (Best Practices)
*   **Distinct Names:** Rename your contexts to something readable (e.g., `prod-us-east` instead of `gke_project-123_us-east1-a_cluster-1`).
*   **The "Golden Rule":** Always run `kubectl config current-context` before running a `delete` or `apply` command to ensure you aren't accidentally hitting production.
*   **Shell Prompt:** Use a tool like **Starship** or **Oh My Zsh** to display your current K8s context directly in your terminal prompt.

---

> **Summary:** Your `kubeconfig` is the "address book" for your infrastructure. Keep it organized, use contexts to define your boundaries, and leverage tools like `kubectx` to stay fast and safe.

```


------------------------------------------------

## Runbook #27: Deploying Applications with YAML Manifests

**Level:** Beginner

**Focus:** Multi-resource Management & Declarative Workflows

This runbook covers the transition from imperative commands to **declarative manifests**, allowing you to manage complex application stacks as a single unit of truth.

---

### 📋 Prerequisites

* **Kubernetes Cluster:** Access to a running cluster (Minikube, Kind, or Cloud).
* **kubectl CLI:** Installed and configured.
* **YAML Basics:** Understanding of key-value pairs, lists, and indentation.

---

### 🛠 Step-by-Step Instructions

#### 1. Construct the Multi-Resource YAML

In Kubernetes, you can stack multiple objects in a single file using the `---` (three dashes) document separator. This ensures the API processes them as distinct entities.

**Example: `full-app.yaml**`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  api-url: "http://api.production.com"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

```

#### 2. Perform a Dry Run

Before committing changes to the cluster, validate the syntax and permissions.

* **Command:** `kubectl apply -f full-app.yaml --dry-run=client`
* **Why:** It catches "fat-finger" typos or schema errors without actually creating resources.

#### 3. Apply the Manifest

Apply the entire stack in one go.

* **Command:** `kubectl apply -f full-app.yaml`
* **Note:** `kubectl` is smart enough to handle the resources in order (usually creating ConfigMaps before the Pods that need them).

#### 4. Verify the Deployment

Check that all components are running and correctly labeled.

* **Command:** `kubectl get all -l app=web-app`
* **Tip:** Using labels (`-l`) is the most efficient way to filter a multi-resource stack.

#### 5. Bulk Management (Directory Apply)

If your app has many files, you don’t need to list them individually.

* **Command:** `kubectl apply -f ./manifests/`
* **Effect:** This applies every `.yaml` and `.yml` file found within the specified folder.

#### 6. Clean Up

To remove every resource defined in your file:

* **Command:** `kubectl delete -f full-app.yaml`

---

### 📝 Key Technical Notes

| Concept | Description |
| --- | --- |
| **Document Separator (`---`)** | Required to define multiple YAML documents in a single physical file. |
| **Declarative vs Imperative** | **Declarative** (Apply) tells K8s what the final state should look like; **Imperative** (Run/Create) tells K8s what specific action to take right now. |
| **Idempotency** | Running `kubectl apply` multiple times with the same file will result in "unchanged" status if the cluster matches the file. |

> **Pro-Tip:** Always keep your YAML manifests in a Git repository. This practice, known as **GitOps**, ensures that your infrastructure is versioned and easily reproducible if the cluster fails.

---

### ✅ Conclusion

By mastering multi-resource manifests, you have moved away from manual "one-off" commands. You can now define entire environments—from databases to frontends—in a single, shareable format that serves as the "Source of Truth" for your infrastructure.


----------------------


---

## 📘 Runbook: Implementing Startup Probes for Slow-Starting Apps

### 1. Assessment Phase

Before adding the probe, determine the "Grace Window."

* **Formula:** $T_{startup} \times 1.25 = \text{Total Buffer}$
* **Configuration:**
* `periodSeconds`: How often to check (default 10s).
* `failureThreshold`: $(\text{Total Buffer} / \text{periodSeconds})$.



### 2. Implementation Template (YAML)

Apply this pattern to bypass liveness/readiness during the boot sequence:

```yaml
spec:
  containers:
  - name: heavy-app
    startupProbe:
      httpGet:
        path: /healthz/startup
        port: 8080
      failureThreshold: 30 # Allows 5 mins if period is 10s
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz/live
        port: 8080
      periodSeconds: 20
    readinessProbe:
      httpGet:
        path: /healthz/ready
        port: 8080
      periodSeconds: 5

```

### 3. Verification Commands

* **Monitor the Hand-off:**
`kubectl get po -w`
*(Look for the container to become 1/1 READY only after the startup window)*
* **Audit Probe Timing:**
`kubectl describe po [pod-name] | grep -i "probes"`

---

## 📝 Technical Notes: The "SRE Cheat Sheet"

### 🛡️ The Startup vs. Liveness Conflict

* **The Problem:** Without a Startup probe, the Liveness probe starts ticking at $T=0$. If your Java/Spring Boot app takes 60s to start, and Liveness `failureThreshold` is 3, the pod restarts at 30s.
* **The Fix:** Startup probes **block** Liveness and Readiness probes. They are the only probe active during the boot phase. Once it succeeds **once**, it disappears until the container restarts.

### ⚡ Troubleshooting "CrashLoopBackOff"

If a pod is restarting despite having a Startup probe:

1. **Check Events:** `kubectl get events --sort-by=.lastTimestamp`
* *Message:* "Startup probe failed" -> Increase `failureThreshold`.
* *Message:* "Liveness probe failed" -> The app started but crashed *after* initialization.


2. **Resource Throttling:** Ensure the container has enough CPU `requests`. If CPU is throttled during startup, the probe might timeout even if the logic is correct.

### 🚀 Best Practices for Senior DevOps

* **Use the same endpoint?** It’s common to use the same `/healthz` for all three, but use different thresholds.
* **Exec vs. HTTP:** For legacy apps without an HTTP endpoint, use a Marker File:
`exec: command: ["cat", "/tmp/initialized"]`.
* **GKE/AKS Specifics:** Ensure your Cloud Load Balancer (Ingress) timeout is longer than your Readiness probe window, or you'll see 502 errors during rolling updates.

---

-----------------------------

## Kubernetes Service Accounts: Exercise #29

Based on the instructions provided, here are the YAML manifests required to complete the steps. These define the custom Service Account, the Pod that utilizes it, and a secured Pod with token mounting disabled.

---

### 1. Service Account Manifest

While you can create this via CLI, having the YAML is best practice for version control.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default

```

### 2. Pod with Custom Service Account (`sa-pod.yaml`)

This Pod references the `app-sa` created in step 2.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-pod
  namespace: default
spec:
  serviceAccountName: app-sa
  containers:
  - name: nginx
    image: nginx:latest

```

### 3. Pod with Disabled Token Auto-mount (`no-token-pod.yaml`)

For enhanced security (Step 5), this configuration prevents the API token from being automatically injected into the container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-token-pod
  namespace: default
spec:
  automountServiceAccountToken: false
  containers:
  - name: nginx
    image: nginx:latest

```

---

### 📝 Implementation Notes

* **Token Path:** When you run the `exec` command in Step 4, the token is located at `/var/run/secrets/kubernetes.io/serviceaccount/token`. This is a projected volume managed by the Kubelet.
* **Security Context:** By default, every namespace has a `default` Service Account. If you don't specify `serviceAccountName`, Kubernetes assigns this default one. In production, always create a dedicated SA with **Least Privilege** as shown in Step 6.
* **RBAC Binding:** The RoleBinding in Step 6 connects your `app-sa` to the `view` ClusterRole, allowing the Pod to "read" resources within the `default` namespace without granting administrative power.

---

### Summary Table: Commands Reference

| Action | Command |
| --- | --- |
| **List SAs** | `kubectl get sa` |
| **Create SA** | `kubectl create sa app-sa` |
| **Check Token** | `kubectl describe sa app-sa` |
| **Apply Pod** | `kubectl apply -f sa-pod.yaml` |
| **Verify RBAC** | `kubectl auth can-i list pods --as=system:serviceaccount:default:app-sa` |



--------------------------

## #30 — Complete Runbook + YAML Lab
### Viewing and Streaming Pod Logs | Beginner Level

---

## 🎯 Objectives
- Retrieve logs from single and multi-container pods
- Stream live logs
- Follow logs from a crashed previous container instance

---

## Prerequisites
- `kubectl` CLI configured and connected to a cluster
- A running Pod or Deployment to target
- Basic understanding of Kubernetes Pod lifecycle

---

## Core Concepts

| Term | Description |
|---|---|
| **stdout/stderr** | Kubernetes captures container stdout and stderr as logs |
| **Container Runtime** | Logs stored by container runtime (containerd) on the node |
| **Log Rotation** | Node-level log rotation applies — very old logs may not be available |
| **Previous Instance** | When a container restarts, old logs accessible via `--previous` |
| **Label Selector** | Kubernetes labels allow targeting multiple pods in one command |

---

---

# STEP 1 — Single Container Pod

## YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
    env: dev
spec:
  containers:
    - name: app-container
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

## Apply
```bash
kubectl apply -f 01-single-pod.yaml
kubectl get pod my-pod
```

## View Logs
```bash
# All logs
kubectl logs my-pod

# With namespace
kubectl logs my-pod -n my-namespace

# With timestamps
kubectl logs my-pod --timestamps=true
```

## Cleanup
```bash
kubectl delete -f 01-single-pod.yaml
```

---

---

# STEP 2 — Stream Logs (Follow Mode)

## YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
    env: dev
spec:
  containers:
    - name: app-container
      image: busybox:1.35
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            echo "[app] $(date) - request processed"
            sleep 3
          done
      resources:
        requests:
          memory: "32Mi"
          cpu: "100m"
        limits:
          memory: "64Mi"
          cpu: "200m"
```

## Apply
```bash
kubectl apply -f 02-stream-pod.yaml
kubectl get pod my-pod
```

## Stream Logs
```bash
# Follow live output
kubectl logs -f my-pod

# Follow with namespace
kubectl logs -f my-pod -n my-namespace

# Follow with timestamps
kubectl logs -f my-pod --timestamps=true
```

> Use `Ctrl+C` to stop streaming.

## Cleanup
```bash
kubectl delete -f 02-stream-pod.yaml
```

---

---

# STEP 3 — Tail Last N Lines

## YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
    env: dev
spec:
  containers:
    - name: app-container
      image: busybox:1.35
      command: ["/bin/sh", "-c"]
      args:
        - |
          i=1
          while true; do
            echo "[app] line $i - $(date)"
            i=$((i+1))
            sleep 1
          done
      resources:
        requests:
          memory: "32Mi"
          cpu: "100m"
        limits:
          memory: "64Mi"
          cpu: "200m"
```

## Apply
```bash
kubectl apply -f 03-tail-pod.yaml
kubectl get pod my-pod
```

## Tail Logs
```bash
# Last 50 lines
kubectl logs my-pod --tail=50

# Last 100 lines
kubectl logs my-pod --tail=100

# Stream from last 50 lines
kubectl logs -f my-pod --tail=50

# Last 30 minutes of logs
kubectl logs my-pod --since=30m

# Last 1 hour with timestamps
kubectl logs my-pod --since=1h --timestamps=true
```

## Cleanup
```bash
kubectl delete -f 03-tail-pod.yaml
```

---

---

# STEP 4 — Previous Container Logs (CrashLoopBackOff)

## YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
  labels:
    app: my-app
    env: dev
spec:
  restartPolicy: Always
  containers:
    - name: crash-container
      image: busybox:1.35
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "App starting..."
          sleep 5
          echo "ERROR: Simulated crash! Exiting with code 1"
          exit 1
      resources:
        requests:
          memory: "32Mi"
          cpu: "100m"
        limits:
          memory: "64Mi"
          cpu: "200m"
```

## Apply
```bash
kubectl apply -f 04-crash-pod.yaml

# Watch it crash and restart
kubectl get pod crash-pod -w
```

## Previous Logs
```bash
# Current instance logs
kubectl logs crash-pod

# Previously crashed instance logs
kubectl logs crash-pod --previous

# Last 50 lines of crashed instance
kubectl logs crash-pod --previous --tail=50

# With timestamps
kubectl logs crash-pod --previous --timestamps=true
```

## Cleanup
```bash
kubectl delete -f 04-crash-pod.yaml
```

---

---

# STEP 5 — Multi-Container Pod (App + Sidecar)

## YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  labels:
    app: my-app
    env: dev
spec:
  containers:
    - name: app-container
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"

    - name: sidecar-container
      image: busybox:1.35
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            echo "[sidecar] $(date) - heartbeat ok"
            sleep 10
          done
      resources:
        requests:
          memory: "32Mi"
          cpu: "100m"
        limits:
          memory: "64Mi"
          cpu: "200m"
```

## Apply
```bash
kubectl apply -f 05-multi-container-pod.yaml
kubectl get pod multi-container-pod
```

## List Containers First
```bash
kubectl get pod multi-container-pod -o jsonpath='{.spec.containers[*].name}'
```

## Logs Per Container
```bash
# App container
kubectl logs multi-container-pod -c app-container

# Sidecar container
kubectl logs multi-container-pod -c sidecar-container

# Stream sidecar
kubectl logs -f multi-container-pod -c sidecar-container

# Tail app container
kubectl logs multi-container-pod -c app-container --tail=50
```

## Cleanup
```bash
kubectl delete -f 05-multi-container-pod.yaml
```

---

---

# STEP 6 — Pod with Init Container

## YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
  labels:
    app: my-app
    env: dev
spec:
  initContainers:
    - name: init-container
      image: busybox:1.35
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "Init: checking dependencies..."
          sleep 5
          echo "Init: all checks passed. Starting main container."
      resources:
        requests:
          memory: "32Mi"
          cpu: "100m"
        limits:
          memory: "64Mi"
          cpu: "200m"

  containers:
    - name: app-container
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

## Apply
```bash
kubectl apply -f 06-init-pod.yaml

# Watch pod transition through Init phase
kubectl get pod init-pod -w
```

## Logs
```bash
# Init container logs (during or after init phase)
kubectl logs init-pod -c init-container

# App container logs (after init completes)
kubectl logs init-pod -c app-container
```

## Cleanup
```bash
kubectl delete -f 06-init-pod.yaml
```

---

---

# STEP 7 — All Pods in Deployment (Label Selector)

## YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
  labels:
    app: my-app
    env: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        env: dev
    spec:
      containers:
        - name: app-container
          image: busybox:1.35
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                echo "[$(hostname)] $(date) - request ok"
                sleep 5
              done
          resources:
            requests:
              memory: "32Mi"
              cpu: "100m"
            limits:
              memory: "64Mi"
              cpu: "200m"
```

## Apply
```bash
kubectl apply -f 07-deployment.yaml

# Verify all 3 pods running
kubectl get pods -l app=my-app
```

## Logs — All Pods
```bash
# All pods at once
kubectl logs -l app=my-app --all-containers=true

# With prefix — shows which pod each line came from
kubectl logs -l app=my-app --all-containers=true --prefix=true

# Full streaming with timestamps
kubectl logs -f -l app=my-app --all-containers=true --prefix=true --timestamps=true

# Tail last 20 lines from all pods
kubectl logs -l app=my-app --all-containers=true --tail=20 --prefix=true
```

## Cleanup
```bash
kubectl delete -f 07-deployment.yaml
```

---

---

# STEP 8 — All-in-One Lab File

## YAML (apply everything at once)

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
    env: dev
spec:
  containers:
    - name: app-container
      image: busybox:1.35
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            echo "[my-pod] $(date) - running"
            sleep 5
          done
      resources:
        requests:
          memory: "32Mi"
          cpu: "100m"
        limits:
          memory: "64Mi"
          cpu: "200m"
---
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  labels:
    app: my-app
    env: dev
spec:
  containers:
    - name: app-container
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
    - name: sidecar-container
      image: busybox:1.35
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            echo "[sidecar] $(date) - heartbeat ok"
            sleep 10
          done
      resources:
        requests:
          memory: "32Mi"
          cpu: "100m"
        limits:
          memory: "64Mi"
          cpu: "200m"
---
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
  labels:
    app: my-app
    env: dev
spec:
  restartPolicy: Always
  containers:
    - name: crash-container
      image: busybox:1.35
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "App starting..."
          sleep 5
          echo "ERROR: Simulated crash!"
          exit 1
      resources:
        requests:
          memory: "32Mi"
          cpu: "100m"
        limits:
          memory: "64Mi"
          cpu: "200m"
---
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
  labels:
    app: my-app
    env: dev
spec:
  initContainers:
    - name: init-container
      image: busybox:1.35
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "Init: checking dependencies..."
          sleep 5
          echo "Init: all checks passed."
      resources:
        requests:
          memory: "32Mi"
          cpu: "100m"
        limits:
          memory: "64Mi"
          cpu: "200m"
  containers:
    - name: app-container
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
  labels:
    app: my-app
    env: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        env: dev
    spec:
      containers:
        - name: app-container
          image: busybox:1.35
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                echo "[$(hostname)] $(date) - request ok"
                sleep 5
              done
          resources:
            requests:
              memory: "32Mi"
              cpu: "100m"
            limits:
              memory: "64Mi"
              cpu: "200m"
```

## Apply All
```bash
kubectl apply -f 08-all-in-one.yaml

# Check all pods
kubectl get pods -l env=dev
```

## Full Lab Test Sequence
```bash
# 1. Single pod logs
kubectl logs my-pod
kubectl logs -f my-pod

# 2. Crash pod - wait for restart then check previous
kubectl get pod crash-pod -w
kubectl logs crash-pod --previous

# 3. Multi-container pod
kubectl get pod multi-container-pod -o jsonpath='{.spec.containers[*].name}'
kubectl logs multi-container-pod -c app-container
kubectl logs multi-container-pod -c sidecar-container

# 4. Init pod
kubectl logs init-pod -c init-container
kubectl logs init-pod -c app-container

# 5. Deployment - all pods
kubectl get pods -l app=my-app
kubectl logs -l app=my-app --all-containers=true --prefix=true
kubectl logs -f -l app=my-app --all-containers=true --prefix=true --timestamps=true
```

## Cleanup All
```bash
kubectl delete -f 08-all-in-one.yaml
```

---

---

# Flags Cheat Sheet

| Flag | Description | Example |
|---|---|---|
| `-f` | Stream / follow live output | `kubectl logs -f my-pod` |
| `--tail=N` | Show last N lines only | `--tail=50` |
| `--previous` | Logs from previous crashed instance | `kubectl logs my-pod --previous` |
| `-c` | Target specific container | `-c sidecar-container` |
| `-l` | Label selector — multiple pods | `-l app=my-app` |
| `--all-containers=true` | All containers across matched pods | combined with `-l` |
| `--prefix=true` | Prefix each line with pod/container name | combined with `-l` |
| `--since=Xm` | Logs since duration | `--since=30m` |
| `--timestamps=true` | Include timestamp on each line | `--timestamps=true` |
| `--limit-bytes=N` | Cap output size in bytes | `--limit-bytes=1048576` |
| `-n` | Target namespace | `-n production` |

---

# Save Logs to File

```bash
# Single pod
kubectl logs my-pod > my-pod-logs.txt

# Previous crashed instance
kubectl logs my-pod --previous > crashed-logs.txt

# All pods in deployment
kubectl logs -l app=my-app --all-containers=true --prefix=true > deployment-logs.txt

# With timestamps
kubectl logs my-pod --timestamps=true > my-pod-timestamped.txt
```

---

# Filter Logs

```bash
# Linux / Mac
kubectl logs my-pod | grep -i "error\|exception\|fatal"

# Windows PowerShell
kubectl logs my-pod | Select-String -Pattern "error|exception"

# Grep and save
kubectl logs my-pod | grep -i "error" > errors.txt
```

---

# Beyond kubectl — Production Tools

| Tool | Use Case |
|---|---|
| **stern** | Multi-pod log tailing with color-coded output and regex filtering |
| **kubetail** | Bash script to tail multiple pods by name prefix |
| **Loki + Grafana** | Production log aggregation, querying, dashboarding |
| **EFK Stack** | Elasticsearch + Fluentd + Kibana — enterprise log pipeline |
| **Azure Monitor** | Managed log aggregation for AKS with Log Analytics |
| **CloudWatch** | Managed log aggregation for EKS on AWS |

```bash
# stern — tail all pods matching name pattern
stern my-app --namespace production --since 15m

# stern — with regex filter
stern my-app --include="error|exception" --since 1h
```

---

# Debugging Decision Flow

```
Pod not running?
  └── kubectl describe pod my-pod          ← check Events section

Pod running but misbehaving?
  └── kubectl logs my-pod                  ← current logs

Pod crashed and restarted?
  └── kubectl logs my-pod --previous       ← previous instance

Multi-container pod?
  └── kubectl get pod my-pod \
        -o jsonpath='{.spec.containers[*].name}'
  └── kubectl logs my-pod -c <container>

Multi-pod deployment issue?
  └── kubectl logs -l app=my-app \
        --all-containers=true --prefix=true
```

---

## Conclusion

> Log access is the **first step** in any debugging session. Master `kubectl logs` basics first, then layer on `stern` or a full log aggregation stack for production-grade observability.

**Debugging priority order:**
1. Pod not running → `kubectl describe pod my-pod` → check **Events**
2. Pod running but misbehaving → `kubectl logs my-pod`
3. Pod crashed and restarted → `kubectl logs my-pod --previous`
4. Multi-container pod → `kubectl logs my-pod -c <container-name>`
5. Multi-pod deployment issue → `kubectl logs -l app=my-app --all-containers=true --prefix=true`