# Kubernetes PODs

## Overview

- In Kubernetes, the goal is to deploy our application in the form of **containers** on **worker nodes** within a **Kubernetes cluster**.
- Kubernetes **does not deploy application containers directly** onto worker nodes.
- Instead, the container is encapsulated into a Kubernetes object called a **Pod**.
- A **Pod** is the **smallest deployable unit** in Kubernetes and represents a **single instance** of an application.
- Pods generally have a **one-to-one** relationship with containers.
- To **scale up**, create additional Pods.
- To **scale down**, delete existing Pods.

> **Note:** It is not recommended to have multiple containers of the **same kind** (e.g., two NGINX containers doing the same job) within a single Pod.

---

## Multi-Container Pods

- You can have **multiple containers** in a single Pod, as long as they **serve different purposes**.
- These are commonly known as **sidecar containers** or **helper containers**.

### Examples of Helper Containers:

- **Data Pullers**: Fetch data required by the main container.
- **Data Pushers**: Collect data (e.g., logs) from the main container and send it elsewhere.
- **Proxies**: Generate static content with a helper container and serve it through the main container.

### Communication

- Containers within the same Pod **share the same network and storage space**, allowing **easy communication** between them.

---

## Managing Pods with `kubectl`

### Create a Pod

```bash
kubectl create pod myfirstpod --image=nginx
# OR
kubectl run my-first-pod --image=stacksimplify/nginx:1.0.0
```

### List Pods

```bash
kubectl get pods
kubectl get po
kubectl get pods -o wide
```

### What Happens in the Background?

1. Kubernetes creates a Pod.
2. It pulls the specified Docker image from DockerHub.
3. A container is created within the Pod.
4. The container is started inside the Pod.

### Describe a Pod

```bash
kubectl describe pod <pod-name>
```

### Delete a Pod

```bash
kubectl delete pod <pod-name>
```

---

## Summary

- Pods are a fundamental concept in Kubernetes, enabling container orchestration and application scalability.
- Understanding Pods and how they operate is critical to deploying and managing applications effectively in Kubernetes.
