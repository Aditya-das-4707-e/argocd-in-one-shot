# ArgoCD Setup and Installation

Let's see how we can Setup & Install ArgoCD (UI and CLI) and access via the browser.

---

# Prerequisites

Before starting, ensure you have the following installed on your system:

1. **Docker** → Required for Kind to run containers as cluster nodes.

   ```bash
   sudo apt-get update
   sudo apt install docker.io -y
   sudo usermod -aG docker $USER && newgrp docker
   docker --version

   docker ps
   ```

2. **Kind (Kubernetes in Docker)** → To create the cluster.

   ```bash
   kind version
   ```

   [Install Guide](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)

3. **kubectl** → To interact with the cluster.

   ```bash
   kubectl version --client
   ```

   [Install Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

4. **Helm (for Helm-based installation)**

   ```bash
   helm version
   ```

   [Install Guide](https://helm.sh/docs/intro/install/)

---

> [!IMPORTANT]
> 
> You can either follow the below steps or directly run the script [setup_argocd.sh](./setup_argocd.sh)
> 
> The script will create **kind cluster** and **Installs ArgoCD UI and CLI** based on your choice (using HELM or manifest)
> 
> But before using this guide or `setup_argocd.sh`, make sure you replace the `172.31.19.178` address with your EC2 instance private ip in Cluster config for `apiServerAddress`

---

# Step 1: Create Kind Cluster

Save your cluster config as `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "172.31.19.178"   # Change this to your EC2 private IP (run "hostname -I" to check or from your EC2 dashboard)
  apiServerPort: 33893
nodes:
  - role: control-plane
    image: kindest/node:v1.33.1
  - role: worker
    image: kindest/node:v1.33.1
  - role: worker
    image: kindest/node:v1.33.1
```

> Why `apiServerAddress` & `apiServerPort` in kind config?
→ To ensure each kind cluster API server is reachable from the ArgoCD pods. This avoids conflicts (since kind defaults to random localhost ports).

Create the cluster:

```bash
kind create cluster --name argocd-cluster --config kind-config.yaml
```

Verify:

```bash
kubectl cluster-info
kubectl get nodes
```

---

#  Step 2: Install ArgoCD

We’ll cover **two professional installation methods**.

---

## **Method 1: Install ArgoCD using Helm** (recommended for customization/production)

### 1. Add Argo Helm repo

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### 2. Create namespace

```bash
kubectl create namespace argocd
```

### 3. Install ArgoCD

```bash
helm install argocd argo/argo-cd -n argocd
```

### 4. Verify installation

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

### 5. Access the ArgoCD UI

Port-forward the service:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address=0.0.0.0 &
```

Now open → **[https://<instance_public_ip>:8080](https://<instance_public_ip>:8080)**

### 6. Get initial admin password

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login with:

* Username: `admin`
* Password: (above output)

### ArgoCD CLI 
1. Install
```bash
sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
```
2. Provide executable permission to this CLI
```bash
sudo chmod +x /usr/local/bin/argocd
```

---

## **Method 2: Install ArgoCD using Official Manifests (kubectl apply)**

(fastest for demos & learning)

### 1. Create namespace

```bash
kubectl create namespace argocd
```

### 2. Apply ArgoCD installation manifest

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 3. Verify installation

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

### 4. Expose ArgoCD server

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address=0.0.0.0 &
```

Access → **[https://<instance_public_ip>:8080](https://<instance_public_ip>:8080)**

### 5. Get initial password

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login with:

* Username: `admin`
* Password: (above output)

---

# Step 3: Install ArgoCD CLI (Ubuntu/Linux)

ArgoCD server runs inside Kubernetes, but to interact with it from the terminal you need the **ArgoCD CLI (`argocd`)**.  
This is separate from the server installation.

### 1. Install ArgoCD CLI

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

### 2. Verify installation

```bash
# Verify installation
argocd version --client
```

### 3. Login to ArgoCD CLI

```bash
argocd login <instance_public_ip>:8080 --username admin --password <initial_password> --insecure
```

> Note: The --insecure flag is required when using port-forward with self-signed TLS certs.
For production, you’d configure proper TLS certs (then --insecure is not needed).

### 4. Get user info

```bash
argocd account get-user-info
```

---

#  Helm vs Manifest Installation

| Feature         | Helm Install (Method 1)     | Manifests (Method 2)         |
| --------------- | --------------------------- | ---------------------------- |
| **Flexibility** | High (override values.yaml) | Low (default configs only)   |
| **Ease of Use** | Requires Helm               | Works with just kubectl      |
| **Best for**    | Production & customization  | Quick demo / lab environment |

---

# Professional Best Practices

* For **local demo/testing** → use **kubectl apply**.
* For **production or enterprise** → use **Helm** (better upgrades & customization).
* Always **separate namespaces** (don’t install into `default`).
* Store **Application CRDs** in Git repos (GitOps best practice).


