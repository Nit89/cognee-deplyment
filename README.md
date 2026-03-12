# Cognee Deployment Guide

This guide covers how to deploy **Cognee** on **Azure Virtual Machine (VM)** and **Azure Kubernetes Service (AKS)**. Cognee uses a three-tier architecture:

| Layer | Component |
|---|---|
| **Graph database** | Stores relationships and entities (e.g. Neo4j, FalkorDB, Kuzu) |
| **Vector database** | Stores embeddings (e.g. Qdrant, Milvus, Redis) |
| **Application layer** | Cognee SDK / API service |

---

## 1️⃣ Deploy Cognee on Azure Virtual Machine

### Architecture

```
Azure VM (Ubuntu)
 ├─ Docker
 ├─ Cognee SDK API
 ├─ Vector DB (Qdrant / Milvus / Redis)
 └─ Graph DB (Neo4j / FalkorDB / Kuzu)
```

### Step 1 — Create Azure VM

Use the Azure CLI to provision an Ubuntu VM with enough resources to run the Cognee stack:

```bash
az vm create \
  --resource-group cognee-rg \
  --name cognee-vm \
  --image Ubuntu2204 \
  --size Standard_D4s_v3 \
  --admin-username azureuser \
  --generate-ssh-keys
```

Then open the required ports so external traffic can reach your services:

```bash
# SSH access
az vm open-port --port 22   --resource-group cognee-rg --name cognee-vm --priority 100

# Cognee API
az vm open-port --port 8000 --resource-group cognee-rg --name cognee-vm --priority 200

# Qdrant Vector DB
az vm open-port --port 6333 --resource-group cognee-rg --name cognee-vm --priority 300

# Neo4j HTTP + Bolt
az vm open-port --port 7474 --resource-group cognee-rg --name cognee-vm --priority 400
az vm open-port --port 7687 --resource-group cognee-rg --name cognee-vm --priority 500
```

| Port | Service |
|------|---------|
| 22   | SSH |
| 8000 | Cognee API |
| 6333 | Qdrant Vector DB |
| 7474 | Neo4j HTTP console |
| 7687 | Neo4j Bolt protocol |

### Step 2 — Install Docker

SSH into the VM and install Docker and Docker Compose:

```bash
ssh azureuser@<VM-PUBLIC-IP>

sudo apt update && sudo apt upgrade -y
sudo apt install docker.io docker-compose -y

# Allow your user to run Docker without sudo
sudo usermod -aG docker $USER

# Apply group change without logging out
newgrp docker
```

Verify the installation:

```bash
docker --version
docker compose version
```

### Step 3 — Clone the Cognee Repository

```bash
git clone https://github.com/topoteretes/cognee.git
cd cognee
```

### Step 4 — Create a `docker-compose.yml`

Create (or edit) a `docker-compose.yml` file in the repository root with the following content. This wires together Cognee, Qdrant (vector DB), and Neo4j (graph DB):

```yaml
version: "3"

services:

  cognee:
    image: cogneeai/cognee:latest
    ports:
      - "8000:8000"
    environment:
      - VECTOR_DB=qdrant
      - GRAPH_DB=neo4j
      - VECTOR_DB_HOST=qdrant
      - VECTOR_DB_PORT=6333
      - GRAPH_DB_HOST=neo4j
      - GRAPH_DB_PORT=7687
    depends_on:
      - qdrant
      - neo4j
    restart: unless-stopped

  qdrant:
    image: qdrant/qdrant
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage
    restart: unless-stopped

  neo4j:
    image: neo4j:latest
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      - NEO4J_AUTH=neo4j/Ch@ngeMe!2024#
    volumes:
      - neo4j_data:/data
    restart: unless-stopped

volumes:
  qdrant_data:
  neo4j_data:
```

> **Security note:** Replace `neo4j/Ch@ngeMe!2024#` with a cryptographically random password (e.g. generated with `openssl rand -base64 24`). Consider storing it in an environment file (`.env`) and referencing it with `${NEO4J_PASSWORD}`.

### Step 5 — Start the Services

Launch all containers in detached mode:

```bash
docker compose up -d
```

Check that all containers are running:

```bash
docker compose ps
docker compose logs -f cognee   # tail Cognee logs
```

Test the Cognee API in your browser or with curl:

```bash
curl http://<VM-PUBLIC-IP>:8000/health
```

Or open `http://<VM-PUBLIC-IP>:8000` in a browser.

---

## 2️⃣ Deploy Cognee on AKS (Production Approach)

### Architecture

```
Azure AKS
   │
   ├── Cognee API (Deployment)
   ├── Vector DB (Qdrant — StatefulSet)
   ├── Graph DB  (Neo4j  — StatefulSet)
   ├── Persistent Volumes (Azure Disk)
   └── LoadBalancer / Ingress Controller
```

AKS provides a managed Kubernetes control plane, horizontal scaling, and native Azure integrations — ideal for production workloads.

### Step 1 — Create the AKS Cluster

```bash
# Create a resource group
az group create --name cognee-rg --location eastus

# Provision a 2-node AKS cluster with managed identity
az aks create \
  --resource-group cognee-rg \
  --name cognee-aks \
  --node-count 2 \
  --node-vm-size Standard_D4s_v3 \
  --enable-managed-identity \
  --generate-ssh-keys

# Download cluster credentials (configures kubectl)
az aks get-credentials \
  --resource-group cognee-rg \
  --name cognee-aks

# Verify connectivity
kubectl get nodes
```

### Step 2 — Deploy the Vector DB (Qdrant)

Save the following as `qdrant-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qdrant
  labels:
    app: qdrant
spec:
  replicas: 1
  selector:
    matchLabels:
      app: qdrant
  template:
    metadata:
      labels:
        app: qdrant
    spec:
      containers:
      - name: qdrant
        image: qdrant/qdrant
        ports:
        - containerPort: 6333
        volumeMounts:
        - name: qdrant-storage
          mountPath: /qdrant/storage
      volumes:
      - name: qdrant-storage
        persistentVolumeClaim:
          claimName: qdrant-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: qdrant-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: qdrant
spec:
  selector:
    app: qdrant
  ports:
    - protocol: TCP
      port: 6333
      targetPort: 6333
```

Apply:

```bash
kubectl apply -f qdrant-deployment.yaml
```

### Step 3 — Deploy the Graph DB (Neo4j)

Save the following as `neo4j-deployment.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: neo4j-secret
type: Opaque
stringData:
  NEO4J_AUTH: "neo4j/Ch@ngeMe!2024#"  # Replace with a cryptographically random password
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: neo4j
  labels:
    app: neo4j
spec:
  replicas: 1
  selector:
    matchLabels:
      app: neo4j
  template:
    metadata:
      labels:
        app: neo4j
    spec:
      containers:
      - name: neo4j
        image: neo4j:latest
        ports:
        - containerPort: 7474
        - containerPort: 7687
        envFrom:
        - secretRef:
            name: neo4j-secret
        volumeMounts:
        - name: neo4j-data
          mountPath: /data
      volumes:
      - name: neo4j-data
        persistentVolumeClaim:
          claimName: neo4j-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: neo4j-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: v1
kind: Service
metadata:
  name: neo4j
spec:
  selector:
    app: neo4j
  ports:
    - name: http
      port: 7474
      targetPort: 7474
    - name: bolt
      port: 7687
      targetPort: 7687
```

Apply:

```bash
kubectl apply -f neo4j-deployment.yaml
```

### Step 4 — Deploy the Cognee API

Save the following as `cognee-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cognee
  labels:
    app: cognee
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cognee
  template:
    metadata:
      labels:
        app: cognee
    spec:
      containers:
      - name: cognee
        image: cogneeai/cognee:latest
        ports:
        - containerPort: 8000
        env:
        - name: VECTOR_DB_HOST
          value: qdrant
        - name: VECTOR_DB_PORT
          value: "6333"
        - name: GRAPH_DB_HOST
          value: neo4j
        - name: GRAPH_DB_PORT
          value: "7687"
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 30
```

Apply:

```bash
kubectl apply -f cognee-deployment.yaml
```

### Step 5 — Expose the Cognee Service

Save the following as `cognee-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cognee-service
spec:
  type: LoadBalancer
  selector:
    app: cognee
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
```

Apply and verify:

```bash
kubectl apply -f cognee-service.yaml

# Watch until an EXTERNAL-IP is assigned (may take 1–2 minutes)
kubectl get svc cognee-service --watch
```

Once `EXTERNAL-IP` is populated, test the API:

```bash
curl http://<EXTERNAL-IP>/health
```

---

## 3️⃣ Production Improvements

The following additions are strongly recommended for any production deployment:

### Storage
- **Azure Disk** — persistent block storage for stateful workloads (databases).
- **Azure Files** — shared file storage accessible by multiple pods.

### Secrets Management
- Use **Kubernetes Secrets** to avoid hard-coding credentials in manifests.
- Integrate **Azure Key Vault** with the [Secrets Store CSI Driver](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver) to inject secrets into pods at runtime.

### Scaling
- **Horizontal Pod Autoscaler (HPA)** — automatically scales pod replicas based on CPU/memory metrics:

  ```bash
  kubectl autoscale deployment cognee --cpu-percent=60 --min=2 --max=10
  ```

- **Cluster Autoscaler** — automatically adds/removes VM nodes based on pending pods:

  ```bash
  az aks update \
    --resource-group cognee-rg \
    --name cognee-aks \
    --enable-cluster-autoscaler \
    --min-count 2 \
    --max-count 5
  ```

### Networking
- **Ingress Controller** (e.g. NGINX or Azure Application Gateway) — route traffic to multiple services via a single public IP with path-based or host-based rules.
- **TLS termination** — use [cert-manager](https://cert-manager.io/) with Let's Encrypt to automatically provision and renew TLS certificates.

### Observability
- **Prometheus + Grafana** — collect and visualise cluster and application metrics.
- **Azure Monitor + Container Insights** — native Azure monitoring with log aggregation and alerting.

  ```bash
  az aks enable-addons \
    --resource-group cognee-rg \
    --name cognee-aks \
    --addons monitoring
  ```

---

## 4️⃣ CI/CD Pipeline (Recommended)

A typical GitOps pipeline for Cognee on AKS:

```
GitHub (push / PR)
       ↓
GitHub Actions — build & test
       ↓
Build Docker image
       ↓
Push to Azure Container Registry (ACR)
       ↓
Deploy to AKS via Helm (helm upgrade --install)
```

### Example GitHub Actions Workflow (`.github/workflows/deploy.yml`)

```yaml
name: Build and Deploy Cognee

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Azure Container Registry
        uses: azure/docker-login@v1
        with:
          login-server: ${{ secrets.ACR_LOGIN_SERVER }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Build and push Docker image
        run: |
          docker build -t ${{ secrets.ACR_LOGIN_SERVER }}/cognee:${{ github.sha }} .
          docker push ${{ secrets.ACR_LOGIN_SERVER }}/cognee:${{ github.sha }}

      - name: Set AKS context
        uses: azure/aks-set-context@v3
        with:
          resource-group: cognee-rg
          cluster-name: cognee-aks
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy with Helm
        run: |
          helm upgrade --install cognee ./helm/cognee \
            --set image.repository=${{ secrets.ACR_LOGIN_SERVER }}/cognee \
            --set image.tag=${{ github.sha }}
```

---

## ✅ Summary

| Platform  | Best for |
|-----------|----------|
| **Azure VM** | Quick proof-of-concept / local development |
| **AKS**      | Scalable, production-grade deployment |

---

## 💡 Further Reading

- [Cognee GitHub Repository](https://github.com/topoteretes/cognee)
- [Azure Kubernetes Service Documentation](https://learn.microsoft.com/en-us/azure/aks/)
- [Azure VM Documentation](https://learn.microsoft.com/en-us/azure/virtual-machines/)
- Production AKS architecture diagram
- Helm chart deployment
- Terraform automation for VM + AKS + Cognee