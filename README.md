# Kubernetes – Practical Notes & Example Minikube Setup

This README summarizes core Kubernetes concepts and a hands-on example using **MongoDB** and **Mongo Express** deployed on **Minikube**.

---

#### What is Kubernetes
* Open‑source container orchestration platform
* Manages containerized applications across different environments
* Handles deployment, scaling, networking, and self‑healing

#### Kubernetes Architecture
* **Master Node (Control Plane)** – manages the cluster (at least one)
* **Worker Nodes** – run application workloads, are connected to Master node
* Each worker node runs **kubelet** (node ↔ cluster communication)
* Each worker has containers of different applications deployed on it

#### Master Node Processes
* **API Server** (container) – entry point to the Kubernates cluster (UI, CLI, REST API)
* **Controller Manager** (container) – keeps track of what is happening in the cluster
* **Scheduler** (container) – schedule containers on different nodes based on workload/available serve resources
* **etcd** (storage) – distributed key‑value store holding cluster state

#### Virtual Network
* Connects all nodes into a single logical network
* Allows Pods and Services to communicate as if on one machine

---

## Core Kubernetes Components

#### Node (Worker Node)
* Physical or virtual machine
* Hosts Pods and application containers

#### Pod
* Smallest deployable unit in Kubernetes
* Abstraction over one or more containers
* Usually runs **one application container**
* Has its own internal IP address
* Pods are **ephemeral** – they can die very easliy with container crush and then be recreated with a new IP

Because Pods can die and change IPs, Kubernetes uses **Services**.

---

#### Service
* Provides a **stable IP and DNS name** for Pods
* Enables reliable Pod‑to‑Pod communicatio
* If Pod dies the Service will stay

**Types:**

* **ClusterIP** – internal service (default)
* **LoadBalancer / NodePort** – external service

#### Ingress
Request goes first to Ingrees which forward them to the Service, what is soultion to External Service problem with public address.
* Routes external HTTP/HTTPS traffic to Services

---

## ConfigMap & Secret

### ConfigMap
* Stores non‑sensitive configuration
* Example: database URLs, service names
* Injected into Pods as environment variables

### Secret
* Stores sensitive data (credentials, tokens)
* Base64‑encoded
* Stored inside the cluster (not in source code)

---

## Volumes
* Provide persistent storage for Pods
* Prevent data loss when Pods restart
* Can be local or cloud‑based
* Similar concept to Docker volumes

---

## Deployment & StatefulSet

* **Deployment** – manages stateless applications
* **StatefulSet** – used for stateful apps (e.g. databases)

Best practice:

* Databases often run **outside** the cluster
* Stateless apps scale easily using Deployments

---

## Minikube & kubectl

### Minikube

* Local Kubernetes cluster for development/testing
* Single‑node cluster (control plane + worker)
* Runs inside a virtual machine

### kubectl

* CLI tool for interacting with Kubernetes clusters
* Used to create, update, and inspect resources

```bash
minikube start
```

---

## Basic kubectl Commands

```bash
kubectl get nodes
kubectl get pods
kubectl get services
kubectl logs <pod-name>
kubectl describe pod <pod-name>
kubectl exec -it <pod-name> -- bash
```

---

## Creating a Deployment

```bash
kubectl create deployment nginx-depl --image=nginx
kubectl create deployment mongo-depl --image=mongo
```

Deployment is an abstraction over Pods — Pods are not created directly.

---

## MongoDB Deployment Example

### 1. MongoDB Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: mongo-root-username
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: mongo-root-password
```

---

### 2. MongoDB Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  mongo-root-username: dXNlcm5hbWU=
  mongo-root-password: cGFzc3dvcmQ=
```

Secrets must be created **before** the Deployment.

---

### 3. MongoDB Internal Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
```

Provides stable internal access to MongoDB.

---

## Mongo Express Setup

### 1. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
data:
  database_url: mongodb-service
```

---

### 2. Mongo Express Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express-deployment
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
        - name: mongo-express
          image: mongo-express
          ports:
            - containerPort: 8081
          env:
            - name: ME_CONFIG_MONGODB_SERVER
              valueFrom:
                configMapKeyRef:
                  name: mongodb-configmap
                  key: database_url
            - name: ME_CONFIG_MONGODB_ADMINUSERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: mongo-root-username
            - name: ME_CONFIG_MONGODB_ADMINPASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: mongo-root-password
```

---

### 3. Mongo Express External Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-express-service
spec:
  selector:
    app: mongo-express
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
```

---

## Accessing Mongo Express (Minikube)

```bash
minikube service mongodb-express-service
```

* Assigns an external URL
* Opens Mongo Express in the browser

---

## Key Takeaways

* Pods are ephemeral — Services provide stability
* ConfigMaps and Secrets externalize configuration
* Deployments manage stateless apps
* Minikube is ideal for local Kubernetes learning

---

## Status

✔ MongoDB running as internal service
✔ Mongo Express exposed externally
✔ Configuration handled via ConfigMaps and Secrets
