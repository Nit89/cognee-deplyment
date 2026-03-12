# cognee-deplyment
To deploy Cognee on Azure Virtual Machine and Azure Kubernetes Service (AKS), you basically run the Cognee SDK service plus its required storage components (vector DB + graph DB). Cognee uses a three-tier architecture:

Graph database → stores relationships/entities

Vector database → stores embeddings

Application layer → Cognee SDK / API service

Below is a DevOps-style deployment approach for both Azure VM and AKS, which is typically expected in interviews or production setups.

1️⃣ Deploy Cognee on Azure Virtual Machine
Architecture
Azure VM (Ubuntu)
 ├─ Docker
 ├─ Cognee SDK API
 ├─ Vector DB (Qdrant / Milvus / Redis)
 └─ Graph DB (Neo4j / FalkorDB / Kuzu)
Step 1 — Create Azure VM
az vm create \
  --resource-group cognee-rg \
  --name cognee-vm \
  --image Ubuntu2204 \
  --size Standard_D4s_v3 \
  --admin-username azureuser \
  --generate-ssh-keys

Open ports:

22   SSH
8000 Cognee API
6333 Vector DB
7474 Neo4j
Step 2 — Install Docker

SSH into VM.

sudo apt update
sudo apt install docker.io docker-compose -y
sudo usermod -aG docker $USER
Step 3 — Clone Cognee Repo
git clone https://github.com/topoteretes/cognee.git
cd cognee
Step 4 — Create docker-compose

Example:

version: "3"

services:

  cognee:
    image: cogneeai/cognee:latest
    ports:
      - "8000:8000"
    environment:
      VECTOR_DB=qdrant
      GRAPH_DB=neo4j
    depends_on:
      - qdrant
      - neo4j

  qdrant:
    image: qdrant/qdrant
    ports:
      - "6333:6333"

  neo4j:
    image: neo4j
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      NEO4J_AUTH=neo4j/password
Step 5 — Run services
docker compose up -d

Test:

http://<VM-IP>:8000
2️⃣ Deploy Cognee on AKS (Production Approach)
Architecture
Azure AKS
   │
   ├── Cognee API (Deployment)
   ├── Vector DB (StatefulSet)
   ├── Graph DB (StatefulSet)
   ├── Persistent Volumes
   └── LoadBalancer / Ingress

AKS is ideal because it provides scaling and managed Kubernetes control plane for container workloads.

Step 1 — Create AKS cluster
az group create --name cognee-rg --location eastus

az aks create \
  --resource-group cognee-rg \
  --name cognee-aks \
  --node-count 2 \
  --enable-managed-identity \
  --generate-ssh-keys

Connect:

az aks get-credentials \
  --resource-group cognee-rg \
  --name cognee-aks
Step 2 — Deploy Vector DB

Example Qdrant

apiVersion: apps/v1
kind: Deployment
metadata:
  name: qdrant
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

Service:

apiVersion: v1
kind: Service
metadata:
  name: qdrant
spec:
  selector:
    app: qdrant
  ports:
    - port: 6333
Step 3 — Deploy Graph DB

Example Neo4j

apiVersion: apps/v1
kind: Deployment
metadata:
  name: neo4j
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
        image: neo4j
        ports:
        - containerPort: 7687
        - containerPort: 7474
Step 4 — Deploy Cognee API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cognee
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
        - name: GRAPH_DB_HOST
          value: neo4j
Step 5 — Expose service
apiVersion: v1
kind: Service
metadata:
  name: cognee-service
spec:
  type: LoadBalancer
  selector:
    app: cognee
  ports:
  - port: 80
    targetPort: 8000

Apply:

kubectl apply -f .

Check:

kubectl get svc
3️⃣ Production Improvements (Important for interviews)

Add these:

Storage
Azure Disk / Azure Files
Secrets
Kubernetes Secrets
Azure Key Vault
Scaling
HPA
Cluster Autoscaler
Networking
Ingress Controller
TLS
Observability
Prometheus
Grafana
Azure Monitor
4️⃣ CI/CD (Recommended)

Pipeline:

GitHub
   ↓
Build Docker image
   ↓
Push to Azure Container Registry
   ↓
Deploy to AKS via Helm

✅ Summary

Platform	Use case
Azure VM	Quick POC / development
AKS	Scalable production deployment

💡 Since you are a DevOps engineer preparing for interviews, I can also show you:

Production architecture diagram for Cognee on AKS

Helm chart deployment

Terraform to deploy VM + AKS + Cognee automatically

These are very impressive in DevOps interviews.
