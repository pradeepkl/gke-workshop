# GKE Cluster Setup Guide - Multi-AZ with CLI and Terraform

This guide provides step-by-step instructions for creating a Google Kubernetes Engine (GKE) cluster with worker nodes distributed across three availability zones in South India (asia-south1) region.

## Cluster Specifications
- **Region**: asia-south1 (Mumbai)
- **Zones**: asia-south1-a, asia-south1-b, asia-south1-c
- **Cluster Name**: gke-training
- **Node Type**: e2-standard-2 (2 vCPU, 8GB RAM)
- **Node Count**: 4 nodes
- **Node Pool Name**: primary-pool

## Option 1: Using gcloud CLI

### Prerequisites Setup
```bash
# Install gcloud CLI (if not already installed)
gcloud init
gcloud auth login
gcloud config set project trainer-gketraining21
```

### Enable Required APIs
```bash
gcloud services enable container.googleapis.com compute.googleapis.com
```

### Create GKE Cluster
```bash
gcloud container clusters create gke-training --project=trainer-gketraining21 --region=asia-south1 --node-locations=asia-south1-a,asia-south1-b,asia-south1-c --num-nodes=2 --machine-type=e2-standard-2 --disk-size=50 --network=default --no-enable-ip-alias --cluster-version=latest --enable-network-policy --enable-master-authorized-networks --master-authorized-networks=0.0.0.0/0 --addons=HorizontalPodAutoscaling,HttpLoadBalancing
```

Note: We're creating 2 nodes per zone (total 6 nodes) to maintain high availability.

### Verify Cluster Creation
```bash
# List clusters
gcloud container clusters list

# Get cluster details
gcloud container clusters describe gke-training --region=asia-south1

# Configure kubectl
gcloud container clusters get-credentials gke-training --region=asia-south1 --project=trainer-gketraining21
```

### Verify Nodes
```bash
kubectl get nodes -o wide
```

### CLI Teardown Process
```bash
# Delete the cluster
gcloud container clusters delete gke-training --region=asia-south1 --project=trainer-gketraining21 --quiet
```

## Option 2: Using Terraform

### Create the following files:

1. `provider.tf`:
```hcl
provider "google" {
  project = "trainer-gketraining21"
  region  = "asia-south1"
}

terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 4.0"
    }
  }
}
```

2. `variables.tf`:
```hcl
variable "project_id" {
  description = "The project ID"
  default     = "trainer-gketraining21"
}

variable "region" {
  description = "The region for the cluster"
  default     = "asia-south1"
}

variable "zones" {
  description = "The zones for the cluster nodes"
  type        = list(string)
  default     = ["asia-south1-a", "asia-south1-b", "asia-south1-c"]
}

variable "cluster_name" {
  description = "The name of the cluster"
  default     = "gke-training"
}

variable "machine_type" {
  description = "The machine type for nodes"
  default     = "e2-standard-2"
}
```

3. `main.tf`:
```hcl
resource "google_container_cluster" "primary" {
  name               = var.cluster_name
  location          = var.region
  node_locations    = var.zones
  
  # Remove default node pool
  remove_default_node_pool = true
  initial_node_count       = 1

  network_policy {
    enabled = true
  }

  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "0.0.0.0/0"
      display_name = "All"
    }
  }
}

resource "google_container_node_pool" "primary_nodes" {
  name       = "primary-pool"
  location   = var.region
  cluster    = google_container_cluster.primary.name
  
  node_count = 2  # 2 nodes per zone = 6 total

  node_config {
    machine_type = var.machine_type
    disk_size_gb = 50

    oauth_scopes = [
      "https://www.googleapis.com/auth/devstorage.read_only",
      "https://www.googleapis.com/auth/logging.write",
      "https://www.googleapis.com/auth/monitoring",
      "https://www.googleapis.com/auth/servicecontrol",
      "https://www.googleapis.com/auth/service.management.readonly",
      "https://www.googleapis.com/auth/trace.append"
    ]
  }
}
```

4. `outputs.tf`:
```hcl
output "cluster_endpoint" {
  description = "Cluster endpoint"
  value       = google_container_cluster.primary.endpoint
}

output "cluster_ca_certificate" {
  description = "Cluster CA certificate"
  value       = google_container_cluster.primary.master_auth[0].cluster_ca_certificate
  sensitive   = true
}
```

### Terraform Commands

Initialize Terraform:
```bash
terraform init
```

Plan the deployment:
```bash
terraform plan
```

Apply the configuration:
```bash
terraform apply -auto-approve
```

Configure kubectl:
```bash
gcloud container clusters get-credentials gke-training --region=asia-south1 --project=trainer-gketraining21
```

### Terraform Teardown Process
```bash
terraform destroy -auto-approve
```

## Verification Steps (For Both Methods)

1. Check Cluster Status:
```bash
kubectl cluster-info
```

2. Verify Nodes:
```bash
kubectl get nodes -o wide
```

3. Check Node Distribution:
```bash
kubectl get nodes -L topology.kubernetes.io/zone
```

4. Test Basic Deployment:
```bash
kubectl create deployment nginx --image=nginx:latest
kubectl expose deployment nginx --port=80 --type=LoadBalancer
kubectl get services
```

## Common Issues and Troubleshooting

1. **API Enablement Issues**
   ```bash
   gcloud services enable container.googleapis.com
   gcloud services enable compute.googleapis.com
   ```

2. **Permission Issues**
   ```bash
   # Verify your permissions
   gcloud auth list
   gcloud projects get-iam-policy trainer-gketraining21
   ```

3. **Quota Issues**
   - Check quotas in GCP Console
   - Request quota increase if needed

## Best Practices

1. **Security**
   - Enable network policies
   - Configure master authorized networks
   - Regular security updates

2. **High Availability**
   - Spread nodes across zones
   - Use regional clusters
   - Proper resource requests/limits

3. **Monitoring**
   - Enable monitoring
   - Set up alerts
   - Regular health checks

## Next Steps

1. Set up monitoring and logging
2. Configure cluster autoscaling
3. Implement security policies
4. Set up CI/CD pipelines