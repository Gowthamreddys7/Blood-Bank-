# 🩸 Blood Bank — Kubernetes Deployment on AWS EC2

A containerized Blood Bank web application deployed on a Kubernetes cluster running on AWS EC2. This project demonstrates a production-ready Kubernetes setup with high availability, auto-scaling, persistent storage, and secure configuration management.

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Kubernetes Manifests](#kubernetes-manifests)
- [Deployment Guide](#deployment-guide)
- [Accessing the Application](#accessing-the-application)
- [Scaling & High Availability](#scaling--high-availability)
- [Configuration & Secrets](#configuration--secrets)
- [Cleanup](#cleanup)

---

## Project Overview

The Blood Bank Application is a web platform for managing blood donation and inventory. It is containerized using Docker and orchestrated on Kubernetes (self-managed on AWS EC2), providing:

- Scalable and resilient application deployment
- Persistent database storage with StatefulSets
- Auto-scaling based on CPU/memory load
- Secure secret and config management
- Ingress-based HTTP routing

---

## Architecture


                        Internet
                           │
                    [ Ingress Controller ]
                           │
              ┌────────────┴────────────┐
              │                         │
      [ App Service ]           [ DB Service ]
              │                         │
     [ App Deployment ]       [ StatefulSet (DB) ]
     (Replicated Pods)        (Persistent Volume)
              │                         │
       [ ConfigMap ]             [ PVC / PV ]
       [ Secrets ]
              │
         [ HPA ] (Auto-scaling)
         [ PDB ] (Disruption Budget)


Infrastructure: AWS EC2 instances running a self-managed Kubernetes cluster  
Containerization: Docker (app + database images)  
Networking: Kubernetes Ingress, ClusterIP Services, NetworkPolicy  
Storage: PersistentVolumeClaim for database durability

## Repository Structure

Blood-Bank/
├── docker-app              # Dockerfile for the Blood Bank web application
├── docker-db               # Dockerfile for the database
├── Deployment.yml          # Kubernetes Deployment for the app (replicated pods)
├── statefulset.yml         # StatefulSet for the database (stable storage)
├── app-service.yml         # Service to expose the application internally
├── db-service.yml          # Service to expose the database internally
├── ingress.yml             # Ingress rules for external HTTP routing
├── confimap.yml            # ConfigMap for non-sensitive app configuration
├── secrets.yml             # Kubernetes Secrets for sensitive credentials
├── pvc.yml                 # PersistentVolumeClaim for database storage
├── network.yml             # NetworkPolicy for pod-to-pod traffic control
├── Hpa.yml                 # HorizontalPodAutoscaler for app auto-scaling
├── pdb.yml                 # PodDisruptionBudget for high availability
├── resorce-quota.yml       # ResourceQuota to limit namespace resource usage
└── README.md


---

## Prerequisites

Ensure the following are installed and configured:

| Tool | Purpose |
|------|---------|
| AWS EC2 | Kubernetes worker/master nodes |
| `kubectl` | Kubernetes CLI |
| `kubeadm` | Cluster bootstrapping (if self-managed) |
| Docker | Container runtime |
| An Ingress Controller | e.g., NGINX Ingress Controller |

---

## Getting Started

### 1. Clone the Repository

bash
git clone https://github.com/Gowthamreddys7/Blood-Bank.git
cd Blood-Bank


### 2. Build Docker Images

```bash
# Build the application image
docker build -f docker-app -t blood-bank-app:latest .

# Build the database image
docker build -f docker-db -t blood-bank-db:latest .
```

> Push images to Docker Hub or your private registry and update the image references in `Deployment.yml` and `statefulset.yml` accordingly.

### 3. Apply Namespace & Resource Quota (Optional but Recommended)

```bash
kubectl create namespace blood-bank
kubectl apply -f resorce-quota.yml -n blood-bank
```

---

## Kubernetes Manifests

### `Deployment.yml` — Application Deployment
Defines the replicated pods for the Blood Bank web app. Controls the number of replicas, container image, resource limits, and environment variable injection.

### `statefulset.yml` — Database StatefulSet
Deploys the database with stable network identity and persistent storage. Ensures data survives pod restarts.

### `app-service.yml` — Application Service
Exposes the app pods inside the cluster via a ClusterIP service so the Ingress can route traffic to them.

### `db-service.yml` — Database Service
Provides a stable DNS name for the database so the application pods can connect to it reliably.

### `ingress.yml` — Ingress
Configures external HTTP/HTTPS routing rules. Maps a hostname or path to the application service.

### `confimap.yml` — ConfigMap
Stores non-sensitive environment variables (e.g., database host, app settings) that are injected into pods.

### `secrets.yml` — Secrets
Stores sensitive data like database passwords and API keys in base64-encoded form. Referenced by the Deployment and StatefulSet.

### `pvc.yml` — PersistentVolumeClaim
Requests persistent storage for the database so data is not lost when pods are rescheduled.

### `network.yml` — NetworkPolicy
Restricts pod-to-pod traffic. Allows only the app pods to communicate with the database pods, improving security.

### `Hpa.yml` — HorizontalPodAutoscaler
Automatically scales the number of app pod replicas up or down based on CPU or memory utilization.

### `pdb.yml` — PodDisruptionBudget
Ensures a minimum number of app pods remain available during voluntary disruptions (e.g., node drains, upgrades).

### `resorce-quota.yml` — ResourceQuota
Limits total CPU, memory, and pod count in the namespace to prevent resource exhaustion on the cluster.

---

## Deployment Guide

Apply manifests in the following order:

```bash
# 1. Secrets and Config (must exist before pods start)
kubectl apply -f secrets.yml -n blood-bank
kubectl apply -f confimap.yml -n blood-bank

# 2. Storage
kubectl apply -f pvc.yml -n blood-bank

# 3. Network Policy
kubectl apply -f network.yml -n blood-bank

# 4. Database (StatefulSet + Service)
kubectl apply -f statefulset.yml -n blood-bank
kubectl apply -f db-service.yml -n blood-bank

# 5. Application (Deployment + Service)
kubectl apply -f Deployment.yml -n blood-bank
kubectl apply -f app-service.yml -n blood-bank

# 6. Ingress
kubectl apply -f ingress.yml -n blood-bank

# 7. Auto-scaling & Availability
kubectl apply -f Hpa.yml -n blood-bank
kubectl apply -f pdb.yml -n blood-bank
```

### Verify Everything is Running

```bash
kubectl get all -n blood-bank
kubectl get ingress -n blood-bank
kubectl get pvc -n blood-bank
```

---

## Accessing the Application

Once the Ingress is set up, get the external IP:

```bash
kubectl get ingress -n blood-bank
```

Then open your browser and navigate to:

```
http://<INGRESS-EXTERNAL-IP>
```

> If using a custom domain, update your DNS records to point to the Ingress IP.

---

## Scaling & High Availability

The **HPA** automatically scales the application:

```bash
# View current scaling status
kubectl get hpa -n blood-bank
```

The **PDB** protects availability during maintenance:

```bash
kubectl get pdb -n blood-bank
```

To manually scale:

```bash
kubectl scale deployment blood-bank-app --replicas=5 -n blood-bank
```

---

## Configuration & Secrets

**ConfigMap** — Edit non-sensitive settings:

```bash
kubectl edit configmap -n blood-bank
```

**Secrets** — Update sensitive values (base64 encode first):

```bash
echo -n 'your-password' | base64
kubectl edit secret -n blood-bank
```

---

## Cleanup

To remove all resources:

```bash
kubectl delete namespace blood-bank
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Application | Web App (Dockerized) |
| Database | Containerized DB with StatefulSet |
| Orchestration | Kubernetes |
| Cloud | AWS EC2 |
| Networking | Kubernetes Ingress, NetworkPolicy |
| Scaling | HorizontalPodAutoscaler |
| Storage | PersistentVolumeClaim |

---

## 👤 Author

**Gowthamreddys7**  
GitHub: [github.com/Gowthamreddys7](https://github.com/Gowthamreddys7)
