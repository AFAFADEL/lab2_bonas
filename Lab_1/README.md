# Lab 1: From Pods to Deployments - My First Application

## Scenario
You've proven you can run a single pod. Now your tech lead wants you to deploy a web application that can handle failures automatically. You need to move from a single pod to a Deployment to ensure high availability.

## Lab Description
The student will create an Nginx Deployment, understand how it manages pods through ReplicaSets, and perform basic scaling and updates.

## Tasks to Implement

1. Using declarative YAML, create a Deployment named `web-app` with the `nginx:1.23` image. Start with 1 replica. Add a label `app: web`.

2. Use `kubectl` commands to observe the Pod Lifecycle as the Deployment creates the pod. Identify the ReplicaSet created by the Deployment.

3. Scale the Deployment to 3 replicas using an imperative command. Verify the new pods are created and spread across the cluster's pod networking.

4. Update the Deployment to use the `nginx:1.24` image. Watch the rolling update process and observe how new pods are created and old ones are terminated.

5. Roll back the Deployment to the previous version (`nginx:1.23`) using `kubectl rollout undo`.
-------------------
# Lab 1: From Pods to Deployments – My First Application

## Overview

In this lab, we deploy a web application using a **Deployment** in Kubernetes instead of a single Pod.
A Deployment provides **high availability, automatic recovery (self-healing), scaling, and safe updates**.

The application used is **Nginx** running inside a container.

---

# Step 1: Create the Deployment YAML

Create a file named:

```
web-app.yaml
```

Add the following configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web

spec:
  replicas: 1

  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web

    spec:
      containers:
      - name: nginx
        image: nginx:1.23
        ports:
        - containerPort: 80
```

---

# Step 2: Apply the Deployment

Run the following command to create the Deployment:

```
kubectl apply -f web-app.yaml
```

Verify the Deployment:

```
kubectl get deployments
```

Check the running Pods:

```
kubectl get pods
```

---

# Step 3: Identify the ReplicaSet

A Deployment automatically creates a **ReplicaSet** to manage Pods.

Check the ReplicaSet with:

```
kubectl get rs
```

You should see something similar to:

```
web-app-7d8f6c9d
```

---

# Step 4: Observe Pod Lifecycle

Watch the Pod creation process:

```
kubectl get pods -w
```

Typical states include:

```
ContainerCreating
Running
```

---

# Step 5: Scale the Deployment

Increase the number of replicas to **3 Pods**:

```
kubectl scale deployment web-app --replicas=3
```

Verify the Pods:

```
kubectl get pods
```

Expected result:

```
web-app-xxxxx
web-app-yyyyy
web-app-zzzzz
```

---

# Step 6: Update the Application (Rolling Update)

Update the container image to a newer version:

```
kubectl set image deployment/web-app nginx=nginx:1.24
```

Monitor the update process:

```
kubectl rollout status deployment web-app
```

Kubernetes will:

* Create new Pods
* Gradually terminate old Pods
* Ensure the application remains available

---

# Step 7: Rollback to Previous Version

If the update fails, revert to the previous version:

```
kubectl rollout undo deployment web-app
```

Check rollout history:

```
kubectl rollout history deployment web-app
```

---

# Key Concepts Learned

| Concept        | Description                                    |
| -------------- | ---------------------------------------------- |
| Deployment     | Manages application Pods                       |
| ReplicaSet     | Ensures the desired number of Pods are running |
| Scaling        | Increase or decrease application replicas      |
| Rolling Update | Update application without downtime            |
| Rollback       | Restore a previous working version             |

---

# Architecture

```
Deployment
     │
     ▼
ReplicaSet
     │
     ▼
Pods (Nginx Containers)
```

---

# Conclusion

Using Deployments in Kubernetes allows applications to be:

* Highly available
* Automatically recovered if Pods fail
* Easily scalable
* Safely updated without downtime
