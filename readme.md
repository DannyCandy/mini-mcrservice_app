# Node.js Microservices on Kubernetes

This project demonstrates a microservices architecture using Node.js, deployed on Kubernetes. It consists of three services: User Service, Task Service, and Notification Service, along with MongoDB for data persistence and RabbitMQ for message queuing.

## Architecture

```
                 Client
                   │  HTTP
            ┌──────▼──────┐
            │   Ingress   │  (ingress-nginx controller)
            └──┬───────┬──┘
   /api/users  │       │  /api/tasks
               ▼       ▼
      ┌────────────┐ ┌────────────┐      ┌──────────────────────┐
      │user-service│ │task-service│      │ notification-service │
      │   :3001    │ │   :3002    │      │  (consumer only)     │
      └─────┬──────┘ └──┬──────┬──┘      └──────────▲───────────┘
            │           │      │ publish            │ consume
            │           │      ▼                    │
            │           │   ┌───────────────────────┴┐
            │           │   │ RabbitMQ               │
            │           │   │ queue "task_created"   │
            │           │   └────────────────────────┘
            ▼           ▼
      ┌──────────────────────────┐
      │ MongoDB (service "mongo")│
      │  db: users   db: tasks   │
      └──────────────────────────┘
```

-   **User Service** (`:3001`): Manages user data. Stores users in the `users` database. Does not talk to any other service.
-   **Task Service** (`:3002`): Manages tasks. Stores tasks in the `tasks` database. After a task is saved, it publishes a message `{id, title, description, userId}` to the RabbitMQ queue `task_created`.
-   **Notification Service**: A RabbitMQ consumer only (no HTTP server). It reads messages from `task_created`, logs a notification, and acks the message.
-   **MongoDB**: Shared instance, one database per service (`users`, `tasks`).
-   **RabbitMQ**: Message broker that decouples Task Service from Notification Service. If the notification service is down, messages wait in the queue.

Services find each other through Kubernetes Service DNS names: `mongodb://mongo:27017` and `amqp://rabbitmq`.

## Prerequisites

-   Docker installed and running
-   Kubernetes cluster (Docker Desktop, Minikube, or cloud provider)
-   kubectl configured to access your cluster
-   Docker Hub account (for pushing images)

## Setup

1. Clone this repository:

    ```bash
    git clone <repository-url>
    cd nodejs-microservices-kubernetes
    ```

2. Ensure Docker is running and you're logged into Docker Hub:
    ```bash
    docker login
    ```

## Building Docker Images

Each service folder contains a `Dockerfile` and a `.dockerignore`. The images:

-   use `node:22-alpine` (`mongoose@9` requires Node.js >= 20, so `node:18` does **not** work)
-   install production dependencies only (`npm install --omit=dev`)
-   run as the non-root `node` user

Build and push the images (replace `dannycandy` with your Docker Hub username):

```bash
# Build images
docker build -t dannycandy/nms_user-service:latest ./user-service
docker build -t dannycandy/nms_task-service:latest ./task-service
docker build -t dannycandy/nms_notification-service:latest ./notification-service

# Push images to Docker Hub
docker push dannycandy/nms_user-service:latest
docker push dannycandy/nms_task-service:latest
docker push dannycandy/nms_notification-service:latest
```

If you use a different Docker Hub username, update the `image:` field in `k8s/*-service-deployment.yml` accordingly.

## Deploying to Kubernetes

### 1. Install the Ingress Controller

`k8s/ingress.yml` only defines routing rules. An **Ingress Controller** (an nginx pod) is needed to actually receive traffic and apply those rules. The manifest below creates the `ingress-nginx` namespace, RBAC, the controller Deployment, a `LoadBalancer` Service (mapped to `localhost` on Docker Desktop), the `nginx` IngressClass, and an admission webhook.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml
```

**Wait until the controller is ready** before creating any Ingress (see [Troubleshooting](#troubleshooting) for what happens if you don't):

```bash
kubectl wait -n ingress-nginx --for=condition=ready pod -l app.kubernetes.io/component=controller --timeout=180s
kubectl get svc -n ingress-nginx   # EXTERNAL-IP of ingress-nginx-controller should be "localhost"
```

> **Note:** The ingress-nginx project has been retired (no more releases or security fixes after March 2026). It is fine for local learning; for production, use a [Gateway API](https://gateway-api.sigs.k8s.io/guides/) implementation instead.

### 2. Deploy MongoDB

```bash
kubectl apply -f k8s/mongodb-deployment.yml
kubectl apply -f k8s/mongodb-svc.yml
```

### 3. Deploy RabbitMQ

```bash
kubectl apply -f k8s/rabbitmq-configmap.yml
kubectl apply -f k8s/rabbitmq-statefulset.yml
kubectl apply -f k8s/rabbitmq-service.yml
```

### 4. Deploy Services

```bash
kubectl apply -f k8s/user-service-deployment.yml
kubectl apply -f k8s/user-service-svc.yml

kubectl apply -f k8s/task-service-deployment.yml
kubectl apply -f k8s/task-service-svc.yml

kubectl apply -f k8s/notification-service-deployment.yml
kubectl apply -f k8s/notification-service-svc.yml
```

### 5. Deploy Ingress

```bash
kubectl apply -f k8s/ingress.yml
```

## Verification

Check the status of your deployments:

### Check Pods

```bash
kubectl get pods
```

### Check Services

```bash
kubectl get svc
```

### Check Deployments

```bash
kubectl get deployments
```

### Check Ingress

```bash
kubectl get ingress
```

### Check Ingress Controller (in ingress-nginx namespace)

```bash
kubectl get svc -n ingress-nginx
```

## Logs and Debugging

### View logs for a specific pod

```bash
kubectl logs <pod-name>
```

Example:

```bash
kubectl logs user-service-abc123-def456
```

### View logs for all pods in a deployment

```bash
kubectl logs deployment/user-service
```

### Check pod events

```bash
kubectl describe pod <pod-name>
```

### Port forward for local testing

```bash
kubectl port-forward svc/user-service 3001:3001
```

Then test the API:

```bash
curl http://localhost:3001/api/users
```

## API Endpoints

-   **User Service**: `http://localhost/api/users`

    -   GET `/api/users` - Get all users
    -   POST `/api/users` - Create a new user
    -   PUT `/api/users/:id` - Update a user
    -   DELETE `/api/users/:id` - Delete a user

-   **Task Service**: `http://localhost/api/tasks`

    -   Similar CRUD operations for tasks

-   **Notification Service**: Handles notifications via RabbitMQ

## Scaling

Scale deployments as needed:

```bash
kubectl scale deployment user-service --replicas=5
```

## Troubleshooting

1. **Pods not starting**: Check pod status with `kubectl describe pod <pod-name>`
2. **Services not accessible**: Verify service configuration and pod labels
3. **Ingress not working**: Check ingress controller status and ingress rules
4. **Database connection issues**: Ensure MongoDB pod is running and service is accessible
5. **Image pull errors**: Make sure images are pushed to Docker Hub and accessible

## Cleanup

To remove all resources:

```bash
kubectl delete -f k8s/
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```
