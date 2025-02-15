Here is a **fully automated script** with **exact step-by-step instructions** to deploy a **MongoDB Replica Set on AWS EKS**. This includes **installing all prerequisites**, setting up **AWS CLI, eksctl, kubectl, and Helm**, and **deploying MongoDB**.

---

## **Step 1: Install Prerequisites**
Before running the script, ensure you have the following installed on your local machine:

### **1. Install AWS CLI**
The AWS CLI is required for managing AWS services.
```sh
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version
```
Configure AWS credentials:
```sh
aws configure
```
Enter your:
- AWS **Access Key**
- AWS **Secret Key**
- **Region** (e.g., `us-east-1`)
- Output format (`json` recommended)

---

### **2. Install kubectl (Kubernetes CLI)**
`kubectl` is needed to manage your Kubernetes cluster.
```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

---

### **3. Install eksctl (EKS Management Tool)**
`eksctl` is required for managing AWS EKS clusters.
```sh
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

---

### **4. Install Helm (Kubernetes Package Manager)**
```sh
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

### **5. Verify All Installations**
Run the following command to verify all tools are installed:
```sh
aws --version
kubectl version --client
eksctl version
helm version
```

---

## **Step 2: Deploy MongoDB on AWS EKS**
Save the following **fully automated script** as `deploy-mongodb-eks.sh`.

```sh
#!/bin/bash

# Set Variables
CLUSTER_NAME="mongodb-cluster"
NAMESPACE="mongodb"
HELM_CHART_NAME="mongodb"
REGION="us-east-1"
INSTANCE_TYPE="t3.medium"
NODE_COUNT=3
MONGO_PASSWORD="mysecurepassword"

# Create an EKS cluster
echo "Creating EKS Cluster..."
eksctl create cluster --name $CLUSTER_NAME --region $REGION --nodes $NODE_COUNT --node-type $INSTANCE_TYPE

# Update kubeconfig
echo "Updating kubeconfig..."
aws eks update-kubeconfig --region $REGION --name $CLUSTER_NAME

# Verify cluster connection
kubectl get nodes

# Create namespace for MongoDB
echo "Creating MongoDB namespace..."
kubectl create namespace $NAMESPACE

# Install Helm if not installed
if ! command -v helm &> /dev/null; then
    echo "Helm not found. Installing..."
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
fi

# Add Bitnami Helm repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Deploy MongoDB using Helm
echo "Deploying MongoDB..."
helm install $HELM_CHART_NAME bitnami/mongodb --namespace $NAMESPACE \
  --set architecture=replicaset \
  --set replicaCount=3 \
  --set persistence.size=10Gi \
  --set auth.rootPassword=$MONGO_PASSWORD \
  --set auth.username=admin \
  --set auth.password=$MONGO_PASSWORD \
  --set auth.database=mydatabase

# Wait for MongoDB pods to be ready
echo "Waiting for MongoDB pods to be ready..."
kubectl wait --for=condition=Ready pods --all -n $NAMESPACE --timeout=300s

# Get MongoDB pod name
MONGO_POD=$(kubectl get pods -n $NAMESPACE -l app.kubernetes.io/name=mongodb -o jsonpath="{.items[0].metadata.name}")

# Initialize MongoDB Replica Set
echo "Initializing MongoDB Replica Set..."
kubectl exec -it $MONGO_POD -n $NAMESPACE -- mongosh --eval "
rs.initiate({
  _id: 'rs0',
  members: [
    { _id: 0, host: 'mongodb-0.mongodb.mongodb.svc.cluster.local:27017' },
    { _id: 1, host: 'mongodb-1.mongodb.mongodb.svc.cluster.local:27017' },
    { _id: 2, host: 'mongodb-2.mongodb.mongodb.svc.cluster.local:27017' }
  ]
});
"

# Check replica set status
kubectl exec -it $MONGO_POD -n $NAMESPACE -- mongosh --eval "rs.status()"

echo "MongoDB Replica Set deployed successfully!"
```

---

### **Step 3: Run the Script**
Make the script executable:
```sh
chmod +x deploy-mongodb-eks.sh
```

Run the script:
```sh
./deploy-mongodb-eks.sh
```

---

## **Step 4: Verification**
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

#### **Check MongoDB Logs**
```sh
kubectl logs -l app.kubernetes.io/name=mongodb -n mongodb
```

#### **Access MongoDB Shell**
```sh
kubectl exec -it mongodb-0 -n mongodb -- mongosh
```
Inside the MongoDB shell, check the replica set:
```js
rs.status()
```

---

##  What This Script Does**
 **Installs all necessary tools (AWS CLI, kubectl, eksctl, Helm).**  
 **Creates an AWS EKS Cluster with 3 nodes.**  
 **Deploys MongoDB Replica Set with Helm.**  
 **Configures Persistent Storage (`10Gi` PVC).**  
 **Initializes MongoDB Replica Set (`rs.initiate()`).**  
 **Verifies the deployment.**  

