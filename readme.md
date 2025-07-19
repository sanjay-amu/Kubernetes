**Kubernetes Architecture & Design: Interview-Focused Guide**

---

### 🔧 Core Kubernetes Architecture

#### 🧠 What is the Kubernetes Control Plane?

The control plane is the brain of the Kubernetes cluster. It includes the components responsible for global decisions about the cluster (e.g., scheduling), as well as detecting and responding to cluster events.

**Control Plane Components:**

* **API Server:** The front door to the cluster. Accepts REST requests and serves as the communication hub for all components.
* **etcd:** A distributed key-value store that stores all cluster data (desired and observed state). It acts as the single source of truth.
* **Scheduler:** Assigns new pods to appropriate nodes based on resource availability, affinity rules, taints/tolerations, etc.
* **Controller Manager:** Hosts various controllers like the ReplicaSet Controller, Deployment Controller, Node Controller, etc. These continuously reconcile actual state with desired state.

**Worker Node Components:**

* **kubelet:** Runs on each node. Ensures containers are running and healthy.
* **kube-proxy:** Maintains network rules and handles service traffic routing.
* **Container Runtime:** e.g., containerd, CRI-O. Responsible for running containers.

**Example Use Case:**

* Deploying a microservice via a Deployment spec results in:

  * API server storing the new spec.
  * Deployment controller creating or updating a ReplicaSet.
  * Scheduler assigning pods to nodes.
  * kubelet launching containers and reporting back status.

---

### 🔁 Desired vs Actual State Reconciliation

Kubernetes uses a declarative approach. You define the desired state in YAML or through `kubectl apply`, and Kubernetes ensures that the actual state of the cluster matches it.

**Reconciliation Process:**

1. The user applies a Deployment YAML.
2. The API server stores the desired state in etcd.
3. The controller (e.g., Deployment controller) detects the change.
4. If there's a difference from the actual state, it creates a new ReplicaSet.
5. New pods are scheduled and launched; old ones are terminated gradually.

**Diagram:**

```
User ➝ kubectl apply ➝ API Server ➝ etcd
                               ⬇
                Controller watches for changes
                               ⬇
          Create/update ReplicaSet ➝ Create Pods
                               ⬇
                Scheduler ➝ kubelet ➝ Container Runtime
```

**Interview Tip:** Reconciliation is what enables Kubernetes' "self-healing" nature.

---

### ⚡ High Availability and Failover

**etcd HA:**

* Runs in odd-number quorum (e.g., 3, 5, or 7 members).
* Cluster remains available as long as quorum (majority) is maintained.

**API Server HA:**

* Multiple instances behind a load balancer.
* Clients (kubectl, controllers) connect via load balancer.

**Controller/Scheduler HA:**

* Run in active-passive leader election mode.
* Failover is automatic if a leader node crashes.

**If etcd is unavailable:**

* API Server can't read/write state.
* Controllers can't make decisions.
* The cluster becomes read-only.
* Running pods remain active, but no changes (scheduling, scaling) occur.

**Use Case:**

* During etcd outage in a PCI-compliant fintech app, payments continue but new instances of payment processing pods can't be scheduled.

---

### 🏛️ Controller Manager & Built-In Controllers

The `kube-controller-manager` runs multiple independent control loops:

**Key Controllers:**

* **ReplicaSet Controller:** Maintains desired pod replica count.
* **Deployment Controller:** Handles rollout and rollback of application versions.
* **StatefulSet Controller:** Manages pods that require stable identity and storage.
* **Node Controller:** Monitors node health and handles pod eviction.
* **ServiceAccount Controller:** Creates default service accounts.
* **Job/CronJob Controllers:** Manage one-time or scheduled job execution.

**Interview Scenario:**

> “You’ve deployed an app with 5 replicas. 2 pods crash. How does Kubernetes react?”

* ReplicaSet controller detects deviation.
* Automatically spins up 2 new pods to match desired count.

---

### 🏗️ Pod Lifecycle (from `kubectl apply` to Running)

1. Apply YAML with `kubectl apply`.
2. API Server stores spec in etcd.
3. Deployment Controller compares and creates ReplicaSet.
4. ReplicaSet launches Pods.
5. Scheduler assigns them to appropriate nodes.
6. Kubelet starts containers.
7. Readiness/liveness probes are checked.
8. Once new pods are ready, old pods are gracefully terminated.

**Interview Tip:** Mention rolling updates and readiness gates. During updates, both old and new pods might coexist briefly.

---

### 🔐 Design & Production Considerations

#### ✅ Secrets & Configs

* **Secrets**: Base64-encoded by default (not encrypted).
* Use tools like **SealedSecrets**, **Vault**, or cloud KMS integrations.
* Avoid secrets in plain YAML; use references.

#### ✅ Multi-Tenancy & Isolation

* Use **Namespaces** to segment teams or workloads.
* Combine with **RBAC** for access control.
* Add **NetworkPolicies** for traffic isolation.
* Use **ResourceQuotas** and **LimitRanges** to prevent resource abuse.

#### ✅ Deployment Strategies

* **Recreate**: Stop all old pods, then start new ones.
* **RollingUpdate**: Default; gradually replaces pods.
* **Blue-Green**: Two environments; switch traffic.
* **Canary**: Partial rollout; observe, then proceed.

**Code Snippet – Canary via labels:**

```yaml
selector:
  matchLabels:
    version: v1
---
selector:
  matchLabels:
    version: v2
```

#### ✅ Scaling AI/Fintech Workloads

* Use **HPA (Horizontal Pod Autoscaler)** for CPU/memory based scaling.
* Use **KEDA** for event-based scaling (queue length, Kafka, etc.).
* **Taints/Tolerations** for scheduling pods on GPU-specific nodes.
* **Affinity rules** for co-locating services.

#### ✅ Disaster Recovery & Backup

* Periodic **etcd snapshots** stored off-cluster.
* Use GitOps tools (ArgoCD/Flux) for stateless restoration.
* Store YAML manifests in Git for reproducibility.

**Example Interview Scenario:**

> "You’re running a real-time fraud detection ML model on Kubernetes. How do you ensure availability and scale?"

* Use StatefulSet for model server if persistent.
* Enable GPU nodes with taints + tolerations.
* Auto-scale with HPA/KEDA.
* Use Liveness/Readiness probes.
* Expose via Ingress + TLS.

---

### 🌐 Exposing Services & Ingress

**Service Types:**

* **ClusterIP**: Default, internal-only
* **NodePort**: Exposes on static port across nodes
* **LoadBalancer**: Cloud-based external LB
* **Headless**: No cluster IP; used with StatefulSet

**Ingress Controller:**

* Routes HTTP(S) traffic based on paths/hosts
* TLS termination supported
* NGINX, Traefik, HAProxy are common controllers

**Code Example – Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

---

### 🎓 Interview Takeaways

* Know Kubernetes control plane components and how they work together.
* Be able to explain the lifecycle of a pod clearly.
* Understand reconciliation, HA, controller roles.
* Talk through deployment strategies and failure scenarios.
* Explain how you'd scale and secure fintech/AI applications.
* Bring real use cases into your answers.

---

Absolutely — let’s walk through how to implement each Kubernetes deployment strategy via YAML, specifying exactly:
	•	Which YAML file to edit
	•	Which section to modify
	•	And provide a concrete example for each

⸻

🔁 1. Recreate Strategy

✅ YAML File:

Edit your deployment.yaml file.

🔧 Section to Modify:

spec:
  strategy:
    type: Recreate

📄 Full Example:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: my-image:latest


⸻

🔄 2. RollingUpdate Strategy (default)

✅ YAML File:

Edit your deployment.yaml

🔧 Section to Modify:

spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%

📄 Full Example:

spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

This allows 1 extra Pod during rollout while keeping all existing ones running.

⸻

🔵 3. Blue-Green Strategy (manual via separate Deployments)

✅ Files Needed:
	•	deployment-blue.yaml
	•	deployment-green.yaml
	•	service.yaml

🔧 Key Part:

Create two deployments (with different labels) and switch the selector in the Service.

📄 Example deployment-blue.yaml:

metadata:
  name: my-app-blue
spec:
  template:
    metadata:
      labels:
        app: my-app
        version: blue

📄 Example deployment-green.yaml:

metadata:
  name: my-app-green
spec:
  template:
    metadata:
      labels:
        app: my-app
        version: green

📄 Example service.yaml:

spec:
  selector:
    app: my-app
    version: green  # 👈 switch from blue to green here

You manually flip the version label in the service selector to switch traffic.

⸻

🐤 4. Canary Strategy (manual or via tools like Argo Rollouts)

✅ Files Needed:
	•	deployment-v1.yaml
	•	deployment-v2.yaml
	•	Optional: Ingress with header-based routing or Argo Rollout CR

🔧 Strategy:
	•	Deploy v1 with label version: stable
	•	Deploy v2 with label version: canary
	•	Split traffic in Ingress (manually or via service mesh)

📄 Example Deployments:

# deployment-v1.yaml
metadata:
  name: my-app-stable
spec:
  template:
    metadata:
      labels:
        app: my-app
        version: stable

# deployment-v2.yaml
metadata:
  name: my-app-canary
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
        version: canary

📄 Example NGINX Ingress Traffic Split:

nginx.ingress.kubernetes.io/canary: "true"
nginx.ingress.kubernetes.io/canary-weight: "10"

Or use Argo Rollouts with a CRD:

apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: { duration: 2m }


⸻

🧠 Summary Table:

Strategy	File	Section to Modify	Tools Needed
Recreate	deployment.yaml	spec.strategy.type: Recreate	None
RollingUpdate	deployment.yaml	spec.strategy.type: RollingUpdate	None
Blue-Green	2 deployments + 1 service	Service spec.selector.version	Manual switch
Canary	2 deployments + Ingress OR Argo Rollout	Ingress annotations or Argo spec	Ingress / ArgoCD


⸻
