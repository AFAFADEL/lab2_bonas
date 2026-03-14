# Lab 4: Deployment Strategies and Storage Troubleshooting

## Scenario
The staging environment requires a stateful logging application that must retain logs across pod restarts but not across pod recreation. Additionally, you need to test a blue-green deployment strategy for a critical update to the logging application.

## Lab Description
The student will create a Deployment with `emptyDir` storage, test its behavior, and then perform a blue-green deployment using labels and services. They will troubleshoot a storage-related failure.

## Prerequisites
- Minikube or Kubernetes cluster running
- `staging` namespace created (`kubectl create namespace staging`)

## Container Images and Details

| Component | Image | Purpose |
|-----------|-------|---------|
| logger v1 | `busybox:latest` | Writes timestamp to file every 10 seconds |
| logger v2 | `busybox:latest` | Writes timestamp + hostname to file every 10 seconds |

## Tasks to Implement

### Part 1: Initial Logger Deployment with emptyDir

Create the initial logger Deployment in the `staging` namespace:

- **Name:** `logger`
- **Replicas:** 1
- **Container image:** `busybox:latest`
- **Container command:**
  ```bash
  /bin/sh -c "while true; do date >> /log/output.txt; sleep 10; done"
  ```
- **Add label:** `app: logger`, `version: v1`
- **Add an `emptyDir` volume mounted at `/log`**

### Part 2: Test Storage Behavior

1. **Simulate a container crash:**
   - Exec into the pod and kill the main process (PID 1)
   - Observe what happens to the pod
   - Check if the log file persists after container restart

2. **Delete the pod manually:**
   - Delete the specific pod running the logger
   - Wait for the Deployment to recreate it
   - Check if the old logs are present in the new pod

### Part 3: Blue-Green Deployment Setup

1. **Create a service for the logger:**
   - **Name:** `logger-service`
   - **Type:** ClusterIP
   - **Port:** 80 (target port doesn't matter since this is a logging app, but we need a service for the exercise)
   - **Selector:** `version: v1`

2. **Prepare the new version (v2) Deployment:**
   - **Name:** `logger-v2`
   - **Replicas:** 2 (to ensure at least one works during troubleshooting)
   - **Container image:** `busybox:latest`
   - **Container command (includes hostname):**
     ```bash
     /bin/sh -c "while true; do echo \"\$(date) - Host: \$(hostname)\" >> /log/output.txt; sleep 10; done"
     ```
    - **Add label:** `app: logger`, `version: v2`
    - **INTENTIONAL BUG:** Mount the `emptyDir` volume at `/var/log/app` instead of `/log`

### Part 4: Blue-Green Switch and Troubleshooting

1. **Perform the blue-green switch:**
   - Update the `logger-service` selector to `version: v2`
   - Observe that one of the `logger-v2` pods enters `CrashLoopBackOff`

2. **Troubleshoot and fix the issue:**
   - Diagnose why the pod is crashing
   - Fix the volume mount path in the `logger-v2` Deployment
   - Verify both v2 pods are running successfully
   - Confirm the service is now sending traffic to v2 pods

### Part 5: Verification

- **Verify the fix:**
  - Check logs from a v2 pod to confirm timestamps and hostnames are being written
  - Exec into a v2 pod and verify the log file exists and contains data
  - Test service discovery by running a temporary busybox pod and curling the service (optional)

---

# Required Deliverables:
- Students must push the following to their public GitHub repository:

## README Content Requirements

The README must contain the following sections:

| Section | Required Content |
|---------|------------------|
| Lab Summary | Brief overview of what was accomplished in this lab |
| Storage Behavior Explanation | Clear explanation of what happens to `emptyDir` data when:<br>- A container crashes and restarts<br>- A pod is deleted and recreated by the Deployment |
| Blue-Green Deployment Steps | Step-by-step process followed to perform the blue-green switch |
| Troubleshooting Section | Description of the `logger-v2` pod failure:<br>- What caused the `CrashLoopBackOff`<br>- How it was diagnosed (commands used)<br>- How it was fixed |
| Commands Used | Complete list of all `kubectl` commands executed during the lab |

## Manifests Directory Requirements

| Path | What to Include |
|------|-----------------|
| `manifests/v1/logger-deployment.yaml` | Original v1 Deployment with `emptyDir` volume mounted at `/log` |
| `manifests/v2/logger-deployment.yaml` | Fixed v2 Deployment (after troubleshooting) with correct volume mount |
| `manifests/service/logger-service.yaml` | Service manifest showing both v1 and v2 selector states (or final version with v2 selector) |
------------------
# Lab 4: Deployment Strategies and Storage Troubleshooting

## Lab Summary

In this lab we implemented a logging application in Kubernetes using a **Deployment with emptyDir storage**.
We tested how the storage behaves during **container restarts vs pod recreation**.

Then we implemented a **Blue-Green deployment strategy** to update the logging application from **version v1 to v2** using labels and services.
During the update we intentionally introduced a configuration bug, diagnosed the failure, and fixed it.

---

# Storage Behavior Explanation

## 1️⃣ Container Crash and Restart

When the container inside a pod crashes:

* Kubernetes restarts the container automatically.
* The **pod itself is not deleted**.
* The **emptyDir volume remains attached to the pod**.

Result:

```
Log file remains available after container restart.
```

This means the data written inside `/log` is preserved.

---

## 2️⃣ Pod Deletion and Recreation

When the pod itself is deleted:

* The Deployment creates a **new pod**.
* A **new emptyDir volume is created**.
* The previous data is lost.

Result:

```
All previous logs are removed because emptyDir exists only for the lifetime of the pod.
```

---

# Part 1: Logger Deployment (v1)

Create directory structure:

```
manifests/
 ├── v1/
 ├── v2/
 └── service/
```

Create file:

```
manifests/v1/logger-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logger
  namespace: staging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: logger
      version: v1
  template:
    metadata:
      labels:
        app: logger
        version: v1
    spec:
      containers:
      - name: logger
        image: busybox:latest
        command:
        - /bin/sh
        - -c
        - while true; do date >> /log/output.txt; sleep 10; done
        volumeMounts:
        - name: log-volume
          mountPath: /log
      volumes:
      - name: log-volume
        emptyDir: {}
```

Apply deployment:

```
kubectl apply -f manifests/v1/logger-deployment.yaml
```

---

# Part 2: Testing Storage Behavior

Get pod name:

```
kubectl get pods -n staging
```

Exec into pod:

```
kubectl exec -it <pod-name> -n staging -- sh
```

Kill the main process:

```
kill 1
```

Observe restart:

```
kubectl get pods -n staging
```

Check logs again:

```
cat /log/output.txt
```

The file still exists.

---

## Delete Pod

Delete the pod:

```
kubectl delete pod <pod-name> -n staging
```

Wait for Deployment to recreate it.

Check logs again:

```
kubectl exec -it <new-pod> -n staging -- cat /log/output.txt
```

Result:

```
Old logs are gone because a new emptyDir was created.
```

---

# Part 3: Create Logger Service

Create file:

```
manifests/service/logger-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: logger-service
  namespace: staging
spec:
  type: ClusterIP
  selector:
    version: v1
  ports:
  - port: 80
    targetPort: 80
```

Apply service:

```
kubectl apply -f manifests/service/logger-service.yaml
```

---

# Part 4: Logger v2 Deployment (With Intentional Bug)

Create file:

```
manifests/v2/logger-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logger-v2
  namespace: staging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: logger
      version: v2
  template:
    metadata:
      labels:
        app: logger
        version: v2
    spec:
      containers:
      - name: logger
        image: busybox:latest
        command:
        - /bin/sh
        - -c
        - while true; do echo "$(date) - Host: $(hostname)" >> /log/output.txt; sleep 10; done
        volumeMounts:
        - name: log-volume
          mountPath: /var/log/app
      volumes:
      - name: log-volume
        emptyDir: {}
```

Apply deployment:

```
kubectl apply -f manifests/v2/logger-deployment.yaml
```

---

# Part 5: Blue-Green Switch

Update the service to point to v2.

Edit the service:

```
kubectl edit service logger-service -n staging
```

Change selector:

```
version: v2
```

---

# Troubleshooting Section

## Problem

One of the `logger-v2` pods enters:

```
CrashLoopBackOff
```

---

## Cause

The container writes logs to:

```
/log/output.txt
```

But the volume is mounted at:

```
/var/log/app
```

This means the `/log` directory does not exist.

---

## Diagnosis Commands

Check pod status:

```
kubectl get pods -n staging
```

Describe pod:

```
kubectl describe pod <pod-name> -n staging
```

Check container logs:

```
kubectl logs <pod-name> -n staging
```

---

## Fix

Edit the v2 deployment and correct the mount path:

```
mountPath: /log
```

Correct version:

```yaml
volumeMounts:
- name: log-volume
  mountPath: /log
```

Apply again:

```
kubectl apply -f manifests/v2/logger-deployment.yaml
```

---

# Verification

Check pods:

```
kubectl get pods -n staging
```

Check logs:

```
kubectl logs <v2-pod-name> -n staging
```

Expected output:

```
2026-03-14 - Host: logger-v2-pod
```

Exec into pod:

```
kubectl exec -it <pod-name> -n staging -- sh
```

Verify log file:

```
cat /log/output.txt
```

---

# Commands Used

```
kubectl create namespace staging
kubectl apply -f <file>
kubectl get pods -n staging
kubectl exec -it <pod> -- sh
kubectl delete pod <pod>
kubectl describe pod <pod>
kubectl logs <pod>
kubectl edit service logger-service
```

---

# Conclusion

This lab demonstrates:

* How **emptyDir storage behaves in Kubernetes**
* How **Deployments recreate pods automatically**
* How to perform **Blue-Green deployments using labels and services**
* How to **diagnose and fix container crashes**

These concepts are essential for managing **stateful workloads and production deployments** in Kubernetes.

