# Lab: Deploying Python/Redis on Kubernetes with S3-Backed Storage

## 1. Overview
In this lab, you will orchestrate the application you built previously using **Kubernetes (Minikube)** on an AWS EC2 instance. You will also implement a **Container Storage Interface (CSI)** adapter to mount an **AWS S3 Bucket** as if it were a local disk drive attached to your pod.

### **Lab Objectives**
1.  Install **Minikube** (Single-node Kubernetes) on Amazon Linux.
2.  Create an **AWS S3 Bucket** and IAM Credentials.
3.  Install a **CSI S3 Driver** to allow Kubernetes to talk to S3.
4.  Deploy **Redis** and the **Python App** using Kubernetes Manifests.
5.  Verify that files written by the app appear in your S3 Bucket.

---

## 2. Phase 0: Instance Setup

We need to setup an EC2 instance for this lab with a size t2.micro at least.

## 2. Phase 1: Environment Setup (Amazon Linux)

We need to install the Kubernetes tools (`kubectl`) and the cluster environment (`minikube`).

### Step 1: Install Kubectl
Run the following commands to install the Kubernetes command-line tool:

```bash
# 1. Download the latest release
curl -LO "[https://dl.k8s.io/release/$(curl](https://dl.k8s.io/release/$(curl) -L -s [https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl](https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl)"

# 2. Make it executable
chmod +x kubectl

# 3. Move to path
sudo mv kubectl /usr/local/bin/

# 4. Verify
kubectl version --client
```

### Step 2: Install Minikube
Minikube runs a Kubernetes cluster inside a Docker container.
*Note: Ensure Docker is running (sudo service docker start).*

```bash
# 1. Download Minikube
curl -LO [https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64](https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64)

# 2. Install
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# 3. Start the Cluster (This takes a few minutes)
# We force it to use the 'root' user for Docker since we are on a lab VM
sudo minikube start --driver=docker --force
```

### Step 3: Verify the Cluster
```bash
sudo minikube kubectl -- get nodes
```
*Result: You should see one node named `minikube` with status `Ready`.*

*Alias Tip:* To avoid typing `sudo minikube kubectl --` every time, run:
`alias kubectl="sudo minikube kubectl --"`

---

## 3. Phase 2: AWS S3 Configuration

We need a bucket for storage and a user with keys so the Kubernetes cluster can access it.

### Step 1: Create a Bucket
Run this in your terminal (using AWS CLI):
```bash
# Choose a UNIQUE name (e.g., k8s-lab-storage-99281)
export BUCKET_NAME=k8s-lab-storage-$RANDOM
aws s3 mb s3://$BUCKET_NAME
echo "Bucket created: $BUCKET_NAME"
```

### Step 2: Create IAM User & Keys
For this lab, we will generate specific Access Keys to pass to the Kubernetes cluster.

```bash
# 1. Create User
aws iam create-user --user-name k8s-s3-user

# 2. Attach S3 Full Access Policy (For lab purposes only)
aws iam attach-user-policy --user-name k8s-s3-user --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# 3. Generate Keys and save to a file
aws iam create-access-key --user-name k8s-s3-user > s3_creds.json

# 4. Display Keys (You will need these in the next step!)
cat s3_creds.json
```

---

## 4. Phase 3: The S3 CSI Driver

Kubernetes does not know how to mount S3 buckets natively. We must install a **CSI Driver**. We will use a community-standard driver (`yandex-cloud/k8s-csi-s3`) that works with generic S3 buckets.

### Step 1: Create the Secret
Kubernetes needs your AWS keys to mount the bucket.
*Replace `YOUR_ACCESS_KEY` and `YOUR_SECRET_KEY` below with the values from `s3_creds.json`.*

```bash
# Export your keys (Copy/Paste from s3_creds.json)
export AWS_ACCESS_KEY_ID=YOUR_ACCESS_KEY
export AWS_SECRET_ACCESS_KEY=YOUR_SECRET_KEY

# Create the K8s Secret
kubectl create secret generic csi-s3-secret \
  --from-literal=accessKeyID=$AWS_ACCESS_KEY_ID \
  --from-literal=secretAccessKey=$AWS_SECRET_ACCESS_KEY \
  --namespace=kube-system
```

### Step 2: Install the Driver
We will apply the official manifests for the CSI driver.

```bash
kubectl apply -f [https://raw.githubusercontent.com/yandex-cloud/k8s-csi-s3/master/deploy/kubernetes/driver.yaml](https://raw.githubusercontent.com/yandex-cloud/k8s-csi-s3/master/deploy/kubernetes/driver.yaml)
kubectl apply -f [https://raw.githubusercontent.com/yandex-cloud/k8s-csi-s3/master/deploy/kubernetes/provisioner.yaml](https://raw.githubusercontent.com/yandex-cloud/k8s-csi-s3/master/deploy/kubernetes/provisioner.yaml)
kubectl apply -f [https://raw.githubusercontent.com/yandex-cloud/k8s-csi-s3/master/deploy/kubernetes/attacher.yaml](https://raw.githubusercontent.com/yandex-cloud/k8s-csi-s3/master/deploy/kubernetes/attacher.yaml)
```

### Step 3: Create the StorageClass
This tells Kubernetes: "When I ask for storage of type `s3-bucket`, use this driver."

```bash
cat <<EOF | kubectl apply -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: s3-bucket
provisioner: ru.yandex.s3.csi
parameters:
  mounter: geesefs
  options: "--memory-limit 1000 --dir-mode 0777 --file-mode 0666"
  bucket: $BUCKET_NAME
EOF
```

---

## 5. Phase 4: Deploying the Application

We will deploy Redis (standard) and Python (with S3 mount).

### Step 1: The Persistent Volume Claim (PVC)
This requests a "slice" of storage from our S3 StorageClass.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: s3-pvc
  namespace: default
spec:
  storageClassName: s3-bucket
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
EOF
```

### Step 2: Deploy Redis
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: my-redis
spec:
  selector:
    app: redis
  ports:
    - port: 6379
EOF
```

### Step 3: Deploy Python App (With S3 Mount)
We mount the S3 bucket to `/data` inside the container.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-web
spec:
  selector:
    matchLabels:
      app: python-web
  template:
    metadata:
      labels:
        app: python-web
    spec:
      containers:
      - name: python-web
        image: python:3.9-alpine
        command: ["/bin/sh"]
        # Script writes to S3 immediately on startup
        args: ["-c", "pip install flask redis && echo 'K8s Connected to S3' > /data/k8s_startup_log.txt && python app.py"]
        volumeMounts:
        - name: s3-storage
          mountPath: /data
        # Inject app code dynamically
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "echo 'import os, redis; from flask import Flask; app = Flask(__name__); cache = redis.Redis(host=\"my-redis\", port=6379); @app.route(\"/\")\ndef hello():\n count = cache.incr(\"hits\");\n with open(\"/data/hits.log\", \"a\") as f: f.write(f\"Hit {count}\\n\");\n return f\"Hello K8s! Hits: {count}. Logs saved to S3.\"' > app.py"]
      volumes:
      - name: s3-storage
        persistentVolumeClaim:
          claimName: s3-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: python-web
spec:
  selector:
    app: python-web
  type: NodePort
  ports:
    - port: 5000
      nodePort: 30001
EOF
```

---

## 6. Phase 5: Verification

### Step 1: Check Pod Status
Wait until all pods are `Running`.
```bash
kubectl get pods
```

### Step 2: Access the App
We will use `curl` to hit the application inside the cluster.
```bash
# Get the internal Minikube IP
export MINIKUBE_IP=$(sudo minikube ip)

# Curl the NodePort
curl http://$MINIKUBE_IP:30001
```
*Result: "Hello K8s! Hits: 1. Logs saved to S3."*

### Step 3: The "Magic" (Verify S3)
The app was configured to append a log line to a file in `/data/hits.log`. Since `/data` is actually your S3 bucket, the file should appear in AWS console.

```bash
# List the bucket contents using AWS CLI
aws s3 ls s3://$BUCKET_NAME
```
**Expected Output:**
You should see `k8s_startup_log.txt` and `hits.log`.

---

## 7. Cleanup

Cleaning up Kubernetes resources and S3 buckets is vital to avoid lingering costs.

```bash
# 1. Delete K8s Resources
kubectl delete deployment python-web redis
kubectl delete pvc s3-pvc
kubectl delete service python-web my-redis

# 2. Stop Minikube
sudo minikube stop
sudo minikube delete

# 3. Empty and Delete Bucket
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3 rb s3://$BUCKET_NAME

# 4. Remove IAM User
aws iam detach-user-policy --user-name k8s-s3-user --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam delete-access-key --user-name k8s-s3-user --access-key-id $AWS_ACCESS_KEY_ID
aws iam delete-user --user-name k8s-s3-user
```
