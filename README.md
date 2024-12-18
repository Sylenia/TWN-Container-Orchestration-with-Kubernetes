# Deploy MongoDB and Mongo Express in a Local Kubernetes Cluster with Minikube

This guide provides step-by-step instructions to deploy MongoDB and Mongo Express in a local Kubernetes cluster using Minikube. It also incorporates best practices to ensure a secure and efficient deployment.

---

## Prerequisites

1. **Install Minikube:** Follow the official [Minikube installation guide](https://minikube.sigs.k8s.io/docs/start/).
2. **Install kubectl:** Ensure `kubectl` is installed and configured. Instructions can be found [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
3. **Install Docker:** Docker is required to build and push container images. Install Docker from [here](https://www.docker.com/products/docker-desktop/).
4. **Verify Setup:**
    ```bash
    minikube start
    kubectl version --client
    docker --version
    ```

---

## Steps

### 1. Start Minikube

Start a local Minikube cluster:
```bash
minikube start
```

Verify the cluster is running:
```bash
kubectl cluster-info
```

### 2. Enable the Minikube Docker Environment

Use Minikube’s Docker environment to build and use images locally:
```bash
minikube docker-env
```
Run the following command to configure your terminal session:
```bash
eval $(minikube -p minikube docker-env)
```

### 3. Create Kubernetes Resources

#### a. Create a Namespace

To isolate resources, create a namespace:
```bash
kubectl create namespace demo-apps
```

#### b. Create a Secret for MongoDB Credentials

Store MongoDB credentials securely using a Kubernetes Secret:
```bash
kubectl create secret generic mongodb-secret \
  --namespace=demo-apps \
  --from-literal=mongo-root-username=admin \
  --from-literal=mongo-root-password=securepassword
```
> **Recommendation:** Use strong passwords and avoid hardcoding them in YAML files.

#### c. Create a ConfigMap for MongoDB Configuration

Define MongoDB’s configuration:
```bash
kubectl create configmap mongodb-config \
  --namespace=demo-apps \
  --from-literal=mongo-database=mydb
```

#### d. Create Deployment and Service YAML Files

Create the following YAML files:

**`mongodb-deployment.yaml`**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  namespace: demo-apps
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
        image: mongo:5.0
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

**`mongodb-service.yaml`**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: demo-apps
spec:
  ports:
  - port: 27017
    targetPort: 27017
  selector:
    app: mongodb
  type: ClusterIP
```

**`mongoexpress-deployment.yaml`**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
  namespace: demo-apps
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
        image: mongo-express:latest
        ports:
        - containerPort: 8081
        env:
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
        - name: ME_CONFIG_MONGODB_SERVER
          value: mongodb
```

**`mongoexpress-service.yaml`**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-express
  namespace: demo-apps
spec:
  ports:
  - port: 8081
    targetPort: 8081
  selector:
    app: mongo-express
  type: NodePort
```

### 4. Apply the Configurations

Deploy the resources to the cluster:
```bash
kubectl apply -f mongodb-deployment.yaml
kubectl apply -f mongodb-service.yaml
kubectl apply -f mongoexpress-deployment.yaml
kubectl apply -f mongoexpress-service.yaml
```

### 5. Verify Deployments and Services

Check the status of your deployments:
```bash
kubectl get deployments -n demo-apps
```

Check the services:
```bash
kubectl get services -n demo-apps
```

### 6. Access Mongo Express

Retrieve the NodePort for the `mongo-express` service:
```bash
kubectl get service mongo-express -n demo-apps
```
Use the Minikube IP and the NodePort to access the Mongo Express interface in your browser:
```bash
minikube service mongo-express -n demo-apps
```

### 7. Namespaces

Namespaces are a critical organizational tool in Kubernetes, allowing you to isolate and manage resources effectively. They are particularly useful for:

- **Environment Isolation:** Separate development, testing, and production environments within the same cluster.
- **Resource Management:** Apply resource quotas and access control at the namespace level.
- **Multi-Team Collaboration:** Enable multiple teams to work on the same cluster without interference.

To add a namespace, use:
```bash
kubectl create namespace <namespace-name>
```

#### Automate Namespace Selection

To ensure all operations occur within a specific namespace, set the namespace context permanently using:
```bash
kubectl config set-context --current --namespace=<namespace-name>
```

Alternatively, you can use the `kubens` tool for quick namespace switching:
```bash
kubens <namespace-name>
```

> **Best Practice:** Always use namespaces for better organization and security, especially in multi-tenant or multi-environment setups.

### 8. Services

Kubernetes Services are used to expose applications running on a set of Pods. They provide networking capabilities to Pods and enable access both within and outside the cluster. Here are the main types of Services, their use cases, and example configurations:

#### a. ClusterIP Service

- **Use Case:** Default service type. Exposes the service on a cluster-internal IP, making it accessible only within the cluster.
- **Example Configuration:**
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: example-clusterip
      namespace: demo-apps
    spec:
      selector:
        app: example-app
      ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
      type: ClusterIP
    ```

#### b. Headless Service

- **Use Case:** Used for Stateful applications or when you need direct access to individual Pods.
- **Example Configuration:**
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: example-headless
      namespace: demo-apps
    spec:
      clusterIP: None
      selector:
        app: example-app
      ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
    ```

#### c. NodePort Service

- **Use Case:** Exposes the service on each Node's IP at a static port. Used for local testing or when external access is needed.
- **Example Configuration:**
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: example-nodeport
      namespace: demo-apps
    spec:
      selector:
        app: example-app
      ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
        nodePort: 30007
      type: NodePort
    ```

#### d. LoadBalancer Service

- **Use Case:** Exposes the service externally using a cloud provider's load balancer. Ideal for production.
- **Example Configuration:**
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: example-loadbalancer
      namespace: demo-apps
    spec:
      selector:
        app: example-app
      ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
      type: LoadBalancer
    ```

> **Best Practice:** Use `ClusterIP` for internal services, `NodePort` or `LoadBalancer` for external access, and `Headless` for StatefulSets or direct Pod communication.

### 9. Ingress

Ingress in Kubernetes provides HTTP and HTTPS routing to services within the cluster. It acts as an entry point that routes external traffic to specific services based on defined rules.

#### Use Cases
- **Domain-Based Routing:** Direct traffic to different services using hostnames.
- **Path-Based Routing:** Route requests to different services based on URL paths.
- **SSL Termination:** Terminate SSL at the ingress level to offload certificates.
- **Centralized Entry Point:** Manage multiple services behind a single load balancer.

#### Example Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kubernetes-dashboard
spec:
  rules:
  - host: dashboard.com
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: kubernetes-dashboard
              port:
                number: 80
```

#### Controllers
To use Ingress, an ingress controller must be installed. Common options include:
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Traefik](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
- [HAProxy](https://haproxy-ingress.github.io/)

> **Best Practice:** Choose an ingress controller based on your environment and traffic requirements. Always use SSL for secure connections.

#### Enabling Ingress in Minikube

To enable Ingress in your Minikube environment, use the following command:
```bash
minikube addons enable ingress
```

Verify that the Ingress addon is enabled:
```bash
kubectl get pods -n kube-system
```
You should see pods related to the ingress controller running in the `kube-system` namespace.

#### Enabling the Minikube Dashboard

Minikube provides a built-in Kubernetes Dashboard for monitoring resources. To enable and access it, use the following command:
```bash
minikube dashboard
```
This command will open the dashboard in your default web browser.

> **Tip:** Use the dashboard to visualize the status of your deployments, services, and ingress rules for easier debugging and management.

### 10. Volumes

Volumes in Kubernetes are essential for data persistence, enabling stateful applications, and managing storage across containers and nodes. They abstract the storage layer, ensuring data is available and consistent across application lifecycles.

#### Levels of Volume Abstraction
Volumes in Kubernetes operate at different abstraction levels:
- **Pods:** Temporary volumes tied to a Pod’s lifecycle.
- **Persistent Volumes (PVs):** Storage resources independent of Pods, managed at the cluster level.
- **Persistent Volume Claims (PVCs):** Requests for storage by users.
- **Storage Classes:** Define the types of storage and provisioning mechanisms.

---

#### Persistent Volumes

Persistent Volumes (PVs) are cluster-level storage resources. They provide a way to abstract physical storage for use by applications.

##### Types of Persistent Volumes

**a. Local Volumes**
- **Description:** Storage directly tied to a specific node.
- **Use Case:** High-performance applications requiring node-local storage.
- **Example Configuration:**
  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: local-pv
  spec:
    capacity:
      storage: 10Gi
    accessModes:
      - ReadWriteOnce
    hostPath:
      path: /mnt/local-storage
  ```
- **Benefits:**
  - Low latency.
  - Easy setup.
- **Drawbacks:**
  - Node-specific; not portable.

**b. Remote Volumes**
- **Description:** Storage accessible across the cluster.
- **Use Case:** Distributed systems or applications requiring storage redundancy.
- **Example Configuration:**
  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: remote-pv
  spec:
    capacity:
      storage: 50Gi
    accessModes:
      - ReadWriteMany
    nfs:
      path: /var/nfs
      server: 192.168.1.1
  ```
- **Benefits:**
  - Cluster-wide availability.
  - Flexible scaling.
- **Drawbacks:**
  - Higher latency.

---

#### Persistent Volume Claims

PVCs allow users to request storage resources in a declarative way. They bind to available PVs that meet the specified criteria.

##### Example Configuration
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

##### Use Cases
- Requesting storage for application Pods.
- Abstracting storage details from users.

---

#### Storage Classes

Storage Classes define the provisioning mechanism for dynamic volume creation. They abstract storage backend details, enabling dynamic provisioning.

##### Example Configuration
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "10"
```

##### Use Cases
- Dynamic provisioning for stateful workloads.
- Managing multiple storage backends.

---

### Summary
- **Persistent Volumes:** Provide cluster-level storage.
- **Persistent Volume Claims:** Allow applications to request storage resources.
- **Storage Classes:** Enable dynamic provisioning and backend management.

---

## Best Practices

1. **Use Namespaces:** Always use namespaces to isolate resources.
2. **Secure Secrets:** Use Kubernetes Secrets for sensitive data and avoid hardcoding credentials.
3. **Monitor Resources:** Use `kubectl describe` and `kubectl logs` to debug and monitor.
4. **Resource Limits:** Set resource requests and limits for better resource management.

---

Happy Deploying! 🎉
```