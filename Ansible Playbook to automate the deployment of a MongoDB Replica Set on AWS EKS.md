**Ansible Playbook** to **automate the deployment of a MongoDB Replica Set on AWS EKS**. 
This playbook:
- **Installs all prerequisites** (AWS CLI, eksctl, kubectl, Helm)
- **Creates an AWS EKS cluster**
- **Deploys MongoDB using Helm**
- **Initializes the MongoDB Replica Set**
- **Verifies the setup**

---

### **Step 1: Install Ansible**
Ensure **Ansible is installed** on your control machine.
```sh
sudo apt update && sudo apt install -y ansible
```
For macOS:
```sh
brew install ansible
```

---

### **Step 2: Create Ansible Playbook Structure**
```sh
mkdir -p ansible-mongodb-deploy/{roles/mongodb/tasks,roles/mongodb/templates,roles/prereqs/tasks,inventory}
cd ansible-mongodb-deploy
```

---

### **Step 3: Define Inventory (`inventory/hosts.yml`)**
Replace **your AWS region** if needed.
```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
      aws_region: "us-east-1"
      cluster_name: "mongodb-cluster"
      namespace: "mongodb"
      instance_type: "t3.medium"
      node_count: 3
      mongo_password: "mysecurepassword"
```

---

### **Step 4: Install Prerequisites (`roles/prereqs/tasks/main.yml`)**
This task installs **AWS CLI, eksctl, kubectl, and Helm**.
```yaml
- name: Install AWS CLI
  shell: |
    curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
    sudo installer -pkg AWSCLIV2.pkg -target /
  args:
    creates: /usr/local/bin/aws

- name: Install kubectl
  shell: |
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    chmod +x kubectl
    sudo mv kubectl /usr/local/bin/
  args:
    creates: /usr/local/bin/kubectl

- name: Install eksctl
  shell: |
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv /tmp/eksctl /usr/local/bin
  args:
    creates: /usr/local/bin/eksctl

- name: Install Helm
  shell: |
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
  args:
    creates: /usr/local/bin/helm
```

---

### **Step 5: Create EKS Cluster (`roles/mongodb/tasks/main.yml`)**
This task **creates the EKS cluster** and **deploys MongoDB**.
```yaml
- name: Create EKS Cluster
  shell: >
    eksctl create cluster --name {{ cluster_name }}
    --region {{ aws_region }}
    --nodes {{ node_count }}
    --node-type {{ instance_type }}
  args:
    creates: /root/.eksctl

- name: Update kubeconfig
  shell: >
    aws eks update-kubeconfig --region {{ aws_region }} --name {{ cluster_name }}

- name: Create MongoDB Namespace
  shell: kubectl create namespace {{ namespace }}

- name: Add Helm Repo
  shell: |
    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo update

- name: Deploy MongoDB using Helm
  shell: >
    helm install mongodb bitnami/mongodb
    --namespace {{ namespace }}
    --set architecture=replicaset
    --set replicaCount=3
    --set persistence.size=10Gi
    --set auth.rootPassword={{ mongo_password }}
    --set auth.username=admin
    --set auth.password={{ mongo_password }}
    --set auth.database=mydatabase
```

---

### **Step 6: Initialize MongoDB Replica Set (`roles/mongodb/tasks/init_replica.yml`)**
```yaml
- name: Wait for MongoDB Pods to be Ready
  shell: kubectl wait --for=condition=Ready pods --all -n {{ namespace }} --timeout=300s

- name: Get MongoDB Pod Name
  shell: kubectl get pods -n {{ namespace }} -l app.kubernetes.io/name=mongodb -o jsonpath="{.items[0].metadata.name}"
  register: mongo_pod

- name: Initialize MongoDB Replica Set
  shell: >
    kubectl exec -it {{ mongo_pod.stdout }} -n {{ namespace }} -- mongosh --eval
    "rs.initiate({
      _id: 'rs0',
      members: [
        { _id: 0, host: 'mongodb-0.mongodb.mongodb.svc.cluster.local:27017' },
        { _id: 1, host: 'mongodb-1.mongodb.mongodb.svc.cluster.local:27017' },
        { _id: 2, host: 'mongodb-2.mongodb.mongodb.svc.cluster.local:27017' }
      ]
    });"

- name: Check Replica Set Status
  shell: >
    kubectl exec -it {{ mongo_pod.stdout }} -n {{ namespace }} -- mongosh --eval "rs.status()"
```

---

### **Step 7: Create Main Playbook (`deploy-mongodb.yml`)**
```yaml
- hosts: all
  gather_facts: no
  roles:
    - prereqs
    - mongodb
```

---

### **Step 8: Run the Playbook**
#### **1. Make the script executable**
```sh
chmod +x deploy-mongodb.yml
```

#### **2. Run the Ansible Playbook**
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

## Notes: What This Ansible Playbook Does**
 **Installs all prerequisites (AWS CLI, eksctl, kubectl, Helm).**  
 **Creates an AWS EKS Cluster with 3 nodes.**  
 **Deploys MongoDB Replica Set with Helm.**  
 **Configures Persistent Storage (`10Gi` PVC).**  
 **Initializes MongoDB Replica Set (`rs.initiate()`).**  
 **Verifies the deployment.**  

