# MongoDB Kubernetes Deployment with Color API

A production-ready Kubernetes deployment showcasing MongoDB StatefulSet with a Node.js REST API for managing color data.

## Project Overview

This project demonstrates:
- MongoDB deployment using Kubernetes StatefulSet with persistent storage
- Headless Service for stable network identities
- Secret management for database credentials
- ConfigMap for database initialization
- REST API (Color API) connected to MongoDB

## Architecture

```
External Traffic
      │
      │ Port 30007
      v
┌──────────────────────┐
│   Color Service      │ (NodePort - routes traffic)
│   Port: 80           │
│   TargetPort: 80     │
└──────────┬───────────┘
           │
           │ Selects pods with label: app=color-api
           v
┌──────────────────────┐
│   Color API Pod      │ (Deployment)
│   Port: 80           │
│   Connects to DB     │
└──────────┬───────────┘
           │
           │ Connects to: mongodb-ss-0.mongodb-svc.default.svc.cluster.local:27017
           v
┌──────────────────────┐
│   MongoDB Service    │ (Headless - provides DNS)
│   ClusterIP: None    │
│   Port: 27017        │
└──────────┬───────────┘
           │
           │ Selects pods with label: app=mongodb
           v
┌──────────────────────┐
│   MongoDB Pod        │ (StatefulSet)
│   mongodb-ss-0       │
│   Port: 27017        │
│   ┌────────────────┐ │
│   │ PVC: 5Gi       │ │
│   └────────────────┘ │
└──────────────────────┘
```

## Components

### MongoDB Layer

- **StatefulSet** (`mongodb-ss`): Manages MongoDB pod with stable hostname
- **Headless Service** (`mongodb-svc`): Provides network identity for StatefulSet
- **PersistentVolumeClaim**: 5Gi storage for MongoDB data
- **Secrets**:
  - `mongodb-root-creds`: Root admin credentials
  - `mongodb-colordb-creds`: Application database user credentials
- **ConfigMap** (`mongodb-init-colordb`): Database initialization script

### Application Layer

- **Deployment** (`color-api`): Node.js REST API (image: `ifeolaitan/color-api:2.0.0`)
- **Service** (`color-svc`): NodePort service exposing API on port 30007
- **Database**: Connects to `colordb` database in MongoDB

## Prerequisites

- Kubernetes cluster (minikube, kind, or cloud provider)
- kubectl configured
- Docker (if building custom images)

## Quick Start

### 1. Deploy MongoDB

```bash
# Create secrets
kubectl apply -f k8s-manifests/mongodb-root-creds.yaml
kubectl apply -f k8s-manifests/mongodb-colordb-creds.yaml

# Create database initialization ConfigMap
kubectl apply -f k8s-manifests/mongodb-init-colordb.yaml

# Deploy MongoDB StatefulSet and Service
kubectl apply -f k8s-manifests/mongodb-ss.yaml
kubectl apply -f k8s-manifests/mongodb-svc.yaml
```

### 2. Verify MongoDB is Running

```bash
# Check StatefulSet status
kubectl get statefulset mongodb-ss

# Check pod is ready
kubectl get pods -l app=mongodb

# Check persistent volume
kubectl get pvc
```

### 3. Deploy Color API

```bash
# Deploy the application
kubectl apply -f k8s-manifests/color-api.yaml

# Deploy the service
kubectl apply -f k8s-manifests/color-svc.yaml
```

### 4. Test the API

```bash
# Get the NodePort service URL
kubectl get svc color-svc

# Access the API (adjust IP based on your cluster)
# For minikube:
minikube service color-svc --url

# For direct access:
curl http://<node-ip>:30007/api?colorKey=primary
```

## Configuration

### Environment Variables (Color API)

| Variable | Source | Description |
|----------|--------|-------------|
| `DB_USER` | Secret: mongodb-colordb-creds | Database username |
| `DB_PASSWORD` | Secret: mongodb-colordb-creds | Database password |
| `DB_HOST` | Value | MongoDB hostname (StatefulSet pod) |
| `DB_PORT` | Value | MongoDB port (27017) |
| `DB_NAME` | Value | Database name (colordb) |
| `DB_URL` | Computed | Full MongoDB connection string |

### MongoDB Connection

The Color API connects to MongoDB using the StatefulSet pod's stable DNS name:
```
mongodb-ss-0.mongodb-svc.default.svc.cluster.local:27017
```

## Resource Limits

### Color API
- Memory: 128Mi
- CPU: 500m

## Troubleshooting

### Check pod logs
```bash
# MongoDB logs
kubectl logs mongodb-ss-0

# Color API logs
kubectl logs -l app=color-api
```

### Verify database connectivity
```bash
# Exec into Color API pod
kubectl exec -it <color-api-pod-name> -- sh

# Test MongoDB connection
nc -zv mongodb-svc 27017
```

### Check secrets
```bash
kubectl get secrets
kubectl describe secret mongodb-colordb-creds
```

### Verify ConfigMap
```bash
kubectl get configmap mongodb-init-colordb
kubectl describe configmap mongodb-init-colordb
```

## API Endpoints

The Color API provides endpoints for managing color data stored in MongoDB.

### Base URL

```
http://<node-ip>:30007
```

For minikube users:
```bash
minikube service color-svc --url
```

### Available Endpoints

#### Health Check Endpoints

| Endpoint | Method | Description | Response |
|----------|--------|-------------|----------|
| `/health` | GET | Liveness probe | `ok` or 503 |
| `/ready` | GET | Readiness probe | `ok` or 503 |
| `/up` | GET | Basic health check | `ok` |

#### Color Management Endpoints

| Endpoint | Method | Description | Parameters | Response |
|----------|--------|-------------|------------|----------|
| `/api` | GET | Get a single color | `colorKey` (query), `format` (query, optional) | Plain text or JSON |
| `/api/color` | GET | Get all colors | None | JSON array of colors |
| `/api/color/:key` | GET | Get color by key | `key` (path param) | JSON object or 404 |
| `/api/color/:key` | POST | Create/update color | `key` (path param), `value` (body) | 201 with created color |
| `/api/color/:key` | DELETE | Delete color | `key` (path param) | 204 No Content |

## Project Structure

```
.
├── README.md
├── color-api/                          # Color API application source
│   ├── Dockerfile
│   ├── package.json
│   └── src/
└── k8s-manifests/                      # Kubernetes manifests
    ├── mongodb-root-creds.yaml         # MongoDB root credentials
    ├── mongodb-colordb-creds.yaml      # App database credentials
    ├── mongodb-init-colordb.yaml       # Database init script
    ├── mongodb-ss.yaml                 # MongoDB StatefulSet
    ├── mongodb-svc.yaml                # MongoDB headless service
    ├── color-api.yaml                  # Color API deployment
    └── color-svc.yaml                  # Color API service
```

## Cleanup

```bash
# Delete all resources
kubectl delete -f k8s-manifests/

# Or delete individually
kubectl delete deployment color-api
kubectl delete service color-svc
kubectl delete statefulset mongodb-ss
kubectl delete service mongodb-svc
kubectl delete configmap mongodb-init-colordb
kubectl delete secret mongodb-root-creds mongodb-colordb-creds
kubectl delete pvc mongodb-data-mongodb-ss-0
```