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

#### ConfigMap
* Stores non‑sensitive configuration
* Example: database URLs, service names
* Injected into Pods/Services/App as environment variables

#### Secret
* Stores sensitive data (credentials, tokens)
* Base64‑encoded
* Stored inside the K8s cluster (not in source code)

#### Volumes
If database or the Pod of database would be restarted, data would be gone.
To prevent it you use Volumes.

* Provide persistent storage for Pods
* Prevent data loss when Pods restart
* Can be local or cloud‑based
* Similar concept to Docker volumes

---

## Deployment & StatefulSet
What if App/Pod dies?
Instead of relying on just one App Pod and Database Pod, we replicating everything on multiple servers, whole node/application has replicas/clones on differnet node, which is also connected to the same service.

To create replicat you have to define a blueprint for application Pod
and specify how many replicas of that Pod you want

We can't replicate the database using the deployment, we use statefulSet.
 
* **Deployment** – manages stateless applications
* **StatefulSet** – used for stateful apps (e.g. databases)

Best practice:
* Databases often run **outside** the cluster
* Stateless apps scale easily using Deployments

---

## Minikube & kubectl

#### Minikube
* Local Kubernetes cluster for development/testing
* Single‑node cluster (master + worker)
* Runs inside a virtual machine

#### kubectl
* CLI tool for interacting with Kubernetes clusters
* Used to create, update, and inspect resources

```bash
minikube start
```

---

### Basic kubectl Commands

```bash
kubectl get nodes
kubectl get pods
kubectl get services
kubectl logs <pod-name>
kubectl describe pod <pod-name>
kubectl exec -it <pod-name> -- bash
```

---

### Creating a Deployment

```bash
kubectl create deployment NAME --image=image
kubectl create deployment mongo-depl --image=mongo
```

Deployment is an abstraction over Pods — Pods are not created directly.

---

### Debugging Pods
```bash
kubectl logs
kubectl describe pod [pod name]
kubectl exec - it gets the terminal of application container 
kubectl exec -it [pod name] -- bash
```

### To add configuration to Deployment:
first create file ex. config-file.yaml with basic configiration, then you can applay them:
```bash
kubectl apply -f config-file.yaml
```
-f - file

## Deploying MongoDB and Mongo Express 

### 1. Create MongoDB deployment with deployment config created in VS code

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

### 2. Create Secret with credentials for Deployment
* Secrets must be created **before** the Deployment.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque                          //("Opaque" - default for arbitrary key-value pairs)
data:                                 //(the actual contents - in key-value pairs)
  mongo-root-username: dXNlcm5hbWU=   //(values can't be plain text, they need to have base64 encoded)
  mongo-root-password: cGFzc3dvcmQ=
```
```bash
echo -n 'username' | base64    //(you will get base64 encoded username: dXNlcm5hbWU=)
```

---

### 3. Apply mongodb secret config
```bash
kubectl apply -f mongodb-secret.yml
```

### 4. Apply mongodb deployment config
```bash
kubectl apply -f mongodb-deployment-config.yml
```

### 5. Create Internam Service 
* so other components, other pods could talk to
* Provides stable internal access to MongoDB.

#### 5.1 Create Service Config file
* you can put multpile documents in one file separated by "---" deployment and service will be in one file, cause they belong together 

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:              //(to connect to Pod through label)
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017        //(service port)
      targetPort: 27017  //(containerPort of Deployment)
```

#### 5.2 Apply service
```bash
kubectl apply -f mongodb-deployment-config.yml
```
```yaml
NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
service/kubernetes        ClusterIP   10.96.0.1      <none>        443/TCP     9h
service/mongodb-service   ClusterIP   10.109.96.63   <none>        27017/TCP   2s

kubectl describe service mongodb-service
Name:                     mongodb-service
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=mongodb
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.109.96.63
IPs:                      10.109.96.63
Port:                     <unset>  27017/TCP
TargetPort:               27017/TCP
Endpoints:                10.244.0.20:27017      (ip address of Pod, you can check it with kubectl get pod -o wide)
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

---

### 6. Create Mongo Express Deployment & Service & ConfigMap

#### 6.1 Mongo Express Deployment and Service config file
* Which database to connect?
  * MongoDB Address / Internal Service
  * ...MONGODB_SERVER
* Which credentials to authenticate?
  * ...ADMINUSERNAME
  * ...ADMINPASSWORD

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
    valueFrom:                      (You can use ConfigMap to store this value)
      configMapKeyRef:
        name: mongodb-configmap
        key: database_url                            
    - name: ME_CONFIG_MONGODB_ADMINPASSWORD
    valueFrom:
      secretKeyRef:
      name: mongodb-secret 
      key: mongo-root-password
    - name: ME_CONFIG_MONGODB_ADMINUSERNAME
    valueFrom:
      secretKeyRef:
      name: mongodb-secret
      key: mongo-root-username
```

#### 6.2 Create and apply ConfigMap and then apply Mongo Express Deployment

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
data:								              //(the actual contents - in key-value pairs)
  database_url: mongodb-service	  //(it's a name of the service)
```

```bash
kubectl apply -f mongo-configmap.yml
kubectl apply -f mongo-express-deployment-config.yml
```

#### 6.3 Mongo Express SERVICE
* how to make it external service?
  * add type: LoadBalancer    (internal service also acts as LoadBalancer) assigns service an external IP address in nodePort and so accepts external requests
  * add nodePort (port where external IP will be open, port you need to put into browser must be between 30000-32767)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-express-service
spec:
  selector:
  app: mongo-express
  type: LoadBalancer    (ClusterIP is set by deffault)
  ports:
  - protocol: TCP
    port: 8081
    targetPort: 8081
```

```yaml
NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes                ClusterIP      10.96.0.1       <none>        443/TCP          11h
mongodb-express-service   LoadBalancer   10.104.90.156   <pending>     8081:30000/TCP   12s
mongodb-service           ClusterIP      10.109.96.63    <none>        27017/TCP        94m

CLUSTER-IP  - internal ip address
EXTERNAL-IP - external ip address
```

```bash
minikube service mongodb-express-service   (will assign external service IP address)
```
* Assigns an external URL
* Opens Mongo Express in the browser
