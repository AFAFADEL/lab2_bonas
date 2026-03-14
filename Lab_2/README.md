# Lab 2: Exposing Your Deployment and Basic Troubleshooting

## Scenario
The development team wants to test the web-app Deployment from inside the cluster. You need to expose it as a service and then intentionally break it to practice your troubleshooting skills on a non-responsive application.

## Lab Description
The student will expose a Deployment using a Service and then practice Deployment troubleshooting by introducing common failures and diagnosing them.

## Tasks to Implement

1. Use the `web-app` Deployment from Lab 1 (with 3 replicas).

2. Expose the Deployment using a ClusterIP service named `web-service` that targets port 80. Use an imperative command to create the service.

3. Verify the service endpoints correctly list all 3 pod IPs.

4. **Troubleshooting Task 1:** One of the pods becomes unresponsive. Simulate this by `exec`-ing into one pod and stopping Nginx (e.g., `kill 1` or `nginx -s stop`). Observe the Pod Lifecycle and note what the Deployment does. Check the service endpoints again.

5. **Troubleshooting Task 2:** Introduce a configuration error. Edit the Deployment to use a non-existent image tag: `nginx:broken`. Watch the rollout status and the new pods that get stuck.

6. Use `kubectl rollout status`, `kubectl describe pod <broken-pod>`, and `kubectl logs <broken-pod>` to diagnose the image pull failure. Then, fix the image tag to `nginx:latest` and verify the rollout succeeds.
---------------
# Lab 2: Exposing Your Deployment and Basic Troubleshooting

## Overview

In this lab, we expose the **web-app Deployment** created in Lab 1 using a **ClusterIP Service** so it can be accessed inside the Kubernetes cluster.
Then we simulate common failures to practice **troubleshooting techniques**.

The application used is **Nginx** running in multiple Pods managed by a Deployment.

---

# Prerequisites

* Kubernetes cluster running
* `kubectl` configured
* Lab 1 completed (Deployment **web-app** running with **3 replicas**)

Check the deployment:

```
kubectl get deployment
```

Expected output:

```
NAME      READY   UP-TO-DATE   AVAILABLE
web-app   3/3     3            3
```

Check Pods:

```
kubectl get pods
```

---

# Step 1: Expose the Deployment as a Service

Create a **ClusterIP Service** using an imperative command:

```
kubectl expose deployment web-app \
--name=web-service \
--type=ClusterIP \
--port=80 \
--target-port=80
```

Verify the Service:

```
kubectl get svc
```

Example output:

```
NAME          TYPE        CLUSTER-IP       PORT
web-service   ClusterIP   10.104.45.201    80/TCP
```

---

# Step 2: Verify Service Endpoints

Check the endpoints that the service routes traffic to:

```
kubectl get endpoints web-service
```

Example:

```
NAME          ENDPOINTS
web-service   10.244.0.5:80,10.244.0.6:80,10.244.0.7:80
```

These IPs correspond to the **3 running Pods**.

You can also inspect the service:

```
kubectl describe service web-service
```

---

# Troubleshooting Task 1: Simulate an Unresponsive Pod

List the Pods:

```
kubectl get pods
```

Exec into one Pod:

```
kubectl exec -it <pod-name> -- bash
```

Stop Nginx inside the container:

```
nginx -s stop
```

Or terminate the main process:

```
kill 1
```

Exit the container:

```
exit
```

Watch the Pod lifecycle:

```
kubectl get pods -w
```

You will observe that the failed Pod is terminated and **a new Pod is created automatically**.

This happens because the **Deployment ensures the desired number of replicas are always running**.

Check endpoints again:

```
kubectl get endpoints web-service
```

You should see the **new Pod IP replacing the old one**.

---

# Troubleshooting Task 2: Introduce a Configuration Error

Edit the Deployment:

```
kubectl edit deployment web-app
```

Change the image to a non-existent tag:

```
image: nginx:broken
```

Save and exit.

---

# Step 3: Observe the Failed Rollout

Check rollout status:

```
kubectl rollout status deployment web-app
```

List Pods:

```
kubectl get pods
```

You will likely see:

```
ImagePullBackOff
```

---

# Step 4: Diagnose the Problem

Describe the failing Pod:

```
kubectl describe pod <broken-pod>
```

Look for errors like:

```
Failed to pull image "nginx:broken"
```

Check container logs:

```
kubectl logs <broken-pod>
```

---

# Step 5: Fix the Image

Correct the image:

```
kubectl set image deployment/web-app nginx=nginx:latest
```

Monitor the rollout:

```
kubectl rollout status deployment web-app
```

Verify Pods are running:

```
kubectl get pods
```

Expected result:

```
web-app-xxxxx   Running
web-app-yyyyy   Running
web-app-zzzzz   Running
```

---

# Key Concepts Learned

| Concept          | Description                                      |
| ---------------- | ------------------------------------------------ |
| Service          | Provides stable access to Pods                   |
| ClusterIP        | Internal service accessible inside the cluster   |
| Endpoints        | List of Pod IPs serving the service              |
| Self-Healing     | Deployment recreates failed Pods                 |
| ImagePullBackOff | Occurs when the container image cannot be pulled |
| Troubleshooting  | Diagnose issues using kubectl commands           |

---

# Useful Troubleshooting Commands

```
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- bash
kubectl rollout status deployment web-app
```

---

# Conclusion

This lab demonstrates how Kubernetes:

* Exposes applications through Services
* Automatically replaces failed Pods
* Helps diagnose and fix deployment issues

These troubleshooting skills are essential for managing applications in Kubernetes environments.
