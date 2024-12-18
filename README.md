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

---

## Best Practices

1. **Use Namespaces:** Always use namespaces to isolate resources.
2. **Secure Secrets:** Use Kubernetes Secrets for sensitive data and avoid hardcoding credentials.
3. **Monitor Resources:** Use `kubectl describe` and `kubectl logs` to debug and monitor.
4. **Resource Limits:** Set resource requests and limits for better resource management.

---

Happy Deploying! 🎉
```