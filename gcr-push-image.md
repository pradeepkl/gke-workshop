# GCP Authentication and Container Registry Guide

## 1. GCP Authentication

### Option 1: User Account Login
```bash
# Interactive login
gcloud auth login

# Configure docker with gcloud credentials
gcloud auth configure-docker asia-south1-docker.pkg.dev
```

### Option 2: Service Account Login
```bash
# Using service account key file
gcloud auth activate-service-account --key-file=/path/to/service-account.json

# Configure docker with service account
gcloud auth configure-docker asia-south1-docker.pkg.dev --quiet
```

## 2. Configure Project
```bash
# Set default project
gcloud config set project trainer-gketraining21

# Verify project
gcloud config get-value project
```

## 3. Docker Commands

### Build Image
```bash
# Basic build
docker build -t order-microservice .

# Build with specific tag
docker build -t order-microservice:1.0.0 .

# Build with no cache
docker build --no-cache -t order-microservice .
```

### Tag Image for GCP Artifact Registry
```bash
# Format: REGION-docker.pkg.dev/PROJECT_ID/REPOSITORY/IMAGE
docker tag order-microservice:1.0.0 asia-south1-docker.pkg.dev/trainer-gketraining21/order-microservice-repo/order-microservice:1.0.0

# Tag as latest
docker tag order-microservice:1.0.0 asia-south1-docker.pkg.dev/trainer-gketraining21/order-microservice-repo/order-microservice:latest
```

### Push Images to Artifact Registry
```bash
# Push specific version
docker push asia-south1-docker.pkg.dev/trainer-gketraining21/order-microservice-repo/order-microservice:1.0.0

# Push latest tag
docker push asia-south1-docker.pkg.dev/trainer-gketraining21/order-microservice-repo/order-microservice:latest
```

## 4. Verify Uploads

```bash
# List images in repository
gcloud artifacts docker images list asia-south1-docker.pkg.dev/trainer-gketraining21/order-microservice-repo

# Describe specific image
gcloud artifacts docker images describe asia-south1-docker.pkg.dev/trainer-gketraining21/order-microservice-repo/order-microservice:1.0.0
```

## 5. Additional Useful Commands

### Clean Up Local Images
```bash
# Remove local image
docker rmi asia-south1-docker.pkg.dev/trainer-gketraining21/order-microservice-repo/order-microservice:1.0.0

# Remove all unused images
docker image prune -a
```

### Pull Image from Artifact Registry
```bash
# Pull specific version
docker pull asia-south1-docker.pkg.dev/trainer-gketraining21/order-microservice-repo/order-microservice:1.0.0

# Pull latest
docker pull asia-south1-docker.pkg.dev/trainer-gketraining21/order-microservice-repo/order-microservice:latest
```

### List and Revoke Credentials
```bash
# List active account
gcloud auth list

# Revoke credentials
gcloud auth revoke

# Clear docker credentials
docker logout asia-south1-docker.pkg.dev
```

Remember to replace:
- `order-microservice` with your actual image name
- `1.0.0` with your desired version number
- Make sure you have the necessary permissions in GCP

Would you like me to add any specific commands or explain any part in more detail?