
### **Terraform Structure**
1. **Create an EKS Cluster**
2. **Deploy the EBS CSI Driver**
3. **Provision an EBS Volume**
4. **Deploy MongoDB using Kubernetes manifests via Terraform**

---

### **1️⃣ Create a Terraform Configuration File**
Let's create a **Terraform script** that deploys the entire setup.

---

#### **`main.tf`** - Provision EKS and Deploy MongoDB
```hcl
provider "aws" {
  region = "us-east-1"
}

provider "kubernetes" {
  host                   = aws_eks_cluster.eks.endpoint
  token                  = data.aws_eks_cluster_auth.cluster.token
  cluster_ca_certificate = base64decode(aws_eks_cluster.eks.certificate_authority[0].data)
}

provider "helm" {
  kubernetes {
    host                   = aws_eks_cluster.eks.endpoint
    token                  = data.aws_eks_cluster_auth.cluster.token
    cluster_ca_certificate = base64decode(aws_eks_cluster.eks.certificate_authority[0].data)
  }
}

# --- IAM Role for EKS ---
resource "aws_iam_role" "eks_cluster_role" {
  name = "eks-cluster-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster_role.name
}

# --- Create EKS Cluster ---
resource "aws_eks_cluster" "eks" {
  name     = "my-cluster"
  role_arn = aws_iam_role.eks_cluster_role.arn
  version  = "1.25"

  vpc_config {
    subnet_ids = [aws_subnet.private_1.id, aws_subnet.private_2.id]
  }

  depends_on = [aws_iam_role_policy_attachment.eks_cluster_policy]
}

data "aws_eks_cluster_auth" "cluster" {
  name = aws_eks_cluster.eks.name
}

# --- IAM Role for Worker Nodes ---
resource "aws_iam_role" "eks_node_role" {
  name = "eks-node-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "eks_worker_node_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_node_role.name
}

# --- Create Node Group ---
resource "aws_eks_node_group" "eks_nodes" {
  cluster_name    = aws_eks_cluster.eks.name
  node_group_name = "worker-nodes"
  node_role_arn   = aws_iam_role.eks_node_role.arn
  subnet_ids      = [aws_subnet.private_1.id, aws_subnet.private_2.id]
  
  scaling_config {
    desired_size = 2
    max_size     = 3
    min_size     = 1
  }
}

# --- Create an EBS Volume for MongoDB ---
resource "aws_ebs_volume" "mongo_ebs" {
  availability_zone = "us-east-1a"
  size              = 10
  type              = "gp2"
  tags = {
    Name = "MongoDB-EBS"
  }
}

# --- Deploy EBS CSI Driver (via Helm) ---
resource "helm_release" "ebs_csi_driver" {
  name       = "aws-ebs-csi-driver"
  repository = "https://kubernetes-sigs.github.io/aws-ebs-csi-driver"
  chart      = "aws-ebs-csi-driver"
  namespace  = "kube-system"
}

# --- Kubernetes Storage Class for EBS ---
resource "kubernetes_storage_class" "mongo_storage_class" {
  metadata {
    name = "mongo-sc"
  }

  provisioner = "ebs.csi.aws.com"

  parameters = {
    type = "gp2"
  }
}

# --- Kubernetes Persistent Volume Claim for MongoDB ---
resource "kubernetes_persistent_volume_claim" "mongo_pvc" {
  metadata {
    name = "mongo-pvc"
  }

  spec {
    access_modes = ["ReadWriteOnce"]
    
    resources {
      requests = {
        storage = "10Gi"
      }
    }

    storage_class_name = kubernetes_storage_class.mongo_storage_class.metadata[0].name
  }
}

# --- Deploy MongoDB Deployment ---
resource "kubernetes_deployment" "mongo" {
  metadata {
    name = "mongo"
    labels = {
      app = "mongo"
    }
  }

  spec {
    replicas = 1

    selector {
      match_labels = {
        app = "mongo"
      }
    }

    template {
      metadata {
        labels = {
          app = "mongo"
        }
      }

      spec {
        container {
          name  = "mongo"
          image = "mongo:latest"

          port {
            container_port = 27017
          }

          env {
            name  = "MONGO_INITDB_ROOT_USERNAME"
            value = "admin"
          }

          env {
            name  = "MONGO_INITDB_ROOT_PASSWORD"
            value = "password123"
          }

          volume_mount {
            mount_path = "/data/db"
            name       = "mongo-storage"
          }
        }

        volume {
          name = "mongo-storage"

          persistent_volume_claim {
            claim_name = kubernetes_persistent_volume_claim.mongo_pvc.metadata[0].name
          }
        }
      }
    }
  }
}

# --- Kubernetes Service for MongoDB ---
resource "kubernetes_service" "mongo_service" {
  metadata {
    name = "mongo-service"
  }

  spec {
    selector = {
      app = "mongo"
    }

    port {
      protocol    = "TCP"
      port        = 27017
      target_port = 27017
    }

    type = "ClusterIP"
  }
}
```

---

### **2️⃣ Initialize and Apply Terraform**
Run the following commands to deploy your infrastructure:

```sh
# Initialize Terraform
terraform init

# Plan deployment
terraform plan

# Apply changes
terraform apply -auto-approve
```

---

### **3️⃣ Verify the Setup**
Once the deployment is complete, verify the MongoDB setup in Kubernetes.

1. **Check MongoDB Pods**
   ```sh
   kubectl get pods -l app=mongo
   ```

2. **Check MongoDB Logs**
   ```sh
   kubectl logs -f deployment/mongo
   ```

3. **Connect to MongoDB**
   ```sh
   kubectl exec -it $(kubectl get pods -l app=mongo -o jsonpath="{.items[0].metadata.name}") -- mongo -u admin -p password123 --authenticationDatabase admin
   ```

---

### **4️⃣ Clean Up Resources**
To delete everything:
```sh
terraform destroy -auto-approve
```

---

