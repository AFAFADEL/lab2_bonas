# Lab 3: Multi-Environment Deployments with Namespace Isolation

## Scenario
Your team now manages applications across Development (dev) and Staging (staging) environments within the same cluster. You need to deploy a two-tier application (frontend and backend) in both environments using Deployments, expose them appropriately, and ensure proper isolation and internal communication patterns.

## Lab Description
The student will create `dev` and `staging` namespaces, deploy a frontend and backend application using Deployments, and expose them with Services. They will also configure the frontend to communicate with the backend via DNS.

## Tasks to Implement

1. Create two namespaces: `dev` and `staging`.

2. In each namespace:
   - Create a backend Deployment (e.g., `redis:alpine` or a custom Python HTTP server) with 1 replica.
   - Expose the backend Deployment internally with a ClusterIP service named `backend`.
   - Create a frontend Deployment (e.g., `nginx`) with 2 replicas.
   - Expose the frontend Deployment internally with a ClusterIP service named `frontend`.

3. Configure the frontend pods (e.g., via ConfigMap or environment variable) to connect to the backend service using the name `backend` within its namespace.

4. **Pod Networking Challenge:** From a temporary debug pod in the `dev` namespace, demonstrate that you can resolve `backend` to the dev backend service, and `backend.staging.svc.cluster.local` to the staging backend service.

5. Perform a rolling update on the frontend Deployment in `staging` only, changing the Nginx page to say "Staging Environment". Verify the `dev` frontend remains unchanged.
-------------------
# Lab 3: Multi-Environment Deployments with Namespace Isolation

## Overview

In this lab we simulate a real production scenario where applications run in multiple environments within the same Kubernetes cluster.

We will create **two namespaces**:

* dev
* staging

Inside each namespace we deploy a **two-tier application**:

* Frontend (Nginx)
* Backend (Redis)

Each component will be exposed using **ClusterIP Services**, and the frontend will communicate with the backend using **Kubernetes DNS**.

---

# Architecture

```
dev namespace
 ├── backend Deployment (redis)
 ├── backend Service
 ├── frontend Deployment (nginx)
 └── frontend Service

staging namespace
 ├── backend Deployment (redis)
 ├── backend Service
 ├── frontend Deployment (nginx)
 └── frontend Service
```

Each namespace is isolated, but services can still be reached via **DNS**.

---

# Step 1: Create Namespaces

```
kubectl create namespace dev
kubectl create namespace staging
```

Verify:

```
kubectl get namespaces
```

---

# Step 2: Backend Deployment (dev)

Create file:

```
backend-dev.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
```

Apply:

```
kubectl apply -f backend-dev.yaml
```

---

# Step 3: Backend Service (dev)

```
kubectl expose deployment backend \
--namespace=dev \
--name=backend \
--type=ClusterIP \
--port=6379
```

Verify:

```
kubectl get svc -n dev
```

---

# Step 4: Frontend Deployment (dev)

Create file:

```
frontend-dev.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        env:
        - name: BACKEND_SERVICE
          value: backend
```

Apply:

```
kubectl apply -f frontend-dev.yaml
```

---

# Step 5: Frontend Service (dev)

```
kubectl expose deployment frontend \
--namespace=dev \
--name=frontend \
--type=ClusterIP \
--port=80
```

Verify:

```
kubectl get all -n dev
```

---

# Step 6: Repeat for Staging Namespace

Deploy the same application in **staging**.

Backend:

```
kubectl create deployment backend \
--image=redis:alpine \
--replicas=1 \
--namespace=staging
```

Expose backend:

```
kubectl expose deployment backend \
--namespace=staging \
--name=backend \
--port=6379 \
--type=ClusterIP
```

Frontend:

```
kubectl create deployment frontend \
--image=nginx \
--replicas=2 \
--namespace=staging
```

Expose frontend:

```
kubectl expose deployment frontend \
--namespace=staging \
--name=frontend \
--port=80 \
--type=ClusterIP
```

---

# Step 7: Test DNS and Pod Networking

Create a temporary debug pod inside the **dev namespace**:

```
kubectl run debug \
--rm -it \
--image=busybox \
--namespace=dev \
-- /bin/sh
```

Inside the pod test DNS:

Resolve backend in dev:

```
nslookup backend
```

Test staging backend:

```
nslookup backend.staging.svc.cluster.local
```

Explanation:

```
backend                    → dev namespace backend
backend.staging.svc.cluster.local → staging backend
```

Exit the debug pod:

```
exit
```

---

# Step 8: Rolling Update in Staging Only

Update the frontend deployment in staging:

```
kubectl edit deployment frontend -n staging
```

Change the container command to display a custom page:

```
command: ["/bin/sh"]
args: ["-c", "echo 'Staging Environment' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"]
```

Save and exit.

Watch rollout:

```
kubectl rollout status deployment frontend -n staging
```

---

# Step 9: Verify Isolation

Check staging frontend pods:

```
kubectl get pods -n staging
```

Check dev frontend pods:

```
kubectl get pods -n dev
```

The **staging frontend shows the new page**, while **dev frontend remains unchanged**.

This demonstrates **environment isolation using namespaces**.

---

# Key Concepts Learned

| Concept           | Description                                 |
| ----------------- | ------------------------------------------- |
| Namespace         | Logical isolation inside the cluster        |
| Multi-Environment | Running dev and staging in the same cluster |
| Service Discovery | Pods communicate using DNS names            |
| ClusterIP         | Internal service communication              |
| Rolling Updates   | Updating applications without downtime      |
| DNS Resolution    | Accessing services across namespaces        |

---

# Useful Commands

```
kubectl get all -n dev
kubectl get all -n staging
kubectl get svc -A
kubectl describe pod <pod-name>
kubectl rollout status deployment <deployment>
```

---

# Conclusion

This lab demonstrates how Kubernetes supports **multi-environment deployments** using namespaces.
Applications can be isolated logically while still communicating internally using **DNS-based service discovery**.
