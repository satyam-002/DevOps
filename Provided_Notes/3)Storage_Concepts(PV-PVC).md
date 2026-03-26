
---

# 📘 Kubernetes Beginner Guide

## Storage (PV, PVC), Resource Requests & Autoscaling

### (Minikube + Windows CMD Only)

---

## 🔰 Prerequisites (Tell Students)

Installed already:

* Docker Desktop (running)
* Minikube
* kubectl

Start cluster:

```cmd
minikube start
```

---

# 🧠 SECTION 1: Why Kubernetes Storage Exists

## ❌ Problem (Very Important)

In Kubernetes:

* Pods can restart anytime
* Containers are temporary
* Files inside container ❌ **DO NOT survive**

👉 If NGINX pod restarts → website files are lost

---

## 🏠 Real-Life Analogy (Remember This)

| Real Life    | Kubernetes                    |
| ------------ | ----------------------------- |
| Hard disk    | Persistent Volume (PV)        |
| Request form | Persistent Volume Claim (PVC) |
| Tenant       | Pod                           |

---

# 🔹 Kubernetes Storage Components (Simple)

| Component | Meaning             |
| --------- | ------------------- |
| PV        | Actual storage      |
| PVC       | Request for storage |
| Pod       | Uses the storage    |

---

# 2️⃣ Persistent Volume (PV)

## 📌 What is a PV?

> PV is a **real physical storage** created by admin
> It exists **independently of pods**

In Minikube:

* We use `hostPath`
* Storage exists on Minikube VM

---

## ✅ Step 1: Create Persistent Volume

📄 **pv.yaml**

``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/nginx-data
``` 

Apply:

```cmd
kubectl apply -f pv.yaml
```

Verify:

```cmd
kubectl get pv
```

Expected output:

```
nginx-pv   1Gi   Available
```

📌 **Key Point**
PV is created but **not used yet**

---

# 3️⃣ Persistent Volume Claim (PVC)

## ❓ Big Question:

### 👉 HOW does PVC know WHICH PV to use?

### 🧠 Kubernetes Matching Logic (IMPORTANT)

PVC looks for a PV that matches:

1. **Enough storage**
2. **Same access mode**
3. **Not already bound**
4. **Same StorageClass (if specified)**

👉 Kubernetes automatically binds them.

---

## ✅ Step 2: Create PVC

📄 **pvc.yaml**
 
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

Apply:

```cmd
kubectl apply -f pvc.yaml
```

Verify:

```cmd
kubectl get pvc
```

Expected:

```
nginx-pvc   Bound
```
---
## 🔍 Verify Binding (Very Important)

```cmd
kubectl get pv
```
You will see:

```
nginx-pv   1Gi   Bound   default/nginx-pvc
```

🎯 **Now you know:**

> PVC didn’t choose PV
> Kubernetes matched them automatically

---

# 🔗 PV–PVC Relationship (Visual Logic)
 
```
PVC asks → Kubernetes matches → PV binds
```

| PVC Request | PV Condition  |
| ----------- | ------------- |
| 500Mi       | PV has 1Gi    |
| RWO         | PV allows RWO |
| Available   | PV not used   |

---

# 4️⃣ Using PVC in NGINX Pod

Now we **attach storage to NGINX**

---

## ✅ Step 3: Create NGINX Pod Using PVC

📄 **nginx-pod.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-storage-pod
  labels:
    app: nginx-storage
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: nginx-volume
    persistentVolumeClaim:
      claimName: nginx-pvc
```

Apply:

```cmd
kubectl apply -f nginx-pod.yaml
```

Check:

```cmd
kubectl get pods
```

---

# 5️⃣ REAL DEMO: Custom index.html (PROOF)

## Step 4: Enter Pod

```cmd
kubectl exec -it nginx-storage-pod -- sh
```

## Step 5: Create Custom Page

```sh
echo "<h1>Hello from Persistent Volume</h1>" > /usr/share/nginx/html/index.html
exit
```

---

## Step 6: Expose NGINX

```cmd
kubectl expose pod nginx-storage-pod --type=NodePort --port=80
```

Get URL:

```cmd
minikube service nginx-storage-pod --url
```

👉 Open in browser
You’ll see:

```
Hello from Persistent Volume
```

---

# 🔥 FINAL STORAGE TEST (MOST IMPORTANT)

### Delete Pod

```cmd
kubectl delete pod nginx-storage-pod
```

### Recreate Pod

```cmd
kubectl apply -f nginx-pod.yaml
```

Open browser again.

🎉 **Website still works**
🎉 **File not deleted**

---

## 🧠 What Actually Happened?

| Action        | Result             |
| ------------- | ------------------ |
| Pod deleted   | Container gone     |
| PV remains    | Disk remains       |
| PVC remains   | Claim remains      |
| Pod recreated | Same disk attached |

---

# 6️⃣ Resource Requests & Limits

## ❓ Problem

One pod consumes:

* All CPU
* All memory
* Node crashes ❌

---

## 💡 Solution

Define:

* Minimum required
* Maximum allowed

---

## ✅ Deployment with Resources

📄 **nginx-resources.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-resources
spec:
  replicas: 1
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
        image: nginx
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

Apply:

```cmd
kubectl apply -f nginx-resources.yaml
```

---

## 🧠 What Kubernetes Enforces

| Resource | Behaviour  |
| -------- | ---------- |
| CPU      | Throttled  |
| Memory   | Pod killed |

---

# 7️⃣ Horizontal Pod Autoscaling (HPA)

## ❓ Problem

Traffic increases → app slow

---

## 💡 Solution

Increase **number of pods automatically**

---

## Step 1: Enable Metrics Server

```cmd
minikube addons enable metrics-server
```

---

## Step 2: Create Deployment

```cmd
kubectl create deployment hpa-demo --image=k8s.gcr.io/hpa-example
```

---

## Step 3: Create HPA

```cmd
kubectl autoscale deployment hpa-demo --cpu-percent=50 --min=1 --max=5
```

---

## Step 4: Observe

```cmd
kubectl get hpa
kubectl get pods
```

👉 Load ↑ → Pods ↑
👉 Load ↓ → Pods ↓

---

# 8️⃣ Vertical Pod Autoscaling (Concept Only)

| Feature           | HPA | VPA |
| ----------------- | --- | --- |
| Changes pod count | ✅   | ❌   |
| Changes pod size  | ❌   | ✅   |
| Pod restart       | ❌   | ✅   |

📌 **For beginners: concept only**

---

# ✅ FINAL TAKEAWAYS (VERY IMPORTANT)

| Concept | Remember This           |
| ------- | ----------------------- |
| PV      | Actual disk             |
| PVC     | Storage request         |
| Binding | Kubernetes auto-matches |
| Pod     | Uses PVC                |
| Storage | Lives outside pod       |
| HPA     | Scales pods             |
| Limits  | Protect node            |

---

Just say the word 👍
