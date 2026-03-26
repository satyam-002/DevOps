# 📘 Unit II – Deployment & Configuration of Kubernetes (2025 Edition)

> **Audience:** Absolute Beginners 👶 (No prior Kubernetes knowledge)  
> **Target:** Students / Teaching / Exams + Practicals  
> **Environment:** AWS EC2 (Ubuntu 22.04) + kubeadm  
> **Kubernetes Version:** v1.31 (Latest Stable)  
> **Focus:** Clear **WHAT**, **WHY**, **WHERE**, **HOW** for **every command**

---

## 🧭 What This Unit Covers

* Installing Kubernetes using **kubeadm**
* Master (Control Plane) & Worker Node setup
* Container Runtime (containerd) configuration
* kubectl configuration & context management
* Deploying applications using YAML
* Labels, Selectors & Annotations
* EKS Cluster setup (AWS Console + CLI – conceptual)
* Managing Kubernetes clusters
* **Role-Based Access Control (RBAC)**
* Secrets & ConfigMaps
* **Troubleshooting Common Issues**

---

## 1️⃣ What is Kubernetes?

### 🧠 Simple Explanation
Kubernetes is like a **smart manager** for your containers:
* Automatically restarts failed containers
* Distributes work across multiple machines
* Scales applications up/down
* Updates applications without downtime

### 🔹 What is kubeadm?
`kubeadm` is an **official Kubernetes tool** used to:
* Initialize a Kubernetes cluster
* Set up the Control Plane (Master Node)
* Join Worker Nodes to the cluster

👉 Used in **self-managed clusters** (VMs, EC2, on-premise)

---

## 2️⃣ Machines Required

| Machine | Role | Purpose | Minimum Resources |
|---------|------|---------|-------------------|
| EC2-1 | Control Plane (Master) | Manages cluster | 2 CPU, 4GB RAM |
| EC2-2 | Worker Node | Runs applications | 2 CPU, 4GB RAM |

> 💡 **Important:**
> * Both machines must be **Ubuntu 22.04**
> * Same VPC/Network
> * Security group must allow traffic between nodes

---

## 3️⃣ Prerequisites (Run on ALL Nodes)

> 📍 **Run on:** 🖥️ **Master** AND ⚙️ **Worker Nodes**

### 🧠 Why These Steps?

| Step | Purpose |
|------|---------|
| Disable swap | Kubernetes requires swap to be off |
| Load kernel modules | Enable container networking |
| Configure sysctl | Allow IP forwarding & bridge traffic |

### ▶️ Step 1: Disable Swap (CRITICAL!)

```bash
# Turn off swap immediately
sudo swapoff -a

# Disable swap permanently (survives reboot)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify swap is off
free -h
```

> ❗ **Why?** Kubernetes scheduler needs accurate memory allocation. Swap interferes with this.

---

### ▶️ Step 2: Load Required Kernel Modules

```bash
# Create configuration file
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load modules immediately
sudo modprobe overlay
sudo modprobe br_netfilter

# Verify modules are loaded
lsmod | grep br_netfilter
lsmod | grep overlay
```

> 🧠 **What are these?**
> * `overlay`: Container storage driver
> * `br_netfilter`: Bridge network filtering (pod-to-pod communication)

---

### ▶️ Step 3: Configure Networking Parameters

```bash
# Create sysctl configuration
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply settings immediately
sudo sysctl --system

# Verify settings
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward
```

> 🧠 **What does this do?**
> * Enables IP forwarding (routing between pods)
> * Allows iptables to see bridged traffic (required for networking)

---

## 4️⃣ Install Container Runtime (containerd)

> 📍 **Run on:** 🖥️ **Master** AND ⚙️ **Worker Nodes**

### 🧠 Why containerd?

Kubernetes needs a **container runtime** to actually run containers. Docker is deprecated in Kubernetes; containerd is the modern choice.

### ▶️ Install containerd

```bash
# Install prerequisites
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up Docker repository (for containerd.io package)
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
sudo apt update
sudo apt install -y containerd.io
```

### ▶️ Configure containerd for Kubernetes

```bash
# Create default configuration
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup (CRITICAL for Kubernetes!)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 🔎 Verify containerd is Running

```bash
# Check service status
sudo systemctl status containerd

# Verify socket exists
ls -l /run/containerd/containerd.sock
```

> ✅ You should see: **active (running)**

---

## 5️⃣ Install Kubernetes Components

> 📍 **Run on:** 🖥️ **Master** AND ⚙️ **Worker Nodes**

### 🧠 What are these components?

| Component | Purpose |
|-----------|---------|
| **kubelet** | Agent that runs on each node, manages pods |
| **kubeadm** | Tool to bootstrap cluster |
| **kubectl** | Command-line tool to control cluster |

### ▶️ Install Commands

```bash
# Update system
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes repository (v1.31 - Latest Stable)
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install Kubernetes tools
sudo apt update
sudo apt install -y kubelet kubeadm kubectl

# Prevent automatic updates (stability)
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable --now kubelet
```

### 🔎 Verify Installation

```bash
kubeadm version
kubectl version --client
kubelet --version
```

> ✅ You should see version **v1.31.x**

---

## 6️⃣ Initialize Control Plane (Master Node)

> 📍 **Run on:** 🖥️ **Master Node ONLY**

### 🧠 What happens during initialization?

1. **API Server** starts → brain of Kubernetes
2. **etcd** starts → database for cluster state
3. **Scheduler** starts → decides which pod goes where
4. **Controller Manager** starts → maintains desired state
5. Generates certificates for secure communication

### ▶️ Initialize Cluster

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

> 🧠 **What is `--pod-network-cidr`?**  
> This defines the IP range for pods. Required for Flannel networking.

### 📋 Expected Output

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.45.32:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:1234567890abcdef...
```

> 📌 **CRITICAL:** Copy the `kubeadm join` command! You'll need it for worker nodes.

---

## 7️⃣ Configure kubectl (Master Node)

> 📍 **Run on:** 🖥️ **Master Node**

### 🧠 Why needed?

`kubectl` needs credentials to talk to the API Server. These commands set up your access.

### ▶️ Commands

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 🔎 Verify

```bash
kubectl get nodes
```

**Expected Output:**
```
NAME                STATUS     ROLES           AGE   VERSION
ip-172-31-45-32     NotReady   control-plane   30s   v1.31.x
```

> ❗ **NotReady is normal!** Node becomes Ready after installing networking.

---

## 8️⃣ Install Pod Network (Flannel)

> 📍 **Run on:** 🖥️ **Master Node**

### 🧠 Why needed?

Kubernetes does NOT provide networking by default. You must install a **CNI (Container Network Interface)** plugin.

**Flannel** is simple and perfect for learning.

### ▶️ Install Flannel

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### 🔎 Verify

```bash
# Check node status (should become Ready in 1-2 minutes)
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system
```

**Expected Output:**
```
NAME                STATUS   ROLES           AGE   VERSION
ip-172-31-45-32     Ready    control-plane   2m    v1.31.x
```

> ✅ Status should now be **Ready**

---

## 9️⃣ Join Worker Node

> 📍 **Run on:** ⚙️ **Worker Node ONLY**

### ▶️ Join Command

Use the command from `kubeadm init` output:

```bash
sudo kubeadm join 172.31.45.32:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:1234567890abcdef...
```

### 🔎 Verify (Run on Master)

```bash
kubectl get nodes
```

**Expected Output:**
```
NAME                STATUS   ROLES           AGE   VERSION
ip-172-31-45-32     Ready    control-plane   5m    v1.31.x
ip-172-31-50-100    Ready    <none>          1m    v1.31.x
```

> ✅ Both nodes should show **Ready**

---

### 🔧 If Join Token Expired

Tokens expire after 24 hours. Generate a new one:

```bash
# On Master Node
kubeadm token create --print-join-command
```

---

## 🔟 kubectl Basics & Context

> 📍 **Run on:** 🖥️ **Master / Admin Machine**

### 🧠 Basic kubectl Commands

```bash
# View cluster information
kubectl cluster-info

# List all nodes
kubectl get nodes

# List all namespaces
kubectl get namespaces

# List pods in all namespaces
kubectl get pods -A

 #  List pods in default namespace
kubectl get pods

# Describe a node (detailed info)
kubectl describe node <node-name>
```

### 🔹 Contexts (Multiple Clusters)

```bash
# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>
```

---

## 1️⃣1️⃣ Deploying Applications using YAML

### 🧠 Why YAML?

* **Declarative:** You describe what you want, Kubernetes makes it happen
* **Repeatable:** Same YAML = same result
* **Version Controlled:** Can be stored in Git

---

### 📄 Example: Nginx Deployment
nano create nginx-deployment.yaml
Create file: `nginx-deployment.yaml`
  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
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
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 🧠 Understanding the YAML

| Field | Purpose |
|-------|---------|
| `apiVersion` | Kubernetes API version |
| `kind` | Type of resource (Deployment, Service, Pod) |
| `metadata` | Name and labels |
| `spec.replicas` | Number of pod copies |
| `spec.selector` | How Deployment finds its Pods |
| `spec.template` | Pod definition |

### ▶️ Deploy Application

```bash
# Apply the YAML
kubectl apply -f nginx-deployment.yaml

# Check deployment
kubectl get deployments

# Check pods
kubectl get pods

# Check detailed info
kubectl describe deployment nginx-deploy
```

**Expected Output:** 

```
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   3/3     3            3           30s
```

---

### 📄 Example: Service (Expose Application)

   Create file: `nginx-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

```bash
# Apply service
kubectl apply -f nginx-service.yaml

# Check service
kubectl get svc
  
# Access application
curl http://<NODE-IP>:30080
```

---

## 1️⃣2️⃣ Labels, Selectors & Annotations

### 🔹 Labels (Identification & Grouping)

Labels are **key-value pairs** attached to objects.

```yaml
metadata:
  labels:
    app: frontend
    env: production
    tier: web
```

### 🔹 Selectors (Matching/Filtering)

Used to find objects with specific labels.

```yaml
selector:
  matchLabels:
    app: frontend
    env: production
```

**Command Examples:**
```bash
# Get pods with specific label
kubectl get pods -l app=nginx

# Get pods with multiple labels
kubectl get pods -l app=nginx,env=prod

# Get all pods with label 'app'
kubectl get pods -l app
```

### 🔹 Annotations (Metadata)

Non-identifying information (not used for selection).

```yaml
metadata:
  annotations:
    owner: "devops-team"
    contact: "devops@company.com"
    created-by: "John Doe"
```

---

## 1️⃣3️⃣ Managing Kubernetes Cluster

### 🔹 Scaling Applications

```bash
# Scale deployment to 5 replicas
kubectl scale deployment nginx-deploy --replicas=5

# Verify
kubectl get pods
```

### 🔹 Updating Applications

```bash
# Update image version
kubectl set image deployment/nginx-deploy nginx=nginx:1.21

# Check rollout status
kubectl rollout status deployment/nginx-deploy

# View rollout history
kubectl rollout history deployment/nginx-deploy
```

### 🔹 Rolling Back

```bash
# Undo last deployment
kubectl rollout undo deployment/nginx-deploy

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deploy --to-revision=2
```

### 🔹 Deleting Resources

```bash
# Delete deployment
kubectl delete deployment nginx-deploy

# Delete service
kubectl delete service nginx-service

# Delete using YAML file
kubectl delete -f nginx-deployment.yaml

# Delete all resources in namespace
kubectl delete all --all -n default
```

---

## 1️⃣4️⃣ Role-Based Access Control (RBAC)

### 🧠 What is RBAC?

RBAC controls:  
> **Who** can do **what** on **which resource**

### 🧩 RBAC Components

```
User → Role → Permissions
  ↓
RoleBinding
```

| Component | Purpose |
|-----------|---------|
| **Role** | Defines permissions (what actions) |
| **RoleBinding** | Links user to role |
| **ClusterRole** | Cluster-wide permissions |
| **ClusterRoleBinding** | Cluster-wide user-role link |

---

### 🎯 Example: User Can READ Pods ONLY

#### Step 1️⃣ Create Role

Create file: `pod-reader-role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

```bash
kubectl apply -f pod-reader-role.yaml
```

#### Step 2️⃣ Create RoleBinding

Create file: `pod-reader-binding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f pod-reader-binding.yaml
```

#### Step 3️⃣ Test Permissions

```bash
# Check if user can list pods (should be YES)
kubectl auth can-i list pods --as dev-user

# Check if user can delete pods (should be NO)
kubectl auth can-i delete pods --as dev-user

# Check if user can create deployments (should be NO)
kubectl auth can-i create deployments --as dev-user
```

---

### 📊 Common RBAC Verbs

| Verb | Action |
|------|--------|
| `get` | Read single resource |
| `list` | Read multiple resources |
| `watch` | Watch for changes |
| `create` | Create new resource |
| `update` | Modify existing resource |
| `delete` | Delete resource |
| `*` | All actions |

---

## 1️⃣5️⃣ ConfigMaps & Secrets

### 🔹 ConfigMap (Non-Sensitive Configuration)

**Use Case:** Application settings, environment variables, config files

#### Create ConfigMap

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info

# From file
kubectl create configmap nginx-config \
  --from-file=nginx.conf

# Verify
kubectl get configmap
kubectl describe configmap app-config
```

#### Using ConfigMap in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
```

---

### 🔹 Secret (Sensitive Data)

**Use Case:** Passwords, API keys, certificates

#### Create Secret

```bash
# From literal values
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=MyP@ssw0rd

# From file
kubectl create secret generic tls-secret \
  --from-file=tls.crt \
  --from-file=tls.key

# Verify
kubectl get secrets
```

#### Using Secret in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

> 🔒 **Security Note:** Secrets are base64 encoded, not encrypted! Use encryption at rest in production.

---

## ☁️ Amazon EKS (Elastic Kubernetes Service)

### 🧠 What is EKS?

* **AWS manages:** Control Plane (API Server, etcd, scheduler)
* **You manage:** Worker Nodes
* **Benefits:** High availability, automatic updates, AWS integration

### 🔹 EKS Setup (Conceptual)

#### Prerequisites

```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install eksctl
curl --silent --location "https://github.com/weksctl/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

#### Create EKS Cluster

```bash
# Create cluster (takes 10-15 minutes)
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3

# Configure kubectl
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Verify
kubectl get nodes
```

#### Delete Cluster

```bash
eksctl delete cluster --name my-cluster --region us-east-1
```

---

## 🚨 Troubleshooting Common Issues

### ❌ Issue 1: Container Runtime Not Running

**Error:**
```
[ERROR CRI]: container runtime is not running
```

**Solution:**
```bash
# Check containerd status
sudo systemctl status containerd

# If not running, start it
sudo systemctl restart containerd

# Verify socket exists
ls -l /run/containerd/containerd.sock
```

---

### ❌ Issue 2: Previous Installation Exists

**Error:**
```
[ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]: file already exists
```

**Solution:**
```bash
# Reset kubeadm
sudo kubeadm reset -f

# Clean up files
sudo rm -rf /etc/cni/net.d
sudo rm -rf /etc/kubernetes/
sudo rm -rf ~/.kube/
sudo rm -rf /var/lib/etcd

# Reset iptables
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# Reinitialize
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

---

### ❌ Issue 3: Node Stays NotReady

**Possible Causes:**
1. CNI not installed
2. Containerd not configured properly

**Solution:**
```bash
# Check node status
kubectl describe node <node-name>

# Check containerd
sudo systemctl status containerd

# Reinstall Flannel
kubectl delete -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

---

### ❌ Issue 4: Pods Not Starting

```bash
# Check pod status
kubectl get pods

# Describe pod for details
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>

# Check events
kubectl get events --sort-by='.lastTimestamp'
```

---

## ✅ Unit II Summary Checklist

- [ ] Understand what Kubernetes does
- [ ] Know the difference between Master and Worker nodes
- [ ] Can install containerd and configure it
- [ ] Can initialize a cluster with kubeadm
- [ ] Can join worker nodes
- [ ] Can deploy applications using YAML
- [ ] Understand Labels and Selectors
- [ ] Can create RBAC rules (Roles and RoleBindings)
- [ ] Can use ConfigMaps and Secrets
- [ ] Know basic troubleshooting steps

---

## 🎯 Teaching Tips

> **For Students:**
> 1. **Practice** on free AWS tier or local VMs
> 2. **Break everything** - learn by fixing mistakes
> 3. **Start with single-node cluster** before adding workers
> 4. **Master YAML syntax** - 80% of Kubernetes is YAML

> **For Teachers:**
> 1. Show live demos - seeing is believing
> 2. Have students do troubleshooting exercises
> 3. Focus on YAML structure and RBAC - these are exam favorites
> 4. Use `kubectl explain` command to teach API objects

---

## 📚 Quick Reference Commands

```bash
# Cluster Management
kubectl cluster-info
kubectl get nodes
kubectl get all -A

# Application Deployment
kubectl apply -f <file.yaml>
kubectl get deployments
kubectl get pods
kubectl scale deployment <name> --replicas=5

# Troubleshooting
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/bash

# RBAC
kubectl auth can-i <verb> <resource> --as <user>
kubectl get roles
kubectl get rolebindings

# ConfigMaps & Secrets
kubectl get configmaps
kubectl get secrets
kubectl describe configmap <name>
```

---

📌 **End of Unit II - Kubernetes Deployment & Configuration**

*Last Updated: December 2025*  
*Kubernetes Version: v1.31 (Latest Stable)*
