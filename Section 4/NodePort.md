# Kubernetes Services

We can expose an application running on a set of Pods using different types of services available in Kubernetes:

- **ClusterIP** – Exposes the service only within the cluster.
- **NodePort** – Exposes the service on each Node's IP at a static port (external access).
- **LoadBalancer** – Provisions an external load balancer (typically in cloud environments).

---

## NodePort Service

- Allows access to the application outside of the Kubernetes cluster.
- Exposes the service on each worker node’s IP at a static port (called NodePort).
- A ClusterIP service, to which the NodePort service routes, is automatically created.
- NodePort range: `30000 - 32767`.

**Access format:**

```
https://<WorkerNodeIP>:<NodePort>
```

### Port Definitions

| Term         | Description                                                   |
|--------------|---------------------------------------------------------------|
| NodePort     | External port on the worker node exposed to the outside world |
| port         | Port used internally by the service inside the cluster         |
| targetPort   | The container port where the application is running            |

---

## Steps to Create and Expose a Pod using NodePort

### 1. Create a Pod
```bash
kubectl run my-first-pod --image=stacksimplify/kubenginx:1.0.0
```

### 2. Expose the Pod as a NodePort Service
```bash
kubectl expose pod my-first-pod --type=NodePort --port=80 --name=my-first-service
```

### 3. Get Service Info
```bash
kubectl get service
kubectl get svc
```

### 4. Get Worker Node Public IP
```bash
kubectl get nodes -o wide
```

### 5. Access the Application
```bash
https://<WorkerNodeIP>:<NodePort>
```

---

## Notes on `targetPort`

- If `targetPort` is not defined, it defaults to the value of `port`.

### Example (Incorrect)
This will fail if the container port is `80` but the service port is `81` and no `targetPort` is specified:

```bash
kubectl expose pod my-first-pod --type=NodePort --port=81 --name=my-first-service
```

### Corrected Example
Define the `targetPort` explicitly:

```bash
kubectl expose pod my-first-pod --type=NodePort --port=81 --target-port=80 --name=my-first-service
```
