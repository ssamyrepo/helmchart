## **Step 1: Install Terraform**
Ensure **Terraform** is installed:
```sh
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt update && sudo apt install terraform -y
terraform -version
```
For macOS:
```sh
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

---

## **Step 2: Create Terraform Directory**
```sh
mkdir -p terraform-mongodb && cd terraform-mongodb
```

---

## **Step 3: Create Terraform Configuration Files**
### **1. `providers.tf` (AWS Provider)**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

---

### **2. `variables.tf` (Input Variables)**
```hcl
variable "aws_region" {
  default = "us-east-1"
}

variable "cluster_name" {
  default = "mongodb-cluster"
}

variable "instance_type" {
  default = "t3.medium"
}

variable "node_count" {
  default = 3
}

variable "mongo_password" {
  default = "mysecurepassword"
}
```

---

### **3. `eks.tf` (EKS Cluster Creation)**
```hcl
resource "aws_iam_role" "eks_role" {
  name = "eks-cluster-role"

  assume_role_policy = jsonencode({
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "eks.amazonaws.com"
      }
    }]
    Version = "2012-10-17"
  })
}

resource "aws_eks_cluster" "eks_cluster" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_role.arn

  vpc_config {
    subnet_ids = aws_subnet.public[*].id
  }

  depends_on = [aws_iam_role.eks_role]
}

resource "aws_iam_role" "node_role" {
  name = "eks-node-group-role"

  assume_role_policy = jsonencode({
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
    Version = "2012-10-17"
  })
}

resource "aws_eks_node_group" "eks_nodes" {
  cluster_name    = aws_eks_cluster.eks_cluster.name
  node_group_name = "mongodb-node-group"
  node_role_arn   = aws_iam_role.node_role.arn
  subnet_ids      = aws_subnet.public[*].id
  instance_types  = [var.instance_type]
  scaling_config {
    desired_size = var.node_count
    max_size     = 5
    min_size     = 3
  }

  depends_on = [aws_eks_cluster.eks_cluster]
}
```

---

### **4. `vpc.tf` (Networking Configuration)**
```hcl
resource "aws_vpc" "eks_vpc" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id           = aws_vpc.eks_vpc.id
  cidr_block       = "10.0.${count.index}.0/24"
  availability_zone = element(["us-east-1a", "us-east-1b"], count.index)
  map_public_ip_on_launch = true
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.eks_vpc.id
}
```

---

### **5. `outputs.tf` (Export Terraform Outputs)**
```hcl
output "eks_cluster_name" {
  value = aws_eks_cluster.eks_cluster.name
}

output "eks_cluster_endpoint" {
  value = aws_eks_cluster.eks_cluster.endpoint
}
```

---

## **Step 4: Deploy AWS Infrastructure with Terraform**
### **1. Initialize Terraform**
```sh
terraform init
```

### **2. Apply Terraform Configuration**
```sh
terraform apply -auto-approve
```
This will **provision**:
**VPC, Subnets, and Internet Gateway**  
**EKS Cluster with 3 Nodes**  

---

## **Step 5: Configure `kubectl` for EKS**
Once the cluster is created, update your **kubectl** configuration:
```sh
aws eks update-kubeconfig --region us-east-1 --name mongodb-cluster
kubectl get nodes
```
Expected output:
```
NAME               STATUS   ROLES    AGE   VERSION
ip-10-0-1-123     Ready    <none>   2m    v1.22
ip-10-0-2-234     Ready    <none>   2m    v1.22
ip-10-0-3-345     Ready    <none>   2m    v1.22
```

---

## **Step 6: Run Ansible to Deploy MongoDB**
### **1. Clone the Ansible Playbook**
```sh
git clone https://github.com/your-repo/ansible-mongodb-deploy.git
cd ansible-mongodb-deploy
```

### **2. Run Ansible Playbook**
```sh
ansible-playbook -i inventory/hosts.yml deploy-mongodb.yml
```

---

## **Verification Steps**
#### **Check MongoDB Pods**
```sh
kubectl get pods -n mongodb
```
Expected output:
```
NAME          READY   STATUS    RESTARTS   AGE
mongodb-0     1/1     Running   0          2m
mongodb-1     1/1     Running   0          2m
mongodb-2     1/1     Running   0          2m
```

#### **Check MongoDB Replica Set**
```sh
kubectl exec -it mongodb-0 -n mongodb -- mongosh --eval "rs.status()"
```

---

##  What This Terraform + Ansible Deployment Does**
**Terraform provisions AWS EKS Cluster**  
**Creates networking (VPC, subnets, internet gateway)**  
**Deploys Kubernetes worker nodes**  
**Ansible Playbook installs MongoDB Replica Set on EKS**  
**Configures Persistent Storage (`10Gi` PVC)**  
**Ensures High Availability via Replica Sets**  

---
Here’s a **Terraform Destroy Script** to **safely delete all AWS resources** created for the **MongoDB on EKS deployment**. This script will:
- **Delete the MongoDB StatefulSet and namespace in Kubernetes**
- **Destroy the EKS cluster**
- **Remove networking components (VPC, subnets, etc.)**
- **Ensure no lingering AWS resources remain**

---

## **Step 1: Delete MongoDB from Kubernetes**
Before destroying the infrastructure, remove MongoDB from the **Kubernetes cluster**.

### **Create `delete-mongodb.sh` Script**
```sh
#!/bin/bash

NAMESPACE="mongodb"
HELM_RELEASE="mongodb"

echo "Deleting MongoDB deployment in namespace $NAMESPACE..."
kubectl delete namespace $NAMESPACE --ignore-not-found=true

echo "Uninstalling MongoDB Helm release..."
helm uninstall $HELM_RELEASE -n $NAMESPACE

echo "Waiting for all pods to terminate..."
kubectl wait --for=delete pod --all -n $NAMESPACE --timeout=120s

echo "MongoDB resources deleted successfully!"
```
#### **Make the script executable & run it**
```sh
chmod +x delete-mongodb.sh
./delete-mongodb.sh
```

---

## **Step 2: Destroy AWS EKS Cluster and Networking**
Navigate to your **Terraform directory**:
```sh
cd terraform-mongodb
```

### **Run Terraform Destroy**
```sh
terraform destroy -auto-approve
```
Expected output:
```
Destroy complete! Resources: X destroyed.
```

This command will **delete**:
 **EKS Cluster**  
 **EC2 Nodes (Worker Nodes)**  
 **VPC, Subnets, Internet Gateway**  
 **IAM Roles & Security Groups**  

---

## **Step 3: Verify AWS Resource Cleanup**
After running `terraform destroy`, double-check that all resources are deleted.

### **1. Check if EKS Cluster is Deleted**
```sh
aws eks list-clusters --region us-east-1
```
If the cluster was successfully deleted, the output should be:
```json
{
    "clusters": []
}
```

### **2. Check if EC2 Worker Nodes are Terminated**
```sh
aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,State.Name]" --region us-east-1
```
If the output does not show any running **t3.medium** instances, then all nodes have been terminated.

### **3. Verify VPC Removal**
```sh
aws ec2 describe-vpcs --region us-east-1 --query "Vpcs[*].[VpcId,State]"
```
If your **EKS VPC ID** is **not listed**, it has been removed successfully.

---

## **Step 4: Remove `kubectl` Configurations (Optional)**
If you don’t need the Kubernetes configuration anymore, remove it:
```sh
rm -rf ~/.kube/config
```

---

## **Final Summary**
| **Step** | **Command/Script** | **Purpose** |
|----------|------------------|-------------|
| **Delete MongoDB** | `./delete-mongodb.sh` | Removes MongoDB from Kubernetes |
| **Destroy AWS Resources** | `terraform destroy -auto-approve` | Deletes EKS, nodes, VPC, networking |
| **Check AWS Cleanup** | `aws eks list-clusters` | Verifies EKS deletion |
| **Remove `kubectl` Config** | `rm -rf ~/.kube/config` | Removes K8s access credentials |

---

 **cost optimization guide** for running MongoDB on AWS EKS 

 ### **Cost Optimization Guide for Running MongoDB on AWS EKS**
This guide provides strategies to **reduce costs** while ensuring **performance and availability** for **MongoDB on AWS EKS**.

---

## **1. Choose the Right EC2 Instance Type**
EC2 instances contribute **significantly** to AWS costs. Optimize MongoDB worker nodes by selecting the right **instance type**.

| **Workload Type**         | **Best Instance Type**       | **Notes** |
|--------------------------|----------------------------|-----------|
| **General-purpose**       | `t3.medium`, `m5.large`    | Balanced cost and performance |
| **Read-heavy**           | `r5.large`, `r6g.large`    | Memory-optimized for queries |
| **Write-heavy**          | `i3.large`, `i4i.large`    | NVMe storage for high-speed writes |
| **High availability**     | `m5.xlarge`, `c5.large`    | CPU-optimized, ideal for replicas |

### **Cost-Saving Tips**
 **Use Spot Instances** for secondary nodes (`eksctl create nodegroup --spot`).  
 **Use Graviton2 (ARM-based) instances** (`m6g.large`) for better price-performance.  

---

## **2. Optimize Storage Costs**
EBS storage costs can **escalate** if not managed properly.

### **Storage Strategy for MongoDB**
| **Storage Type**      | **EBS Volume Type** | **Use Case** |
|----------------------|--------------------|--------------|
| **Primary DB Storage** | `gp3 (General Purpose SSD)` | Cost-efficient for production |
| **High-speed Writes**  | `io2 (Provisioned IOPS SSD)` | Low-latency OLTP workloads |
| **Backup Storage**    | `st1 (Throughput HDD)` | Archiving old data |
| **Logs & Monitoring** | `sc1 (Cold HDD)` | Cheapest long-term storage |

### **Cost-Saving Tips**
 **Use `gp3` volumes instead of `gp2`** (50% cheaper with customizable IOPS).  
 **Enable EBS Auto-Scaling** to grow volumes **dynamically**.  
 **Move infrequently used data** to **Amazon S3 or S3 Glacier** instead of keeping it in MongoDB.  

---

## **3. Use Kubernetes Autoscaling**
### **Enable Cluster Autoscaler**
Automatically **scale down nodes when traffic is low**:
```sh
eksctl create cluster --name mongodb-cluster --region us-east-1 --nodes-min=1 --nodes-max=5
```
**Benefit:** You only pay for **active nodes**.

### **Use Horizontal Pod Autoscaler (HPA)**
Automatically **scale MongoDB pods** based on CPU & memory usage.
```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: mongodb-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: mongodb
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```
✅ **Benefit:** Reduces the number of running pods during low traffic.

---

## **4. Optimize MongoDB Replica Set Costs**
**MongoDB clusters use 3 or more replicas**, but you can optimize costs by:
- **Using Spot Instances for secondary replicas**.
- **Storing read-heavy workloads in secondary replicas** instead of running high-cost primary queries.
- **Adjusting Write Concern (`w: 1`) and Read Concern (`majority`)** to avoid unnecessary replication costs.

---

## **5. Optimize Networking & Data Transfer Costs**
Cross-region and excessive data transfer can **increase AWS bills**.

### **Reduce Networking Costs**
| **Optimization Strategy** | **Command/Method** | **Benefit** |
|--------------------------|------------------|-------------|
| **Use same region for EKS & MongoDB** | Deploy in `us-east-1` | Avoid inter-region traffic fees |
| **Enable VPC Peering** | `aws ec2 create-vpc-peering-connection` | Reduces data transfer costs between services |
| **Use Private Endpoints** | `aws eks update-cluster-config --resources-vpc-config endpointPrivateAccess=true` | Avoids expensive public internet traffic |

✅ **Benefit:** Private networking reduces inter-AZ data transfer costs.

---

## **6. Use AWS Savings Plans & Reserved Instances**
| **Option** | **Savings** | **Best Use Case** |
|------------|------------|------------------|
| **Compute Savings Plan** | **Up to 66% off** | Long-running EKS worker nodes |
| **Reserved EC2 Instances** | **Up to 72% off** | Dedicated MongoDB instances |
| **Spot Instances** | **Up to 90% off** | Secondary replicas, backups |

✅ **Tip:** If MongoDB runs **24/7**, **Reserved Instances** offer the **best cost reduction**.

---

## **7. Automate Backups & Reduce Backup Costs**
- **Enable Snapshot Lifecycle Policy** to store backups efficiently:
  ```sh
  aws ec2 create-snapshot --volume-id vol-12345678
  ```
- **Use S3 Intelligent-Tiering** to reduce **long-term storage costs**.

✅ **Tip:** Store **older snapshots in Amazon Glacier** to **save up to 80%** on storage.

---

## **8. Monitor and Optimize Costs with AWS Cost Explorer**
Use **AWS Cost Explorer** to analyze spending patterns:
```sh
aws ce get-cost-and-usage --time-period Start=2024-01-01,End=2024-01-31 --granularity MONTHLY --metrics "BlendedCost"
```
✅ **Tip:** Set up **AWS Budgets Alerts** to avoid unexpected charges.

---

## ** Cost Optimization Checklist**
 **Use Spot Instances for secondary nodes**  
 **Optimize EBS volumes (`gp3`, Auto-Scaling, Archive old data)**  
 **Enable Cluster & Pod Autoscaling (HPA)**  
 **Use Private Networking (VPC Peering, Endpoints)**  
 **Enable AWS Savings Plans or Reserved Instances for EKS nodes**  
 **Automate Backups with S3 Glacier for older snapshots**  
 **Monitor costs using AWS Cost Explorer & Budgets**  


Would you like a **Terraform script for AWS Cost Monitoring Alerts**? 
