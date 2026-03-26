# 🚢 Helm for Beginners – Complete Hands-On Study Material (Using Minikube)

> 🎯 **Goal**: This guide is written for **absolute beginners**.  
If you **blindly follow each step in order**, you will successfully:
- Run Minikube
- Understand Helm concepts
- Create, install, upgrade, rollback, and delete a Helm chart

No prior Helm knowledge required.

---

## 📌 Prerequisites (VERY IMPORTANT)

You must have:
- **Windows 11 / macOS / Linux**
- **Docker Desktop installed & running**
- **Internet connection**

---

## 1️⃣ What is Helm? (1-Minute Understanding)

**Helm is a package manager for Kubernetes.**

Kubernetes apps need many YAML files:
- Deployment
- Service
- ConfigMap
- Ingress

👉 Helm bundles all these into a **Chart** and manages them easily.

📦 Think of Helm as:
- `apt` for Ubuntu
- `npm` for Node.js
- `pip` for Python

---

## 2️⃣ Install Required Tools

### ✅ Install kubectl
```bash
kubectl version --client
````

If not installed, install using Docker Desktop (Settings → Kubernetes).

---

### ✅ Install Minikube

#### Windows

```powershell
winget install minikube
```

#### macOS

```bash
brew install minikube
```

#### Linux

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Verify:

```bash
minikube version
```

---

### ✅ Install Helm

#### Windows

```powershell
winget install Helm.Helm
```

#### macOS

```bash
brew install helm
```

#### Linux

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

---

## 3️⃣ Start Minikube (MANDATORY)

```bash
minikube start
```

Check cluster:

```bash
kubectl get nodes
```

Expected output:

```text
NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    control  ...
```

---

## 4️⃣ Minikube Basic Commands (Remember These)

```bash
minikube status
minikube stop
minikube delete
minikube dashboard
```

Open dashboard:

```bash
minikube dashboard
```

---

## 5️⃣ Helm Core Terminology (Must Know)

| Term        | Meaning               |
| ----------- | --------------------- |
| Helm        | Tool                  |
| Chart       | Application package   |
| Release     | Installed chart       |
| values.yaml | Configuration         |
| templates   | Kubernetes YAML files |

---

## 6️⃣ Create Your First Helm Chart

```bash
helm create my-first-chart
```

Folder structure:

```text
my-first-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
```

---

## 7️⃣ Understand values.yaml (CONFIG FILE)

Open `values.yaml` and **edit only these lines**:

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: latest

service:
  type: NodePort
  port: 80
```

💡 This file controls behavior of the app.

---

## 8️⃣ Install the Helm Chart (DEPLOY APP)

```bash
helm install nginx-release my-first-chart
```

Verify Helm release:

```bash
helm list
```

Verify Kubernetes:

```bash
kubectl get pods
kubectl get svc
```

---

## 9️⃣ Access the Application

Get Minikube IP:

```bash
minikube ip
```

Get NodePort:

```bash
kubectl get svc
```

Open in browser:

```text
http://<MINIKUBE-IP>:<NODE-PORT>
```

You should see **NGINX Welcome Page** 🎉

---

## 🔟 Upgrade Application (MOST IMPORTANT HELM FEATURE)

Edit `values.yaml`:

```yaml
replicaCount: 3
```

Apply upgrade:

```bash
helm upgrade nginx-release my-first-chart
```

Check:

```bash
kubectl get pods
```

You will see **3 pods running** ✅

---

## 1️⃣1️⃣ Rollback to Previous Version

View history:

```bash
helm history nginx-release
```

Rollback:

```bash
helm rollback nginx-release 1
```

Check:

```bash
kubectl get pods
```

---

## 1️⃣2️⃣ Uninstall Helm Release (CLEAN DELETE)

```bash
helm uninstall nginx-release
```

Verify:

```bash
helm list
kubectl get pods
```

---

## 1️⃣3️⃣ Helm vs kubectl (Important Concept)

| Feature            | kubectl | Helm |
| ------------------ | ------- | ---- |
| YAML reuse         | ❌       | ✅    |
| Versioning         | ❌       | ✅    |
| Rollback           | ❌       | ✅    |
| One-command deploy | ❌       | ✅    |

---

## 1️⃣4️⃣ Complete Flow (Exam Friendly)

```text
Minikube → Kubernetes Cluster
Helm → Manages Kubernetes apps
Chart → App template
Release → Running app
values.yaml → Configuration
```

---

## 1️⃣5️⃣ Common Beginner Mistakes

❌ Minikube not running
❌ Docker Desktop stopped
❌ Wrong service type
❌ Editing template files instead of values.yaml

---

## 🎯 Final One-Line Definition

> **Helm is a Kubernetes package manager that simplifies application deployment using reusable and configurable charts.**

---

## ✅ You Have Successfully Learned Helm

You can now:

* Create charts
* Install apps
* Upgrade & rollback
* Manage Kubernetes easily

---
```
```
