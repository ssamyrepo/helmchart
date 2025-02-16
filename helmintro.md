**Helm** is the **package manager for Kubernetes**—think of it like `apt` for Ubuntu or `yum` for RHEL, but for Kubernetes. It helps deploy, manage, and upgrade applications inside Kubernetes clusters **easily and repeatably**. 🚀

---

## **📌 Why Use Helm?**
✔ **Simplifies Kubernetes deployments** – No need to write long YAML files.  
✔ **Manages application versions** – Helm tracks installed versions like a package manager.  
✔ **Enables repeatability** – Deploy apps with the same configuration every time.  
✔ **Customizable** – Use **Helm values** to tweak deployments without modifying YAML files.  

---

## **🚀 Installing Helm**
### **📍 1. Install Helm (Linux/Mac/Windows)**
Run this command:

```sh
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

To verify installation:
```sh
helm version
```
You should see output like:
```
version.BuildInfo{Version:"v3.x.x", GitCommit:"xyz", GoVersion:"go1.x.x"}
```

---

## **🎯 Helm Basics**
Once Helm is installed, you can:
1. **Add Helm Repositories**  
2. **Search for Applications (Charts)**  
3. **Install & Upgrade Applications**  
4. **Uninstall Applications**  

---

## **📌 1. Add Helm Repositories**
A **Helm repository** is like a storehouse of applications (called **charts**).

Add a popular Helm repo (Bitnami):
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```
Update repo to get the latest charts:
```sh
helm repo update
```

To **list all added repositories**:
```sh
helm repo list
```

---

## **📌 2. Search for a Helm Chart**
To find applications, use:
```sh
helm search repo <keyword>
```
Example: Search for `mongodb`:
```sh
helm search repo mongodb
```
Output:
```
bitnami/mongodb      12.1.31  A Helm chart for MongoDB
bitnami/mongodb-sharded 4.1.13 Sharded MongoDB Helm chart
```

---

## **📌 3. Install an Application (Chart)**
To install **MongoDB** using Helm:
```sh
helm install my-mongo bitnami/mongodb
```
Breakdown:
- `helm install` → Command to install an application
- `my-mongo` → Custom release name for the installation
- `bitnami/mongodb` → The Helm chart to install

---

## **📌 4. List Installed Helm Releases**
To check which applications are installed:
```sh
helm list
```
Example Output:
```
NAME       	NAMESPACE 	REVISION	UPDATED                                	STATUS  	CHART          	APP VERSION
my-mongo 	default   	1       	2024-02-14 15:05:43.123456789 +0000 UTC	deployed	mongodb-12.1.31	5.0.6
```

---

## **📌 5. Uninstall an Application**
To remove **MongoDB**:
```sh
helm uninstall my-mongo
```
Verify it's gone:
```sh
helm list
```

---

## **🎯 Customizing Helm Deployments**
Sometimes, you need to **change configurations** (e.g., set a **password**, modify **replicas**, etc.). You can do this using **Helm values**.

### **📍 1. Check Default Values**
To see the default settings for MongoDB:
```sh
helm show values bitnami/mongodb
```

---

### **📍 2. Override Helm Values**
You can **pass custom values** while installing:

Example: Set a **MongoDB root password**:
```sh
helm install my-mongo bitnami/mongodb --set auth.rootPassword="MyStrongPassword"
```

You can also **store values in a YAML file**:

📄 **`values.yaml`**:
```yaml
auth:
  rootPassword: "MyStrongPassword"
  username: "mongoAdmin"
  password: "MongoPass123"
replicaCount: 3
```

Then install using:
```sh
helm install my-mongo bitnami/mongodb -f values.yaml
```

---

## **📌 6. Upgrade an Application**
To **change configuration** after installation:
```sh
helm upgrade my-mongo bitnami/mongodb --set replicaCount=5
```

---

## **📌 7. Rollback to a Previous Version**
To list revision history:
```sh
helm history my-mongo
```

Rollback to version `1`:
```sh
helm rollback my-mongo 1
```

---

## **📌 8. Debugging & Logs**
Check if Helm deployment failed:
```sh
helm status my-mongo
```
Get pod logs:
```sh
kubectl logs -l app.kubernetes.io/name=mongodb
```

---

## **🔥 Practical Example: Deploying WordPress**
Let's deploy **WordPress** with **a MySQL backend** using Helm.

```sh
helm install my-wordpress bitnami/wordpress --set wordpressPassword="MyWordPressPass"
```

To get WordPress URL:
```sh
kubectl get svc --namespace default -w
```

---

## **📝 Summary**
| Command | Purpose |
|---------|---------|
| `helm repo add <repo>` | Add a Helm repository |
| `helm repo update` | Update Helm repositories |
| `helm search repo <name>` | Search for a Helm chart |
| `helm install <name> <chart>` | Install an application |
| `helm list` | List installed applications |
| `helm uninstall <name>` | Remove an application |
| `helm show values <chart>` | Show default configurations |
| `helm upgrade <name> <chart>` | Upgrade an app |
| `helm rollback <name> <revision>` | Rollback to previous version |
| `helm status <name>` | Get app status |
| `kubectl logs -l <label>` | Check logs |

---

## **🎯 TL;DR**
- **Helm = Kubernetes package manager** 🎯  
- **Charts = Pre-configured Kubernetes apps** 📦  
- **`helm install <name> <chart>` → Deploy an app** 🚀  
- **Use `helm upgrade` to modify an app** 🔄  
- **Store configs in `values.yaml` for customization** 🛠  


