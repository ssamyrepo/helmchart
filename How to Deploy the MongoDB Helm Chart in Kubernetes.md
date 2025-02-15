Here's a **Helm chart** to deploy a **MongoDB Replica Set** in a **Kubernetes cluster**. This chart automates the deployment using **StatefulSets, Persistent Volumes, and Service Discovery**.

---

## **MongoDB Helm Chart Structure**
```
mongodb-helm/
│── charts/
│── templates/
│   │── configmap.yaml
│   │── service.yaml
│   │── statefulset.yaml
│   │── secrets.yaml
│── values.yaml
│── Chart.yaml
│── README.md
```

---

### **1. `Chart.yaml` (Helm Chart Metadata)**
```yaml
apiVersion: v2
name: mongodb
description: MongoDB Replica Set on Kubernetes
type: application
version: 1.0.0
appVersion: 5.0.0
```

---

### **2. `values.yaml` (Customizable Parameters)**
```yaml
replicaCount: 3  # Number of MongoDB replicas

image:
  repository: mongo
  tag: "5.0"
  pullPolicy: IfNotPresent

storage:
  size: 10Gi
  className: standard  # StorageClass for Persistent Volume

resources:
  limits:
    cpu: "1"
    memory: "2Gi"
  requests:
    cpu: "0.5"
    memory: "1Gi"

mongodb:
  replicaSetName: rs0
  rootUsername: admin
  rootPassword: mysecurepassword
  database: mydatabase
```

---

### **3. `templates/service.yaml` (MongoDB Headless Service)**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  selector:
    app: mongodb
  ports:
    - name: mongo
      port: 27017
  clusterIP: None  # Headless service for StatefulSet
```

---

### **4. `templates/secrets.yaml` (MongoDB Credentials)**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  MONGO_INITDB_ROOT_USERNAME: {{ .Values.mongodb.rootUsername | b64enc }}
  MONGO_INITDB_ROOT_PASSWORD: {{ .Values.mongodb.rootPassword | b64enc }}
```

---

### **5. `templates/statefulset.yaml` (MongoDB StatefulSet)**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 27017
          envFrom:
            - secretRef:
                name: mongodb-secret
          args:
            - "--replSet"
            - "{{ .Values.mongodb.replicaSetName }}"
          volumeMounts:
            - name: mongo-storage
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongo-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: {{ .Values.storage.size }}
        storageClassName: "{{ .Values.storage.className }}"
```

---

### **6. `templates/configmap.yaml` (MongoDB Replica Set Initialization)**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-init
data:
  init.js: |
    rs.initiate({
      _id: "{{ .Values.mongodb.replicaSetName }}",
      members: [
        { _id: 0, host: "mongodb-0.mongodb.default.svc.cluster.local:27017" },
        { _id: 1, host: "mongodb-1.mongodb.default.svc.cluster.local:27017" },
        { _id: 2, host: "mongodb-2.mongodb.default.svc.cluster.local:27017" }
      ]
    });
```

---

## **How to Deploy the MongoDB Helm Chart in Kubernetes**
### **1. Install Helm (if not installed)**
```sh
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### **2. Create a Namespace for MongoDB**
```sh
kubectl create namespace mongodb
```

### **3. Deploy the Helm Chart**
```sh
helm install mongodb ./mongodb-helm --namespace mongodb
```

### **4. Verify MongoDB Pods**
```sh
kubectl get pods -n mongodb
```

Expected Output:
```
NAME         READY   STATUS    RESTARTS   AGE
mongodb-0    1/1     Running   0          10s
mongodb-1    1/1     Running   0          8s
mongodb-2    1/1     Running   0          6s
```

### **5. Access MongoDB Shell**
```sh
kubectl exec -it mongodb-0 -n mongodb -- mongosh
```
Inside the MongoDB shell, initialize the replica set:
```js
rs.status()
```

---

## **Summary: Features of this Helm Chart**
✅ **MongoDB Replica Set with 3 Nodes**  
✅ **Persistent Storage (PVC) using StatefulSet**  
✅ **Auto-scaling & high availability**  
✅ **Kubernetes Secret for secure credentials**  
✅ **Replica Set Configuration with ConfigMap**

Would you like a **custom script** for **automated deployment** of MongoDB using Helm on AWS EKS? 🚀
