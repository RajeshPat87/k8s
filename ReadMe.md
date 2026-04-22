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