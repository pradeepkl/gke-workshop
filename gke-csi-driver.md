# GKE Storage Integration with Workload Identity and CSI Drivers

This guide demonstrates how to set up persistent storage in Google Kubernetes Engine (GKE) using CSI drivers and Workload Identity. We'll use MongoDB with Persistent Disk (similar to AWS EBS) and Nginx with Filestore (similar to AWS EFS) as practical examples.

## Architecture Overview
 

## Prerequisites

- GCP Project (trainer-gketraining21)
- GKE Cluster with Workload Identity enabled
- `gcloud` CLI installed and configured
- `kubectl` configured to connect to your cluster

## Step 1: Enable Required GCP APIs

```bash
# Enable required APIs
gcloud services enable container.googleapis.com --project=trainer-gketraining21
gcloud services enable iam.googleapis.com --project=trainer-gketraining21
gcloud services enable storage-api.googleapis.com --project=trainer-gketraining21
gcloud services enable file.googleapis.com --project=trainer-gketraining21
```

## Step 2: Enable CSI Drivers in GKE Cluster

```bash
# Enable Persistent Disk CSI Driver
gcloud container clusters update YOUR_CLUSTER_NAME \
    --project=trainer-gketraining21 \
    --location=asia-south1-a \
    --update-addons=GcePersistentDiskCsiDriver=ENABLED

# Enable Filestore CSI Driver
gcloud container clusters update YOUR_CLUSTER_NAME \
    --project=trainer-gketraining21 \
    --location=asia-south1-a \
    --update-addons=FilestoreCsiDriver=ENABLED
```

## Step 3: Configure Service Accounts and Workload Identity

```bash
# Create GCP Service Account
gcloud iam service-accounts create storage-sa \
    --project=trainer-gketraining21 \
    --display-name="Storage Service Account"

# Grant Storage Admin Role
gcloud projects add-iam-policy-binding trainer-gketraining21 \
    --member="serviceAccount:storage-sa@trainer-gketraining21.iam.gserviceaccount.com" \
    --role="roles/storage.admin"

# Allow K8s Service Account to use GCP Service Account
gcloud iam service-accounts add-iam-policy-binding storage-sa@trainer-gketraining21.iam.gserviceaccount.com \
    --role="roles/iam.workloadIdentityUser" \
    --member="serviceAccount:trainer-gketraining21.svc.id.goog[default/storage-sa]"
```

## Step 4: Create Kubernetes Resources

### 4.1 Create Storage RBAC (storage-rbac.yaml)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: storage-sa
  annotations:
    iam.gke.io/gcp-service-account: storage-sa@trainer-gketraining21.iam.gserviceaccount.com

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: storage-role
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: storage-role-binding
subjects:
  - kind: ServiceAccount
    name: storage-sa
    namespace: default
roleRef:
  kind: ClusterRole
  name: storage-role
  apiGroup: rbac.authorization.k8s.io
```

Apply the RBAC configuration:
```bash
kubectl apply -f storage-rbac.yaml
```

### 4.2 Create MongoDB Configuration (mongodb.yaml)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard-rwo
  resources:
    requests:
      storage: 20Gi

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      serviceAccountName: storage-sa
      containers:
      - name: mongodb
        image: mongo:latest
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
      volumes:
      - name: mongodb-data
        persistentVolumeClaim:
          claimName: mongodb-data
```

Deploy MongoDB:
```bash
kubectl apply -f mongodb.yaml
```

### 4.3 Create Nginx Configuration (nginx.yaml)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: filestore-sc
provisioner: filestore.csi.storage.gke.io
parameters:
  tier: "STANDARD"
  network: "default"
volumeBindingMode: Immediate

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-logs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: filestore-sc
  resources:
    requests:
      storage: 1Ti

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: storage-sa
      containers:
      - name: nginx
        image: nginx:latest
        volumeMounts:
        - name: nginx-logs
          mountPath: /var/log/nginx
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
      volumes:
      - name: nginx-logs
        persistentVolumeClaim:
          claimName: nginx-logs
```

Deploy Nginx:
```bash
kubectl apply -f nginx.yaml
```

## Step 5: Verify Setup

### 1. Check Service Account Configuration
```bash
# Verify K8s Service Account
kubectl describe serviceaccount storage-sa

# Check if pods are using the correct service account
kubectl get pods -o jsonpath='{.items[*].spec.serviceAccountName}'
```

### 2. Verify Storage Resources
```bash
# Check PVCs
kubectl get pvc
kubectl describe pvc mongodb-data
kubectl describe pvc nginx-logs

# Check Storage Classes
kubectl get storageclass
```

### 3. Test MongoDB Persistence
```bash
# Connect to MongoDB
kubectl exec -it mongodb-0 -- mongosh

# Create test data
use test
db.items.insertOne({"message": "Hello, Persistent Storage!"})

# Exit and delete pod to verify persistence
kubectl delete pod mongodb-0

# After pod recreates, check data
kubectl exec -it mongodb-0 -- mongosh
use test
db.items.find()
```

### 4. Test Nginx Logs Persistence
```bash
# Generate some traffic
kubectl exec -it $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- curl localhost

# Check logs in shared volume
kubectl exec -it $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- cat /var/log/nginx/access.log
```

## Step 6: Cleanup

```bash
# Delete Applications
kubectl delete -f nginx.yaml
kubectl delete -f mongodb.yaml
kubectl delete -f storage-rbac.yaml

# Delete GCP Service Account
gcloud iam service-accounts delete storage-sa@trainer-gketraining21.iam.gserviceaccount.com --project=trainer-gketraining21
```

## Troubleshooting Guide

### 1. Volume Mounting Issues
```bash
# Check PVC status
kubectl describe pvc mongodb-data
kubectl describe pvc nginx-logs

# Verify StorageClass
kubectl get storageclass
kubectl describe storageclass standard-rwo
kubectl describe storageclass filestore-sc

# Check CSI driver pods
kubectl get pods -n kube-system | grep csi
```

### 2. Permission Issues
```bash
# Verify K8s Service Account
kubectl describe serviceaccount storage-sa

# Check pod events
kubectl describe pod mongodb-0
kubectl describe pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}')

# Verify GCP Service Account permissions
gcloud iam service-accounts get-iam-policy storage-sa@trainer-gketraining21.iam.gserviceaccount.com
```

### 3. Common Problems and Solutions

1. **PVC Stuck in Pending**
   - Check storage class exists
   - Verify CSI driver is running
   - Check if Filestore API is enabled for Filestore PVCs

2. **Pod Can't Access Volume**
   - Verify service account annotation
   - Check Workload Identity binding
   - Ensure pod is using correct service account

3. **Filestore Issues**
   - Remember 1Ti minimum size requirement
   - Check network configuration
   - Verify Filestore API is enabled

## Best Practices

1. **Storage Management**
   - Use Persistent Disk for databases (ReadWriteOnce)
   - Use Filestore for shared storage (ReadWriteMany)
   - Implement regular backup strategies

2. **Security**
   - Follow principle of least privilege for service accounts
   - Regular audit of IAM bindings
   - Monitor storage access patterns

3. **Performance**
   - Choose appropriate storage class for workload
   - Set proper resource limits
   - Monitor storage usage and performance

