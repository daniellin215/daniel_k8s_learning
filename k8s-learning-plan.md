# Kubernetes Learning Plan for Fullstack Engineers

> **Target Audience:** Fullstack SWE with Node.js/React experience transitioning to Infrastructure  
> **Duration:** 6-8 weeks @ 5-10 hours/week  
> **Local Environment:** minikube  
> **Project Stack:** React (Frontend) + Node.js/Express (Backend) + PostgreSQL (Database)  
> **Advanced Topics:** Helm, GitHub Actions CI/CD, Prometheus/Grafana, Service Mesh (Istio)

---

## Table of Contents

1. [Week 1: Kubernetes Fundamentals & Local Setup](#week-1-kubernetes-fundamentals--local-setup)
2. [Week 2: Configuration, Secrets, and Namespaces](#week-2-configuration-secrets-and-namespaces)
3. [Week 3: Building the 3-Tier Application](#week-3-building-the-3-tier-application)
4. [Week 4: Networking Deep Dive - Ingress & DNS](#week-4-networking-deep-dive---ingress--dns)
5. [Week 5: Helm - Package Management for Kubernetes](#week-5-helm---package-management-for-kubernetes)
6. [Week 6: CI/CD with GitHub Actions](#week-6-cicd-with-github-actions)
7. [Week 7: Monitoring with Prometheus & Grafana](#week-7-monitoring-with-prometheus--grafana)
8. [Week 8: Service Mesh & Production Best Practices](#week-8-service-mesh--production-best-practices)
9. [Additional Resources & Certification Path](#additional-resources--certification-path)

---

## Week 1: Kubernetes Fundamentals & Local Setup

### Learning Objectives

- Understand why Kubernetes exists and what problems it solves
- Set up a local development environment with minikube
- Master the core building blocks: Pods, Deployments, Services
- Learn to use kubectl effectively

### Day 1-2: Understanding Kubernetes Architecture

#### Core Concepts to Learn

**What is Kubernetes?**

Kubernetes (K8s) is a container orchestration platform that automates deployment, scaling, and management of containerized applications. Coming from Node.js, think of it as "pm2 on steroids" - but for containers across multiple machines.

**The Control Plane Components:**

```
┌─────────────────────────────────────────────────────────────┐
│                      CONTROL PLANE                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ API Server  │  │  Scheduler  │  │ Controller Manager  │  │
│  │ (kube-api)  │  │             │  │                     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                        etcd                             ││
│  │              (distributed key-value store)              ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      WORKER NODES                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   kubelet   │  │ kube-proxy  │  │  Container Runtime  │  │
│  │             │  │             │  │  (containerd/CRI-O) │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

| Component | Description | Node.js Analogy |
|-----------|-------------|-----------------|
| **API Server** | The front door to K8s. All kubectl commands go here | Express handling HTTP requests |
| **etcd** | Distributed key-value store holding all cluster state | PostgreSQL but for K8s itself |
| **Scheduler** | Decides which node runs your Pod | Load balancer deciding where to send traffic |
| **Controller Manager** | Runs control loops ensuring desired state matches actual state | Reconciliation loop |
| **kubelet** | Agent on each node that ensures containers are running | pm2 per machine |
| **kube-proxy** | Handles networking rules for Services on each node | iptables/networking layer |

#### Hands-On: Trace a `kubectl` Request Flow

Use this mini-lab to trace a real request:
`kubectl` -> API Server -> `etcd` -> Scheduler -> kubelet

**1. Confirm which API Server your `kubectl` talks to**

```bash
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}{"\n"}'
```

**What to notice:** this is the control-plane endpoint every `kubectl` command calls.

**2. Run one command with verbose client logs**

```bash
kubectl --v=8 get namespaces
```

**What to notice:** request URL, HTTP verb, and response code in debug output.  
This is direct evidence of `kubectl` calling the API Server.

**3. Create a Deployment to trigger scheduling**

```bash
kubectl create deployment flow-demo --image=nginx:1.25 --replicas=1
kubectl get pods -l app=flow-demo -w
```

**What to notice:** Pod moves `Pending` -> `Running` after scheduling and kubelet startup.

**4. Inspect Pod events (best single view of the flow)**

```bash
POD=$(kubectl get pods -l app=flow-demo -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD"
```

**What to notice in Events:**
- `default-scheduler`: "Successfully assigned ..." (Scheduler picked a node)
- `kubelet`: Pulling/Created/Started container (node agent executed it)

**5. Verify API Server health checks include etcd**

```bash
kubectl get --raw='/readyz?verbose' | grep etcd
```

**What to notice:** `etcd` readiness appears in API Server checks.  
You do not call `etcd` directly with `kubectl`; the API Server persists desired state into `etcd`.

**6. Optional cleanup**

```bash
kubectl delete deployment flow-demo
```

#### Reading Materials

| Resource | Type | Time | Why This Matters | Focus While Reading | Link |
|----------|------|------|------------------|---------------------|------|
| Kubernetes Components | Official Docs | 30 min | Builds the core control-plane vs worker-node mental model. | Trace one request flow: `kubectl` -> API Server -> `etcd` -> Scheduler -> kubelet. | https://kubernetes.io/docs/concepts/overview/components/ |
| What is Kubernetes? | Official Docs | 20 min | Clarifies desired state and reconciliation, the foundation for every later week. | Compare what K8s solves vs Docker Compose/pm2 workflows you already know. | https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/ |
| Kubernetes: The Documentary | Video | 55 min | Gives historical context on why Kubernetes and CNCF standards emerged. | Note the operational tradeoffs that come with orchestration at scale. | https://www.youtube.com/watch?v=BE77h7dmoQU |
| The Illustrated Children's Guide to Kubernetes | Interactive | 15 min | Fast visual model before reading deeper docs. | Translate each metaphor back to real terms: Pod, Node, Control Plane, Cluster. | https://www.cncf.io/phippy/the-childrens-illustrated-guide-to-kubernetes/ |
| Kubernetes The Hard Way | Deep Dive | 4+ hrs | Deep architecture reference that becomes valuable in Weeks 4-8. | Skim PKI, bootstrap, and control-plane setup now; save full lab for later. | https://github.com/kelseyhightower/kubernetes-the-hard-way |

---

### Day 3-4: Local Environment Setup with minikube

#### Step-by-Step Setup

**1. Install Prerequisites**

```bash
# macOS (using Homebrew)
brew install minikube kubectl

# Linux (Debian/Ubuntu)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify installations
minikube version
kubectl version --client
```

**2. Start Your First Cluster**

```bash
# Start minikube with recommended settings
minikube start --cpus=4 --memory=8192 --driver=docker

# Verify cluster is running
minikube status

# Check cluster info
kubectl cluster-info
```

**3. Explore the Cluster**

```bash
# List all nodes (minikube creates 1 node by default)
kubectl get nodes

# View all namespaces
kubectl get namespaces

# View all pods across all namespaces
kubectl get pods -A
```

**Understanding kubectl Command Structure:**

```
kubectl [command] [TYPE] [NAME] [flags]
        │         │      │      │
        │         │      │      └── Options like -o yaml, -n namespace
        │         │      └── Resource name (optional)
        │         └── Resource type (pods, deployments, services)
        └── Action (get, describe, create, apply, delete)
```

**4. Enable Useful Addons**

```bash
# Enable the dashboard
minikube addons enable dashboard
minikube addons enable metrics-server

# Access dashboard (opens in browser)
minikube dashboard
```

#### Reading Materials

| Resource | Type | Time | Why This Matters | Focus While Reading | Link |
|----------|------|------|------------------|---------------------|------|
| minikube Start Guide | Official Docs | 30 min | Canonical setup path for a stable local cluster baseline. | Pick driver, CPU/memory sizing, and verification steps for your OS. | https://minikube.sigs.k8s.io/docs/start/ |
| kubectl Cheat Sheet | Reference | Bookmark | Daily command reference you will reuse throughout the plan. | Practice `get`, `describe`, `logs`, `exec`, `apply`, `delete`, and `-n`/`-o` flags. | https://kubernetes.io/docs/reference/kubectl/cheatsheet/ |
| kubectl Overview | Official Docs | 20 min | Explains command structure and resource-centric API interactions. | Understand imperative vs declarative usage and how resource types map to commands. | https://kubernetes.io/docs/reference/kubectl/ |
| kubectl Book | Free eBook | 2 hrs | Structured deep dive for long-term CLI fluency. | Read intro + context/namespace navigation first; bookmark troubleshooting chapters. | https://kubectl.docs.kubernetes.io/ |

---

### Day 5-7: Pods, Deployments, and Services

#### Understanding Pods

A **Pod** is the smallest deployable unit in K8s. It wraps one or more containers that share:

- Network namespace (same IP, can talk via localhost)
- Storage volumes
- Lifecycle (created/destroyed together)

**Your First Pod (Imperative - for learning only):**

```bash
# Run an nginx pod
kubectl run my-nginx --image=nginx:latest

# See the pod
kubectl get pods

# Get detailed info
kubectl describe pod my-nginx

# See logs (like docker logs)
kubectl logs my-nginx

# Execute into the pod (like docker exec)
kubectl exec -it my-nginx -- /bin/bash

# Delete the pod
kubectl delete pod my-nginx
```

**Pod YAML (Declarative - the right way):**

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx
  labels:
    app: nginx
    environment: dev
spec:
  containers:
    - name: nginx
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

```bash
# Apply the manifest
kubectl apply -f pod.yaml

# View with labels
kubectl get pods --show-labels
```

#### Understanding Deployments

**Why not just use Pods?** Pods are ephemeral - if they die, they're gone. **Deployments** manage Pods declaratively:

- Maintains desired replica count
- Handles rolling updates
- Supports rollbacks
- Self-healing (recreates failed pods)

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3                    # Run 3 identical pods
  selector:
    matchLabels:
      app: nginx                 # Manage pods with this label
  template:                      # Pod template
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
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

```bash
# Apply deployment
kubectl apply -f deployment.yaml

# Watch pods being created
kubectl get pods -w

# See deployment status
kubectl get deployments

# Scale up/down
kubectl scale deployment nginx-deployment --replicas=5

# Update image (triggers rolling update)
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# View rollout status
kubectl rollout status deployment/nginx-deployment

# Rollback if something goes wrong
kubectl rollout undo deployment/nginx-deployment

# View rollout history
kubectl rollout history deployment/nginx-deployment
```

#### Understanding Services

**Problem:** Pod IPs are ephemeral. When pods restart, they get new IPs. How do other pods find them?

**Solution:** Services provide stable networking:

| Service Type | Description | Use Case |
|--------------|-------------|----------|
| **ClusterIP** (default) | Internal cluster IP only | Pod-to-Pod communication |
| **NodePort** | Exposes on each node's IP at static port (30000-32767) | Development/testing |
| **LoadBalancer** | Creates external load balancer | Production (cloud providers) |
| **ExternalName** | DNS CNAME record | Connect to external services |

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP              # or NodePort, LoadBalancer
  selector:
    app: nginx                 # Route to pods with this label
  ports:
    - protocol: TCP
      port: 80                 # Service port
      targetPort: 80           # Container port
```

**Service Discovery:**

```bash
# Within the cluster, services are accessible via DNS:
# <service-name>.<namespace>.svc.cluster.local
# Example: nginx-service.default.svc.cluster.local

# Or simply by service name within same namespace:
# nginx-service
```

**Testing Service Connectivity:**

```bash
# Apply service
kubectl apply -f service.yaml

# Get service details
kubectl get svc nginx-service

# Test from within cluster
kubectl run curl-test --image=curlimages/curl -it --rm -- \
  curl http://nginx-service

# For NodePort, access via minikube
minikube service nginx-service --url
```

#### Reading Materials

| Resource | Type | Time | Why This Matters | Focus While Reading | Link |
|----------|------|------|------------------|---------------------|------|
| Pods Overview | Official Docs | 30 min | Teaches Pod lifecycle, container grouping, and execution model. | Learn phases, restart behavior, and when sidecars make sense. | https://kubernetes.io/docs/concepts/workloads/pods/ |
| Deployments | Official Docs | 45 min | Core controller for resilient app rollout and self-healing. | Practice `rollout status/history/undo` and update strategy concepts. | https://kubernetes.io/docs/concepts/workloads/controllers/deployment/ |
| Services | Official Docs | 45 min | Provides stable connectivity despite ephemeral Pod IPs. | Compare ClusterIP/NodePort/LoadBalancer and selector-based routing. | https://kubernetes.io/docs/concepts/services-networking/service/ |
| Service Discovery | Official Docs | 20 min | Explains internal DNS and naming conventions across namespaces. | Memorize service FQDN pattern and namespace DNS search behavior. | https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/ |
| Nana K8s Crash Course | Video | 1 hr | Visual end-to-end recap with practical CLI examples. | Watch Pod/Deployment/Service segments and replay commands in minikube. | https://www.youtube.com/watch?v=s_o8dwzRlu4 |
| K8s Basics Interactive Tutorial | Interactive | 1 hr | Hands-on reinforcement in a guided sandbox. | Complete modules through Services/replication before moving to Week 2. | https://kubernetes.io/docs/tutorials/kubernetes-basics/ |

---

### Week 1 Checkpoint

By end of Week 1, you should be able to:

- [ ] Explain K8s architecture and core components
- [ ] Have minikube running locally
- [ ] Create Pods, Deployments, and Services using YAML
- [ ] Use kubectl confidently (get, describe, logs, exec, apply, delete)
- [ ] Understand the difference between imperative and declarative approaches

---

## Week 2: Configuration, Secrets, and Namespaces

### Learning Objectives

- Externalize application configuration using ConfigMaps
- Securely manage sensitive data with Secrets
- Organize resources using Namespaces
- Implement resource quotas and limits

### Day 1-2: ConfigMaps

**Why ConfigMaps?**

As a Node.js developer, you're used to `.env` files or `config.js`. In K8s, we externalize configuration so the same container image works across environments.

#### Creating ConfigMaps

**Method 1: From Literal Values**

```bash
kubectl create configmap app-config \
  --from-literal=NODE_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-literal=API_URL=http://api-service:3000
```

**Method 2: From File**

```bash
# Create a config file
cat > app.env << EOF
DATABASE_HOST=postgres-service
DATABASE_PORT=5432
DATABASE_NAME=myapp
EOF

kubectl create configmap app-config --from-env-file=app.env
```

**Method 3: YAML Manifest (Recommended)**

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Simple key-value pairs
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  
  # Multi-line config file
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      location / {
        proxy_pass http://backend:3000;
      }
    }
```

#### Using ConfigMaps in Pods

**As Environment Variables:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-app
spec:
  containers:
    - name: app
      image: node:18
      env:
        # Single value from ConfigMap
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: NODE_ENV
      
      # All values from ConfigMap as env vars
      envFrom:
        - configMapRef:
            name: app-config
```

**As Volume Mount (for config files):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-config
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: config-volume
      configMap:
        name: app-config
        items:
          - key: nginx.conf
            path: default.conf
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| ConfigMaps | Official Docs | 30 min | https://kubernetes.io/docs/concepts/configuration/configmap/ |
| Configure Pods with ConfigMaps | Tutorial | 45 min | https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/ |

---

### Day 3-4: Secrets

**Secrets vs ConfigMaps:**

- ConfigMaps: Non-sensitive configuration
- Secrets: Sensitive data (passwords, API keys, certificates)

> **Important:** K8s Secrets are base64 encoded, NOT encrypted by default. For production, use external secret managers (Vault, AWS Secrets Manager) or enable encryption at rest.

#### Creating Secrets

**Method 1: Imperative (for quick testing)**

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret123
```

**Method 2: YAML Manifest**

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  # Values must be base64 encoded
  username: YWRtaW4=           # echo -n 'admin' | base64
  password: c3VwZXJzZWNyZXQxMjM=  # echo -n 'supersecret123' | base64
---
# Or use stringData for plain text (K8s will encode it)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-v2
type: Opaque
stringData:
  username: admin
  password: supersecret123
```

#### Using Secrets in Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secrets
spec:
  containers:
    - name: app
      image: node:18
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      
      # Or mount as files
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: db-credentials
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Secrets | Official Docs | 30 min | https://kubernetes.io/docs/concepts/configuration/secret/ |
| Managing Secrets | Official Docs | 30 min | https://kubernetes.io/docs/tasks/configmap-secret/ |
| Encrypting Secrets at Rest | Official Docs | 20 min | https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/ |
| External Secrets Operator | GitHub | Reference | https://github.com/external-secrets/external-secrets |
| Sealed Secrets | GitHub | Reference | https://github.com/bitnami-labs/sealed-secrets |

---

### Day 5-6: Namespaces and Resource Management

#### Namespaces

**What are Namespaces?**

Namespaces provide isolation within a cluster - think of them as virtual clusters. Use them for:

- Environment separation (dev, staging, prod)
- Team isolation
- Resource quota enforcement

```bash
# Default namespaces
kubectl get namespaces

# NAME              STATUS   AGE
# default           Active   1d    # Where resources go if not specified
# kube-system       Active   1d    # K8s system components
# kube-public       Active   1d    # Publicly accessible data
# kube-node-lease   Active   1d    # Node heartbeat data
```

**Creating and Using Namespaces:**

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: prod
```

```bash
# Apply namespace
kubectl apply -f namespace.yaml

# Create resources in a namespace
kubectl apply -f deployment.yaml -n development

# Set default namespace for kubectl context
kubectl config set-context --current --namespace=development

# List resources in specific namespace
kubectl get pods -n production

# List resources across all namespaces
kubectl get pods -A
```

#### Resource Quotas and Limits

**Resource Requests vs Limits:**

- **Requests:** Guaranteed resources (used for scheduling)
- **Limits:** Maximum resources (enforced at runtime)

```yaml
# Pod with resource constraints
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: node:18
      resources:
        requests:
          memory: "256Mi"      # Guaranteed 256MB RAM
          cpu: "250m"          # Guaranteed 0.25 CPU cores
        limits:
          memory: "512Mi"      # Max 512MB RAM (OOMKilled if exceeded)
          cpu: "500m"          # Max 0.5 CPU cores (throttled if exceeded)
```

**LimitRange (Default limits for a namespace):**

```yaml
# limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
    - default:           # Default limits if not specified
        memory: "512Mi"
        cpu: "500m"
      defaultRequest:    # Default requests if not specified
        memory: "256Mi"
        cpu: "250m"
      type: Container
```

**ResourceQuota (Namespace-wide limits):**

```yaml
# resourcequota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
    services: "10"
    secrets: "20"
    configmaps: "20"
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Namespaces | Official Docs | 20 min | https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/ |
| Resource Quotas | Official Docs | 30 min | https://kubernetes.io/docs/concepts/policy/resource-quotas/ |
| Limit Ranges | Official Docs | 20 min | https://kubernetes.io/docs/concepts/policy/limit-range/ |
| Managing Resources | Official Docs | 30 min | https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/ |

---

### Day 7: Health Checks (Probes)

**Why Health Checks?**

K8s needs to know if your app is healthy to:

- Route traffic only to healthy pods
- Restart unhealthy pods
- Wait for pods to be ready before sending traffic

**Three Types of Probes:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: node-api
  template:
    metadata:
      labels:
        app: node-api
    spec:
      containers:
        - name: api
          image: my-node-api:1.0
          ports:
            - containerPort: 3000
          
          # Liveness Probe: Is the container alive?
          # If fails, container is restarted
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15    # Wait before first check
            periodSeconds: 10          # Check every 10s
            timeoutSeconds: 5          # Timeout after 5s
            failureThreshold: 3        # Restart after 3 failures
          
          # Readiness Probe: Is the container ready for traffic?
          # If fails, removed from Service endpoints
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          
          # Startup Probe: Is the container started?
          # Disables liveness/readiness until it succeeds
          # Good for slow-starting containers
          startupProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 30
            periodSeconds: 10
```

**Node.js Health Check Endpoints:**

```javascript
// Express.js example
app.get('/health', (req, res) => {
  // Basic liveness - app is running
  res.status(200).json({ status: 'ok' });
});

app.get('/ready', async (req, res) => {
  // Readiness - check dependencies
  try {
    await db.query('SELECT 1');
    await redis.ping();
    res.status(200).json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready', error: error.message });
  }
});
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Container Probes | Official Docs | 30 min | https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes |
| Configure Probes | Tutorial | 30 min | https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/ |

---

### Week 2 Checkpoint

By end of Week 2, you should be able to:

- [ ] Create and use ConfigMaps for application configuration
- [ ] Manage sensitive data with Secrets
- [ ] Organize resources with Namespaces
- [ ] Set resource requests and limits
- [ ] Implement health checks for your applications

---

## Week 3: Building the 3-Tier Application

### Learning Objectives

- Containerize a React frontend and Node.js backend
- Deploy PostgreSQL with persistent storage
- Connect all three tiers with proper service discovery
- Understand init containers and startup dependencies

### Day 1-2: Prepare Your Application

#### Project Structure

```
k8s-fullstack-app/
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── package.json
│   └── src/
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       └── index.js
└── k8s/
    ├── namespace.yaml
    ├── database/
    │   ├── secret.yaml
    │   ├── configmap.yaml
    │   ├── pvc.yaml
    │   ├── deployment.yaml
    │   └── service.yaml
    ├── backend/
    │   ├── configmap.yaml
    │   ├── deployment.yaml
    │   └── service.yaml
    └── frontend/
        ├── configmap.yaml
        ├── deployment.yaml
        └── service.yaml
```

#### Backend Dockerfile (Node.js/Express)

```dockerfile
# backend/Dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app

# Security: Run as non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001
USER nodejs

COPY --from=builder /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .

EXPOSE 3000
CMD ["node", "src/index.js"]
```

#### Sample Backend Code (Express)

```javascript
// backend/src/index.js
const express = require('express');
const { Pool } = require('pg');

const app = express();
app.use(express.json());

// Database connection using environment variables
const pool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
});

// Health check endpoint (liveness)
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Readiness check endpoint
app.get('/ready', async (req, res) => {
  try {
    await pool.query('SELECT 1');
    res.status(200).json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready', error: error.message });
  }
});

// Sample API endpoint
app.get('/api/items', async (req, res) => {
  try {
    const result = await pool.query('SELECT * FROM items');
    res.json(result.rows);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

#### Frontend Dockerfile (React with Nginx)

```dockerfile
# frontend/Dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.25-alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### Frontend Nginx Config

```nginx
# frontend/nginx.conf
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # Handle React Router (SPA)
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API requests to backend
    location /api {
        proxy_pass http://backend-service:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_cache_bypass $http_upgrade;
    }
}
```

#### Build and Push Images

```bash
# Using minikube's Docker daemon (no need for registry)
eval $(minikube docker-env)

# Build images
docker build -t myapp-backend:1.0 ./backend
docker build -t myapp-frontend:1.0 ./frontend

# Verify images exist
docker images | grep myapp
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Docker Multi-stage Builds | Docker Docs | 20 min | https://docs.docker.com/build/building/multi-stage/ |
| Node.js Docker Best Practices | Guide | 30 min | https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md |
| React Production Build | React Docs | 15 min | https://create-react-app.dev/docs/production-build/ |
| 12-Factor App | Methodology | 45 min | https://12factor.net/ |

---

### Day 3-4: Deploy PostgreSQL with Persistent Storage

#### Understanding Persistent Volumes

**The Storage Hierarchy:**

```
┌─────────────────────────────────────────────────────────┐
│ PersistentVolumeClaim (PVC)                            │
│ "I need 10Gi of SSD storage"                           │
│          │                                              │
│          ▼                                              │
│ PersistentVolume (PV)                                  │
│ "Here's 10Gi backed by storage"                        │
│          │                                              │
│          ▼                                              │
│ StorageClass                                           │
│ "I provision storage on demand"                        │
└─────────────────────────────────────────────────────────┘
```

- **StorageClass:** Defines how storage is provisioned (minikube provides 'standard')
- **PersistentVolume (PV):** Actual storage resource
- **PersistentVolumeClaim (PVC):** Request for storage by a Pod

#### PostgreSQL Manifests

**1. Namespace**

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    app: myapp
```

**2. Database Secret**

```yaml
# k8s/database/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
  namespace: myapp
type: Opaque
stringData:
  POSTGRES_USER: appuser
  POSTGRES_PASSWORD: "your-secure-password-here"
  POSTGRES_DB: myappdb
```

**3. PersistentVolumeClaim**

```yaml
# k8s/database/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: myapp
spec:
  accessModes:
    - ReadWriteOnce          # Single node can mount read-write
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard  # minikube's default StorageClass
```

**4. PostgreSQL Deployment**

```yaml
# k8s/database/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: myapp
spec:
  replicas: 1                  # Databases typically run 1 replica
  selector:
    matchLabels:
      app: postgres
  strategy:
    type: Recreate             # Don't run multiple instances during update
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgres-credentials
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
              subPath: postgres      # Avoid permission issues
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - appuser
                - -d
                - myappdb
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - appuser
                - -d
                - myappdb
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
```

**5. PostgreSQL Service**

```yaml
# k8s/database/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: myapp
spec:
  type: ClusterIP              # Internal only
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

**Deploy the Database:**

```bash
# Apply all database manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/database/

# Verify PVC is bound
kubectl get pvc -n myapp

# Verify postgres is running
kubectl get pods -n myapp -w

# Test connection
kubectl exec -it deploy/postgres -n myapp -- \
  psql -U appuser -d myappdb -c "SELECT version();"

# Create initial schema
kubectl exec -it deploy/postgres -n myapp -- \
  psql -U appuser -d myappdb -c "
    CREATE TABLE IF NOT EXISTS items (
      id SERIAL PRIMARY KEY,
      name VARCHAR(255) NOT NULL,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    INSERT INTO items (name) VALUES ('Test Item 1'), ('Test Item 2');
  "
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Persistent Volumes | Official Docs | 45 min | https://kubernetes.io/docs/concepts/storage/persistent-volumes/ |
| StatefulSets | Official Docs | 30 min | https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/ |
| Storage Classes | Official Docs | 20 min | https://kubernetes.io/docs/concepts/storage/storage-classes/ |
| PostgreSQL on K8s | Tutorial | 30 min | https://www.kubernetesbyday.com/post/deploying-postgresql-on-kubernetes |

---

### Day 5-6: Deploy Backend API

#### Backend Manifests

**1. ConfigMap**

```yaml
# k8s/backend/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: myapp
data:
  NODE_ENV: "production"
  PORT: "3000"
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  DB_NAME: "myappdb"
```

**2. Deployment with Init Container**

```yaml
# k8s/backend/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      # Init container waits for database to be ready
      initContainers:
        - name: wait-for-postgres
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              until nc -z postgres-service 5432; do
                echo "Waiting for PostgreSQL..."
                sleep 2
              done
              echo "PostgreSQL is ready!"
      
      containers:
        - name: backend
          image: myapp-backend:1.0
          imagePullPolicy: Never    # Use local image (minikube)
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: backend-config
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: POSTGRES_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: POSTGRES_PASSWORD
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
```

**3. Service**

```yaml
# k8s/backend/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: myapp
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 3000
      targetPort: 3000
```

**Deploy Backend:**

```bash
kubectl apply -f k8s/backend/

# Watch pods (init container runs first)
kubectl get pods -n myapp -w

# Check logs
kubectl logs -f deploy/backend -n myapp

# Test API internally
kubectl run curl-test --image=curlimages/curl -it --rm -n myapp -- \
  curl http://backend-service:3000/api/items
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Init Containers | Official Docs | 20 min | https://kubernetes.io/docs/concepts/workloads/pods/init-containers/ |
| Pod Lifecycle | Official Docs | 30 min | https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/ |
| Environment Variables | Official Docs | 15 min | https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/ |

---

### Day 7: Deploy Frontend

#### Frontend Manifests

**1. ConfigMap (Nginx config)**

```yaml
# k8s/frontend/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-nginx-config
  namespace: myapp
data:
  default.conf: |
    server {
        listen 80;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;

        location / {
            try_files $uri $uri/ /index.html;
        }

        location /api {
            proxy_pass http://backend-service:3000;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
```

**2. Deployment**

```yaml
# k8s/frontend/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: myapp-frontend:1.0
          imagePullPolicy: Never
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "100m"
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: nginx-config
          configMap:
            name: frontend-nginx-config
```

**3. Service (NodePort for external access)**

```yaml
# k8s/frontend/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: myapp
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080        # Access via minikube IP:30080
```

**Deploy and Access:**

```bash
kubectl apply -f k8s/frontend/

# Get all resources
kubectl get all -n myapp

# Access the application
minikube service frontend-service -n myapp --url
# Opens browser to http://<minikube-ip>:30080
```

---

### Week 3 Checkpoint

By end of Week 3, you should have:

- [ ] Containerized React and Node.js applications
- [ ] PostgreSQL running with persistent storage
- [ ] Backend API connected to the database
- [ ] Frontend served via Nginx, proxying to backend
- [ ] All three tiers communicating properly

---

## Week 4: Networking Deep Dive - Ingress & DNS

### Learning Objectives

- Understand Kubernetes networking model
- Configure Ingress for HTTP routing
- Set up TLS/HTTPS termination
- Implement network policies for security

### Day 1-2: Understanding K8s Networking

#### The Kubernetes Network Model

**Key Principles:**

1. Every Pod gets its own IP address
2. Pods can communicate with all other Pods without NAT
3. Nodes can communicate with all Pods without NAT
4. The IP a Pod sees for itself is the same IP others see for it

```
┌──────────────────────────────────────────────────────────────┐
│                       CLUSTER                                │
│  ┌──────────────────────────────────────────────────────────┐│
│  │                     Node 1                               ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      ││
│  │  │ Pod A       │  │ Pod B       │  │ Pod C       │      ││
│  │  │ 10.244.1.2  │  │ 10.244.1.3  │  │ 10.244.1.4  │      ││
│  │  └─────────────┘  └─────────────┘  └─────────────┘      ││
│  └──────────────────────────────────────────────────────────┘│
│  ┌──────────────────────────────────────────────────────────┐│
│  │                     Node 2                               ││
│  │  ┌─────────────┐  ┌─────────────┐                       ││
│  │  │ Pod D       │  │ Pod E       │                       ││
│  │  │ 10.244.2.2  │  │ 10.244.2.3  │                       ││
│  │  └─────────────┘  └─────────────┘                       ││
│  └──────────────────────────────────────────────────────────┘│
│                                                              │
│  All Pods can reach each other directly via their IPs       │
└──────────────────────────────────────────────────────────────┘
```

#### Service Types Deep Dive

| Type | How It Works | When to Use |
|------|--------------|-------------|
| **ClusterIP** | Virtual IP only routable within cluster | Pod-to-Pod communication |
| **NodePort** | ClusterIP + port on every node (30000-32767) | Development, debugging |
| **LoadBalancer** | NodePort + cloud provider LB | Production external access |
| **ExternalName** | CNAME DNS record | Connect to external services |

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Cluster Networking | Official Docs | 30 min | https://kubernetes.io/docs/concepts/cluster-administration/networking/ |
| Service Types | Official Docs | 20 min | https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types |
| K8s Networking Deep Dive | Blog | 45 min | https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/ |
| Life of a Packet | Video | 35 min | https://www.youtube.com/watch?v=0Omvgd7Hg1I |

---

### Day 3-4: Ingress Controller

**What is Ingress?**

Ingress exposes HTTP/HTTPS routes from outside the cluster to Services within. Think of it as a reverse proxy (like Nginx) managed by Kubernetes.

```
                    ┌─────────────────────────────────────────┐
                    │              CLUSTER                    │
Internet ──────────►│ Ingress Controller (nginx)             │
                    │        │                                │
                    │        ├── /api/* ──► backend-service   │
                    │        │                                │
                    │        └── /* ────► frontend-service    │
                    └─────────────────────────────────────────┘
```

#### Enable Ingress on minikube

```bash
# Enable NGINX Ingress controller
minikube addons enable ingress

# Verify it's running
kubectl get pods -n ingress-nginx

# Wait until the controller is ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

#### Configure Ingress

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.local          # Add to /etc/hosts
      http:
        paths:
          # API routes to backend
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 3000
          
          # Everything else to frontend
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

```bash
# Apply ingress
kubectl apply -f k8s/ingress.yaml

# Get minikube IP
minikube ip

# Add to /etc/hosts (replace <minikube-ip>)
echo "$(minikube ip) myapp.local" | sudo tee -a /etc/hosts

# Access your app
curl http://myapp.local
open http://myapp.local
```

#### TLS/HTTPS Configuration

```yaml
# k8s/ingress-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.local
      secretName: myapp-tls-secret   # Contains TLS cert
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 3000
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

**Create Self-Signed Certificate (for development):**

```bash
# Generate self-signed cert
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.local"

# Create secret
kubectl create secret tls myapp-tls-secret \
  --cert=tls.crt --key=tls.key -n myapp

# Apply TLS ingress
kubectl apply -f k8s/ingress-tls.yaml
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Ingress | Official Docs | 45 min | https://kubernetes.io/docs/concepts/services-networking/ingress/ |
| Ingress Controllers | Official Docs | 20 min | https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/ |
| NGINX Ingress | GitHub Docs | 30 min | https://kubernetes.github.io/ingress-nginx/ |
| cert-manager | Official Docs | 30 min | https://cert-manager.io/docs/ |

---

### Day 5-6: Network Policies

**What are Network Policies?**

By default, all Pods can communicate with each other. Network Policies let you control traffic flow - like firewall rules for Pods.

```yaml
# k8s/network-policies.yaml

# Deny all ingress traffic to database (default deny)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-to-postgres
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress: []               # Empty = deny all

---
# Allow only backend to access database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-postgres
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432

---
# Deny all egress from frontend except to backend and DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 3000
    # Allow DNS resolution
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

```bash
# Enable network policy support in minikube (requires CNI)
minikube start --cni=calico

# Apply policies
kubectl apply -f k8s/network-policies.yaml

# Test: Frontend should NOT reach postgres directly
kubectl exec -it deploy/frontend -n myapp -- \
  nc -zv postgres-service 5432
# Should fail

# Test: Backend CAN reach postgres
kubectl exec -it deploy/backend -n myapp -- \
  nc -zv postgres-service 5432
# Should succeed
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Network Policies | Official Docs | 30 min | https://kubernetes.io/docs/concepts/services-networking/network-policies/ |
| Network Policy Editor | Interactive Tool | - | https://editor.networkpolicy.io/ |
| Calico Network Policies | Calico Docs | 30 min | https://docs.tigera.io/calico/latest/network-policy/ |

---

### Week 4 Checkpoint

By end of Week 4, you should be able to:

- [ ] Explain the K8s networking model
- [ ] Configure Ingress for HTTP routing
- [ ] Set up TLS/HTTPS termination
- [ ] Implement Network Policies for security
- [ ] Access your 3-tier app via a proper domain name

---

## Week 5: Helm - Package Management for Kubernetes

### Learning Objectives

- Understand Helm architecture and concepts
- Convert raw manifests to Helm charts
- Use Helm templates and values
- Manage releases and rollbacks

### Day 1-2: Helm Fundamentals

#### What is Helm?

Helm is the package manager for Kubernetes - think npm for K8s. It allows you to:

- Package multiple K8s manifests as a single unit (Chart)
- Templatize configurations for different environments
- Version and manage releases
- Rollback to previous versions

**Helm Concepts:**

```
┌─────────────────────────────────────────────────────────────┐
│                        HELM                                 │
├─────────────────────────────────────────────────────────────┤
│ Chart     = Package (like npm package)                      │
│ Release   = Installed instance of a chart                   │
│ Repository= Collection of charts (like npm registry)        │
│ Values    = Configuration for a release (like .env)         │
└─────────────────────────────────────────────────────────────┘
```

#### Install Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version

# Add popular repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

#### Using Existing Charts

```bash
# Search for charts
helm search repo postgresql

# Show chart info
helm show readme bitnami/postgresql
helm show values bitnami/postgresql > postgres-values.yaml

# Install PostgreSQL using Helm
helm install my-postgres bitnami/postgresql \
  --namespace myapp \
  --create-namespace \
  --set auth.postgresPassword=mysecretpassword \
  --set auth.database=myappdb

# List releases
helm list -n myapp

# Get release status
helm status my-postgres -n myapp

# Upgrade with new values
helm upgrade my-postgres bitnami/postgresql \
  --namespace myapp \
  --set auth.postgresPassword=newsecretpassword \
  --set primary.persistence.size=10Gi

# View release history
helm history my-postgres -n myapp

# Rollback to previous version
helm rollback my-postgres 1 -n myapp

# Uninstall
helm uninstall my-postgres -n myapp
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Helm Quickstart | Official Docs | 30 min | https://helm.sh/docs/intro/quickstart/ |
| Using Helm | Official Docs | 45 min | https://helm.sh/docs/intro/using_helm/ |
| Artifact Hub | Chart Registry | Reference | https://artifacthub.io/ |
| Helm Best Practices | Official Docs | 30 min | https://helm.sh/docs/chart_best_practices/ |

---

### Day 3-5: Creating Your Own Helm Chart

#### Chart Structure

```bash
# Create a new chart
helm create myapp-chart

# Chart structure
myapp-chart/
├── Chart.yaml           # Chart metadata
├── values.yaml          # Default configuration values
├── charts/              # Dependencies (subcharts)
├── templates/           # Kubernetes manifest templates
│   ├── _helpers.tpl     # Template helpers
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   ├── NOTES.txt        # Post-install notes
│   └── tests/
│       └── test-connection.yaml
└── .helmignore          # Files to ignore
```

#### Chart.yaml

```yaml
# myapp-chart/Chart.yaml
apiVersion: v2
name: myapp
description: A Helm chart for my 3-tier fullstack application
type: application
version: 0.1.0           # Chart version (bump when chart changes)
appVersion: "1.0.0"      # Application version

maintainers:
  - name: Your Name
    email: you@example.com

dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

#### values.yaml

```yaml
# myapp-chart/values.yaml

# Global settings
global:
  environment: production

# Backend configuration
backend:
  replicaCount: 3
  image:
    repository: myapp-backend
    pullPolicy: IfNotPresent
    tag: "1.0.0"
  
  service:
    type: ClusterIP
    port: 3000
  
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi
  
  env:
    NODE_ENV: production
    LOG_LEVEL: info

# Frontend configuration  
frontend:
  replicaCount: 2
  image:
    repository: myapp-frontend
    pullPolicy: IfNotPresent
    tag: "1.0.0"
  
  service:
    type: ClusterIP
    port: 80
  
  resources:
    limits:
      cpu: 100m
      memory: 128Mi
    requests:
      cpu: 50m
      memory: 64Mi

# Ingress configuration
ingress:
  enabled: true
  className: nginx
  annotations: {}
  hosts:
    - host: myapp.local
      paths:
        - path: /api
          pathType: Prefix
          service: backend
        - path: /
          pathType: Prefix
          service: frontend
  tls: []

# Autoscaling
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# PostgreSQL subchart values
postgresql:
  enabled: true
  auth:
    postgresPassword: ""     # Set via --set or secrets
    database: myappdb
  primary:
    persistence:
      size: 5Gi
```

#### Backend Deployment Template

```yaml
# myapp-chart/templates/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}-backend
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: backend
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.backend.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: backend
  template:
    metadata:
      annotations:
        # Force rolling update when config changes
        checksum/config: {{ include (print $.Template.BasePath "/backend-configmap.yaml") . | sha256sum }}
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: backend
    spec:
      initContainers:
        - name: wait-for-db
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              until nc -z {{ .Release.Name }}-postgresql 5432; do
                echo "Waiting for database..."
                sleep 2
              done
      containers:
        - name: backend
          image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
          imagePullPolicy: {{ .Values.backend.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.backend.service.port }}
              protocol: TCP
          envFrom:
            - configMapRef:
                name: {{ include "myapp.fullname" . }}-backend-config
          env:
            - name: DB_HOST
              value: "{{ .Release.Name }}-postgresql"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-postgresql
                  key: postgres-password
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            {{- toYaml .Values.backend.resources | nindent 12 }}
```

#### Template Helpers

```yaml
# myapp-chart/templates/_helpers.tpl
{{/*
Expand the name of the chart.
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Chart label
*/}}
{{- define "myapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}
```

#### Deploy Your Chart

```bash
# Lint the chart
helm lint myapp-chart/

# Dry-run to see generated manifests
helm install myapp myapp-chart/ \
  --namespace myapp \
  --create-namespace \
  --dry-run --debug

# Install the chart
helm install myapp myapp-chart/ \
  --namespace myapp \
  --create-namespace \
  --set postgresql.auth.postgresPassword=secretpassword

# Upgrade with new values
helm upgrade myapp myapp-chart/ \
  --namespace myapp \
  --set backend.replicaCount=5

# Package for distribution
helm package myapp-chart/
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Chart Template Guide | Official Docs | 1 hr | https://helm.sh/docs/chart_template_guide/ |
| Built-in Objects | Official Docs | 20 min | https://helm.sh/docs/chart_template_guide/builtin_objects/ |
| Template Functions | Official Docs | 30 min | https://helm.sh/docs/chart_template_guide/function_list/ |
| Subcharts and Dependencies | Official Docs | 30 min | https://helm.sh/docs/chart_template_guide/subcharts_and_globals/ |

---

### Week 5 Checkpoint

By end of Week 5, you should be able to:

- [ ] Install and use Helm to deploy existing charts
- [ ] Create your own Helm chart from scratch
- [ ] Use Helm templates and values for configuration
- [ ] Manage releases with upgrade and rollback
- [ ] Package and share your charts

---

## Week 6: CI/CD with GitHub Actions

### Learning Objectives

- Set up automated builds with GitHub Actions
- Push images to a container registry
- Implement automated deployments to Kubernetes
- Configure staging and production environments

### Day 1-2: GitHub Actions Fundamentals

#### Workflow Structure

```yaml
# .github/workflows/ci.yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json
      
      - name: Install dependencies
        run: npm ci
        working-directory: ./backend
      
      - name: Run tests
        run: npm test
        working-directory: ./backend
      
      - name: Run linting
        run: npm run lint
        working-directory: ./backend

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-backend
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}
      
      - name: Build and push Backend
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| GitHub Actions Docs | Official Docs | 1 hr | https://docs.github.com/en/actions |
| Workflow Syntax | Reference | 30 min | https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions |
| Docker Build Push Action | GitHub | 20 min | https://github.com/docker/build-push-action |
| GitHub Container Registry | Docs | 20 min | https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry |

---

### Day 3-4: Kubernetes Deployment Automation

#### Deploy to Kubernetes with kubectl

```yaml
# .github/workflows/deploy.yaml
name: Deploy to Kubernetes

on:
  workflow_run:
    workflows: ["CI/CD Pipeline"]
    types: [completed]
    branches: [main]

jobs:
  deploy-staging:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'
      
      - name: Configure kubectl
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > ~/.kube/config
      
      - name: Update image tag
        run: |
          cd k8s/overlays/staging
          kustomize edit set image myapp-backend=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-backend:${{ github.sha }}
      
      - name: Deploy to staging
        run: |
          kubectl apply -k k8s/overlays/staging
          kubectl rollout status deployment/backend -n staging --timeout=300s

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Configure kubectl
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > ~/.kube/config
      
      - name: Deploy to production
        run: |
          kubectl apply -k k8s/overlays/production
          kubectl rollout status deployment/backend -n production --timeout=300s
```

#### Using Kustomize for Environment Overlays

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── staging/
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   └── patches/
    │       └── replicas.yaml
    └── production/
        ├── kustomization.yaml
        ├── namespace.yaml
        └── patches/
            ├── replicas.yaml
            └── resources.yaml
```

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

```yaml
# k8s/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: staging

resources:
  - ../../base
  - namespace.yaml

patches:
  - path: patches/replicas.yaml

images:
  - name: myapp-backend
    newName: ghcr.io/myorg/myapp-backend
    newTag: latest
```

```yaml
# k8s/overlays/staging/patches/replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Kustomize | Official Docs | 45 min | https://kustomize.io/ |
| GitHub Environments | Docs | 20 min | https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment |
| Deployment Strategies | Blog | 30 min | https://www.weave.works/blog/kubernetes-deployment-strategies |

---

### Day 5-7: GitOps with ArgoCD (Alternative Approach)

#### What is GitOps?

GitOps is a way of implementing Continuous Deployment for cloud native applications. It works by using Git as a single source of truth for declarative infrastructure and applications.

```
┌──────────────────────────────────────────────────────────────┐
│                      GitOps Flow                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Developer ──► Git Push ──► Git Repository                   │
│                                    │                         │
│                                    ▼                         │
│                              ArgoCD watches                  │
│                                    │                         │
│                                    ▼                         │
│                          Syncs to K8s Cluster                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access UI at https://localhost:8080 (user: admin)
```

#### Create ArgoCD Application

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/yourusername/myapp-k8s.git
    targetRevision: HEAD
    path: k8s/overlays/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true           # Delete resources removed from Git
      selfHeal: true        # Revert manual changes
    syncOptions:
      - CreateNamespace=true
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| ArgoCD Getting Started | Official Docs | 30 min | https://argo-cd.readthedocs.io/en/stable/getting_started/ |
| GitOps Principles | Weave Works | 20 min | https://www.weave.works/technologies/gitops/ |
| ArgoCD Best Practices | Official Docs | 30 min | https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/ |
| GitOps with ArgoCD | Video | 45 min | https://www.youtube.com/watch?v=MeU5_k9ssrs |

---

### Week 6 Checkpoint

By end of Week 6, you should be able to:

- [ ] Set up GitHub Actions for CI/CD
- [ ] Build and push container images automatically
- [ ] Deploy to Kubernetes from CI/CD pipeline
- [ ] Use Kustomize for environment-specific configs
- [ ] Understand GitOps principles (optional: set up ArgoCD)

---

## Week 7: Monitoring with Prometheus & Grafana

### Learning Objectives

- Deploy the Prometheus monitoring stack
- Configure metrics collection from your applications
- Create Grafana dashboards for visualization
- Set up alerting rules

### Day 1-2: Prometheus Fundamentals

#### What is Prometheus?

Prometheus is an open-source monitoring and alerting toolkit. It:

- Scrapes metrics from configured targets
- Stores metrics in a time-series database
- Provides a query language (PromQL) for analysis
- Supports alerting via Alertmanager

```
┌──────────────────────────────────────────────────────────────┐
│                    Prometheus Architecture                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    scrapes    ┌──────────────────────────┐  │
│  │ Prometheus  │◄──────────────│ Your App /metrics        │  │
│  │   Server    │               │ Node Exporter            │  │
│  │             │               │ kube-state-metrics       │  │
│  └──────┬──────┘               └──────────────────────────┘  │
│         │                                                    │
│         │ stores                                             │
│         ▼                                                    │
│  ┌─────────────┐                                            │
│  │    TSDB     │                                            │
│  │ (Time Series│                                            │
│  │  Database)  │                                            │
│  └──────┬──────┘                                            │
│         │                                                    │
│         │ queries                                            │
│         ▼                                                    │
│  ┌─────────────┐    alerts     ┌─────────────┐              │
│  │   Grafana   │               │ Alertmanager│──► Slack     │
│  │ (Dashboard) │               │             │──► PagerDuty │
│  └─────────────┘               └─────────────┘──► Email     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### Install kube-prometheus-stack

```bash
# Add Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install the stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin123

# Verify installation
kubectl get pods -n monitoring

# Access Grafana
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
# Open http://localhost:3000 (user: admin, password: admin123)

# Access Prometheus UI
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
# Open http://localhost:9090
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Prometheus Overview | Official Docs | 30 min | https://prometheus.io/docs/introduction/overview/ |
| Data Model | Official Docs | 20 min | https://prometheus.io/docs/concepts/data_model/ |
| PromQL Basics | Official Docs | 45 min | https://prometheus.io/docs/prometheus/latest/querying/basics/ |
| kube-prometheus-stack | GitHub | Reference | https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack |

---

### Day 3-4: Application Metrics

#### Add Prometheus Metrics to Node.js

```javascript
// backend/src/metrics.js
const promClient = require('prom-client');

// Create a Registry to register metrics
const register = new promClient.Registry();

// Add default metrics (CPU, memory, etc.)
promClient.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.5, 1, 2, 5]
});
register.registerMetric(httpRequestDuration);

const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});
register.registerMetric(httpRequestsTotal);

const activeConnections = new promClient.Gauge({
  name: 'active_connections',
  help: 'Number of active connections'
});
register.registerMetric(activeConnections);

module.exports = {
  register,
  httpRequestDuration,
  httpRequestsTotal,
  activeConnections
};
```

```javascript
// backend/src/index.js
const express = require('express');
const { register, httpRequestDuration, httpRequestsTotal } = require('./metrics');

const app = express();

// Metrics middleware
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const labels = {
      method: req.method,
      route: req.route?.path || req.path,
      status_code: res.statusCode
    };
    
    httpRequestDuration.observe(labels, duration);
    httpRequestsTotal.inc(labels);
  });
  
  next();
});

// Metrics endpoint for Prometheus to scrape
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// Your other routes...
app.get('/health', (req, res) => res.json({ status: 'ok' }));
```

#### Create ServiceMonitor

```yaml
# k8s/backend/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: backend-metrics
  namespace: myapp
  labels:
    release: prometheus        # Match Prometheus selector
spec:
  selector:
    matchLabels:
      app: backend
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
  namespaceSelector:
    matchNames:
      - myapp
```

```bash
# Apply ServiceMonitor
kubectl apply -f k8s/backend/servicemonitor.yaml

# Verify Prometheus is scraping
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
# Go to Status > Targets - should see your backend
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| prom-client npm | GitHub | 30 min | https://github.com/siimon/prom-client |
| Metric Types | Prometheus Docs | 20 min | https://prometheus.io/docs/concepts/metric_types/ |
| Instrumentation Best Practices | Prometheus Docs | 30 min | https://prometheus.io/docs/practices/instrumentation/ |
| ServiceMonitor | Prometheus Operator | 20 min | https://prometheus-operator.dev/docs/user-guides/getting-started/#deploying-a-sample-application |

---

### Day 5-6: Grafana Dashboards

#### Creating Dashboards

**Key Metrics for a Node.js App:**

```promql
# Request Rate (requests per second)
rate(http_requests_total[5m])

# Request Duration (95th percentile)
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Error Rate (5xx errors)
sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# Memory Usage
process_resident_memory_bytes

# CPU Usage
rate(process_cpu_seconds_total[5m])

# Active Connections
active_connections

# Pod Memory Usage
container_memory_usage_bytes{namespace="myapp", container="backend"}

# Pod CPU Usage
rate(container_cpu_usage_seconds_total{namespace="myapp", container="backend"}[5m])
```

**Dashboard JSON (import into Grafana):**

```json
{
  "dashboard": {
    "title": "MyApp Backend Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{job=\"backend\"}[5m])) by (route)",
            "legendFormat": "{{route}}"
          }
        ]
      },
      {
        "title": "Response Time (p95)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job=\"backend\"}[5m])) by (le, route))",
            "legendFormat": "{{route}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "singlestat",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{job=\"backend\", status_code=~\"5..\"}[5m])) / sum(rate(http_requests_total{job=\"backend\"}[5m])) * 100"
          }
        ],
        "format": "percent"
      }
    ]
  }
}
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Grafana Dashboards | Official Docs | 30 min | https://grafana.com/docs/grafana/latest/dashboards/ |
| Dashboard Best Practices | Grafana Blog | 20 min | https://grafana.com/blog/2022/06/06/grafana-dashboards-best-practices-and-how-to-build-them/ |
| Pre-built K8s Dashboards | Grafana | Reference | https://grafana.com/grafana/dashboards/?search=kubernetes |
| USE Method | Blog | 20 min | https://www.brendangregg.com/usemethod.html |
| RED Method | Weave Works | 20 min | https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/ |

---

### Day 7: Alerting

#### PrometheusRule for Alerting

```yaml
# k8s/monitoring/alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: myapp.rules
      rules:
        # High Error Rate
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{job="backend", status_code=~"5.."}[5m]))
            / sum(rate(http_requests_total{job="backend"}[5m])) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: High error rate detected
            description: "Error rate is {{ $value | humanizePercentage }} (> 5%)"
        
        # High Response Time
        - alert: HighLatency
          expr: |
            histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job="backend"}[5m])) by (le)) > 2
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: High latency detected
            description: "95th percentile latency is {{ $value }}s (> 2s)"
        
        # Pod Not Ready
        - alert: PodNotReady
          expr: |
            kube_pod_status_ready{namespace="myapp", condition="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: Pod not ready
            description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} is not ready"
        
        # High Memory Usage
        - alert: HighMemoryUsage
          expr: |
            container_memory_usage_bytes{namespace="myapp", container="backend"}
            / container_spec_memory_limit_bytes{namespace="myapp", container="backend"} > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: High memory usage
            description: "Container {{ $labels.container }} is using {{ $value | humanizePercentage }} of its memory limit"
```

```bash
# Apply alert rules
kubectl apply -f k8s/monitoring/alerts.yaml

# View alerts in Prometheus UI
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
# Go to Alerts tab
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Alerting Rules | Prometheus Docs | 30 min | https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/ |
| Alertmanager | Prometheus Docs | 30 min | https://prometheus.io/docs/alerting/latest/alertmanager/ |
| Alerting Best Practices | Blog | 20 min | https://prometheus.io/docs/practices/alerting/ |

---

### Week 7 Checkpoint

By end of Week 7, you should be able to:

- [ ] Deploy Prometheus and Grafana using Helm
- [ ] Add Prometheus metrics to your Node.js application
- [ ] Create ServiceMonitors to scrape your apps
- [ ] Build Grafana dashboards for visualization
- [ ] Configure alerting rules

---

## Week 8: Service Mesh & Production Best Practices

### Learning Objectives

- Understand service mesh concepts
- Deploy Istio or Linkerd basics
- Implement traffic management features
- Apply production hardening best practices

### Day 1-3: Service Mesh Introduction

#### What is a Service Mesh?

A service mesh is a dedicated infrastructure layer for handling service-to-service communication. It provides:

- **Traffic Management:** Load balancing, routing, retries, timeouts
- **Security:** mTLS, authorization policies
- **Observability:** Distributed tracing, metrics, logging
- **Resiliency:** Circuit breaking, fault injection

```
┌────────────────────────────────────────────────────────────────┐
│                    Without Service Mesh                        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   ┌─────────┐         ┌─────────┐         ┌─────────┐         │
│   │ Service │────────►│ Service │────────►│ Service │         │
│   │    A    │         │    B    │         │    C    │         │
│   └─────────┘         └─────────┘         └─────────┘         │
│                                                                │
│   Each service handles: retries, timeouts, TLS, tracing...    │
│                                                                │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│                    With Service Mesh                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────┐ │
│   │ ┌─────────────┐ │   │ ┌─────────────┐ │   │ ┌─────────┐ │ │
│   │ │  Service A  │ │   │ │  Service B  │ │   │ │Service C│ │ │
│   │ └─────────────┘ │   │ └─────────────┘ │   │ └─────────┘ │ │
│   │ ┌─────────────┐ │   │ ┌─────────────┐ │   │ ┌─────────┐ │ │
│   │ │   Sidecar   │◄├──►│ │   Sidecar   │◄├──►│ │ Sidecar │ │ │
│   │ │   Proxy     │ │   │ │   Proxy     │ │   │ │  Proxy  │ │ │
│   │ └─────────────┘ │   │ └─────────────┘ │   │ └─────────┘ │ │
│   └─────────────────┘   └─────────────────┘   └─────────────┘ │
│                                                                │
│   Sidecar handles: retries, timeouts, mTLS, tracing...        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

#### Install Istio (Recommended for Learning)

```bash
# Download istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio with demo profile
istioctl install --set profile=demo -y

# Verify installation
kubectl get pods -n istio-system

# Enable automatic sidecar injection for your namespace
kubectl label namespace myapp istio-injection=enabled

# Restart pods to inject sidecars
kubectl rollout restart deployment -n myapp

# Verify sidecars are injected (should see 2/2 containers)
kubectl get pods -n myapp

# Install Kiali (visualization), Jaeger (tracing), Prometheus, Grafana
kubectl apply -f samples/addons
kubectl rollout status deployment/kiali -n istio-system

# Access Kiali dashboard
istioctl dashboard kiali
```

#### Alternative: Install Linkerd (Simpler)

```bash
# Install linkerd CLI
brew install linkerd

# Check cluster compatibility
linkerd check --pre

# Install Linkerd
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# Verify
linkerd check

# Inject sidecars to your namespace
kubectl get deploy -n myapp -o yaml | linkerd inject - | kubectl apply -f -

# Install viz extension (dashboard)
linkerd viz install | kubectl apply -f -

# Access dashboard
linkerd viz dashboard
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| What is a Service Mesh? | Blog | 20 min | https://www.redhat.com/en/topics/microservices/what-is-a-service-mesh |
| Istio Concepts | Official Docs | 45 min | https://istio.io/latest/docs/concepts/what-is-istio/ |
| Linkerd Getting Started | Official Docs | 30 min | https://linkerd.io/2.14/getting-started/ |
| Service Mesh Comparison | Blog | 30 min | https://www.solo.io/topics/service-mesh/ |

---

### Day 4-5: Traffic Management

#### Istio Traffic Management

```yaml
# Virtual Service - Traffic Routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend-routing
  namespace: myapp
spec:
  hosts:
    - backend-service
  http:
    # Canary deployment: 90% to v1, 10% to v2
    - route:
        - destination:
            host: backend-service
            subset: v1
          weight: 90
        - destination:
            host: backend-service
            subset: v2
          weight: 10
      
      # Retry configuration
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,reset,connect-failure
      
      # Timeout
      timeout: 30s

---
# Destination Rule - Define subsets and load balancing
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend-destination
  namespace: myapp
spec:
  host: backend-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    
    # Circuit breaker
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

#### Fault Injection (Testing Resilience)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend-fault-injection
  namespace: myapp
spec:
  hosts:
    - backend-service
  http:
    - fault:
        # Inject 500ms delay for 10% of requests
        delay:
          percentage:
            value: 10
          fixedDelay: 500ms
        # Return 503 for 5% of requests
        abort:
          percentage:
            value: 5
          httpStatus: 503
      route:
        - destination:
            host: backend-service
```

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Istio Traffic Management | Official Docs | 45 min | https://istio.io/latest/docs/concepts/traffic-management/ |
| Virtual Services | Official Docs | 30 min | https://istio.io/latest/docs/reference/config/networking/virtual-service/ |
| Destination Rules | Official Docs | 30 min | https://istio.io/latest/docs/reference/config/networking/destination-rule/ |
| Canary Deployments | Blog | 30 min | https://istio.io/latest/blog/2017/0.1-canary/ |

---

### Day 6-7: Production Best Practices

#### Security Hardening

```yaml
# Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# Security Context for Pods
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: myapp
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      
      containers:
        - name: backend
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          
          # Mount writable directories as volumes
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: cache
              mountPath: /app/.cache
      
      volumes:
        - name: tmp
          emptyDir: {}
        - name: cache
          emptyDir: {}
```

#### Resource Management

```yaml
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15

---
# Pod Disruption Budget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-pdb
  namespace: myapp
spec:
  minAvailable: 2      # or maxUnavailable: 1
  selector:
    matchLabels:
      app: backend
```

#### Production Checklist

**Deployment Checklist:**

**Resource Management**
- [ ] Resource requests and limits set for all containers
- [ ] HPA configured for auto-scaling
- [ ] PDB configured to ensure availability during disruptions

**Security**
- [ ] Run as non-root user
- [ ] Read-only root filesystem
- [ ] No privilege escalation
- [ ] Network policies in place
- [ ] Secrets not stored in ConfigMaps
- [ ] Pod Security Standards enforced

**Reliability**
- [ ] Liveness probes configured
- [ ] Readiness probes configured
- [ ] Multiple replicas (minimum 3)
- [ ] Anti-affinity rules to spread pods across nodes
- [ ] Rolling update strategy configured

**Observability**
- [ ] Prometheus metrics exposed
- [ ] ServiceMonitor configured
- [ ] Alerting rules defined
- [ ] Dashboards created
- [ ] Log aggregation configured

**Networking**
- [ ] Ingress with TLS configured
- [ ] Network policies restrict traffic
- [ ] Service mesh (optional) for advanced traffic management

**CI/CD**
- [ ] Automated builds on push
- [ ] Image scanning for vulnerabilities
- [ ] Automated deployments to staging
- [ ] Manual approval for production
- [ ] Rollback strategy documented

#### Reading Materials

| Resource | Type | Time | Link |
|----------|------|------|------|
| Pod Security Standards | Official Docs | 30 min | https://kubernetes.io/docs/concepts/security/pod-security-standards/ |
| Security Best Practices | Official Docs | 45 min | https://kubernetes.io/docs/concepts/security/overview/ |
| HPA | Official Docs | 30 min | https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/ |
| Production Best Practices | Learnk8s | 1 hr | https://learnk8s.io/production-best-practices |
| K8s Security Checklist | GitHub | Reference | https://github.com/kubernetes-sigs/kubernetes-security-recommendations |

---

### Week 8 Checkpoint

By end of Week 8, you should be able to:

- [ ] Explain service mesh concepts and benefits
- [ ] Deploy Istio or Linkerd basics
- [ ] Configure traffic management (canary, retries, circuit breaking)
- [ ] Apply security hardening to your deployments
- [ ] Configure HPA and PDB for production resilience
- [ ] Complete a production readiness checklist

---

## Additional Resources & Certification Path

### Recommended Books

| Book | Author | Level |
|------|--------|-------|
| Kubernetes Up & Running | Kelsey Hightower | Beginner |
| Kubernetes in Action | Marko Luksa | Intermediate |
| Programming Kubernetes | Michael Hausenblas | Advanced |
| Cloud Native DevOps with Kubernetes | John Arundel | Intermediate |
| Kubernetes Patterns | Bilgin Ibryam | Intermediate |

### Video Courses

| Course | Platform | Duration |
|--------|----------|----------|
| Kubernetes for Developers | Pluralsight | 6 hrs |
| Certified Kubernetes Administrator | A Cloud Guru | 20 hrs |
| Kubernetes Mastery | Udemy (Bret Fisher) | 10 hrs |
| Kubernetes Deep Dive | Nigel Poulton | 8 hrs |

### Certification Path

```
┌─────────────────────────────────────────────────────────────┐
│                   CNCF Kubernetes Certifications            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────┐                                              │
│  │   KCNA    │  Kubernetes and Cloud Native Associate       │
│  │           │  Entry-level, multiple choice                │
│  └─────┬─────┘                                              │
│        │                                                    │
│        ▼                                                    │
│  ┌───────────┐     ┌───────────┐                           │
│  │   CKAD    │     │    CKA    │                           │
│  │           │     │           │                           │
│  │ Developer │     │   Admin   │                           │
│  │  focused  │     │  focused  │                           │
│  └─────┬─────┘     └─────┬─────┘                           │
│        │                 │                                  │
│        └────────┬────────┘                                  │
│                 ▼                                           │
│           ┌───────────┐                                     │
│           │    CKS    │                                     │
│           │           │                                     │
│           │ Security  │                                     │
│           │ Specialist│                                     │
│           └───────────┘                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| Certification | Focus | Prerequisites | Exam Format |
|---------------|-------|---------------|-------------|
| **KCNA** | Cloud Native fundamentals | None | Multiple choice |
| **CKAD** | Application development | None | Hands-on lab |
| **CKA** | Cluster administration | None | Hands-on lab |
| **CKS** | Security | CKA required | Hands-on lab |

### Practice Environments

| Resource | Type | Link |
|----------|------|------|
| Killercoda | Interactive Labs | https://killercoda.com/kubernetes |
| Play with Kubernetes | Online Playground | https://labs.play-with-k8s.com/ |
| KodeKloud | Paid Labs | https://kodekloud.com/ |
| Kubernetes Playground | Katacoda Alternative | https://www.katacoda.com/courses/kubernetes |

### Community & Staying Updated

| Resource | Type | Link |
|----------|------|------|
| CNCF Slack | Community | https://slack.cncf.io/ |
| Kubernetes Podcast | Podcast | https://kubernetespodcast.com/ |
| KubeWeekly | Newsletter | https://www.cncf.io/kubeweekly/ |
| r/kubernetes | Reddit | https://reddit.com/r/kubernetes |
| CNCF YouTube | Videos | https://www.youtube.com/c/cloudnativefdn |

---

## Summary

Congratulations on completing this Kubernetes learning plan! Here's what you've learned:

| Week | Topic | Key Skills |
|------|-------|------------|
| 1 | Fundamentals | Pods, Deployments, Services, kubectl |
| 2 | Configuration | ConfigMaps, Secrets, Namespaces, Probes |
| 3 | 3-Tier App | Dockerfiles, PVC, Service Discovery |
| 4 | Networking | Ingress, TLS, Network Policies |
| 5 | Helm | Charts, Templates, Values, Releases |
| 6 | CI/CD | GitHub Actions, Kustomize, GitOps |
| 7 | Monitoring | Prometheus, Grafana, Alerting |
| 8 | Service Mesh | Istio/Linkerd, Traffic Management, Security |

### Next Steps

1. **Practice:** Keep building and deploying applications
2. **Certify:** Consider CKAD or CKA certification
3. **Contribute:** Join the Kubernetes community
4. **Explore:** Cloud-managed K8s (EKS, GKE, AKS)
5. **Advance:** Operators, Custom Resources, Multi-cluster

Good luck on your infrastructure journey!
