# CCS3308 – Virtualization and Containers  
## Lab 6: Kubernetes Fundamentals with Minikube

**Student ID:** CIT-24-01-0110 
**Module:** CCS3308 – Virtualization and Containers  
**Topic:** Container Orchestration & Kubernetes  
**Platform:** Kubernetes + Minikube + Docker  

---

# 1. Introduction

This laboratory demonstrates the transition from Docker container management to Kubernetes container orchestration.

Using **Minikube**, a local single-node Kubernetes cluster was created to deploy, manage, scale, update, and troubleshoot a multi-container application.

The application architecture consists of:

- Frontend Tier → Nginx
- API Tier → HTTPBin REST API
- Cache Tier → Redis
- Database Tier → PostgreSQL with Persistent Storage

The following Kubernetes concepts were implemented:

- Pods
- Deployments
- Services
- StatefulSets
- PersistentVolumeClaims
- Scaling
- Self-healing
- Rolling Updates
- Rollbacks
- Troubleshooting

---

# 2. Environment Setup

## Prerequisites

The following tools were installed:

- Docker
- kubectl
- Minikube

## Start Minikube Cluster

Command used:

```bash
minikube start --driver=docker
```

Verify kubectl installation:

```bash
kubectl version --client
```

Check Kubernetes node:

```bash
kubectl get nodes
```

Expected result:

```
NAME       STATUS   ROLES           AGE
minikube   Ready    control-plane   xx
```

---

# 3. Explore Kubernetes Cluster Architecture

## Commands Used

Check cluster information:

```bash
kubectl cluster-info
```

Display node details:

```bash
kubectl get nodes -o wide
```

View Kubernetes system pods:

```bash
kubectl get pods -n kube-system
```

## Task 1.1 Screenshot

Screenshot of Kubernetes system components:

```
screenshots/task1.1.png
```

## Kubernetes Components Identified

| Pod | Component | Category |
|---|---|---|
| kube-apiserver | API Server | Control Plane |
| kube-controller-manager | Controller Manager | Control Plane |
| kube-scheduler | Scheduler | Control Plane |
| etcd | Cluster Database | Control Plane |
| kube-proxy | Network Proxy | Worker Node |

## Explanation

The control plane manages the Kubernetes cluster and maintains the desired state of applications.

Worker nodes run the actual application workloads through Pods and containers.

---

# 4. Creating the First Pod

A simple frontend Pod was created using the nginx image.

Manifest:

```
k8s/pod-frontend.yaml
```

Apply the Pod:

```bash
kubectl apply -f k8s/pod-frontend.yaml
```

Check Pod status:

```bash
kubectl get pods
```

View Pod information:

```bash
kubectl describe pod frontend
```

View logs:

```bash
kubectl logs frontend
```

Access nginx:

```bash
kubectl port-forward pod/frontend 8080:80
```

Open browser:

```
http://localhost:8080
```

## Task 2.1 Screenshot

```
screenshots/task2.1.png
```

---

# 5. Pod Recreation and Ephemeral Behaviour

The Pod was deleted:

```bash
kubectl delete pod frontend
```

The Pod was recreated:

```bash
kubectl apply -f k8s/pod-frontend.yaml
```

Check Pod IP:

```bash
kubectl get pods -o wide
```

## Observation

The Pod IP address changed after recreation.

This happened because Pods are ephemeral resources. When a Pod is deleted, Kubernetes creates a new Pod instance with a new IP address.

---

# 6. Deployment and Self-Healing

A Deployment was created with three frontend replicas.

Manifest:

```
k8s/deployment-frontend.yaml
```

Apply Deployment:

```bash
kubectl apply -f k8s/deployment-frontend.yaml
```

Check Deployment:

```bash
kubectl get deployments
```

Check Pods:

```bash
kubectl get pods -o wide
```

---

## Self-Healing Test

A Pod was manually deleted:

```bash
kubectl delete pod <pod-name>
```

Monitor replacement:

```bash
kubectl get pods -w
```

## Task 3.1 Screenshot

```
screenshots/task3.1.png
```

## Explanation

Kubernetes followed the control-loop model:

1. Desired state required three frontend Pods.
2. Controller continuously monitored the cluster.
3. Actual state changed after deleting one Pod.
4. Kubernetes detected the difference.
5. The Deployment controller created a replacement Pod.

---

# 7. Scaling the Deployment

Scale frontend replicas to five:

```bash
kubectl scale deployment frontend --replicas=5
```

Check Pods:

```bash
kubectl get pods
```

Scale down to two replicas:

```bash
kubectl scale deployment frontend --replicas=2
```

## Task 4.1 Screenshot

```
screenshots/task4.1.png
```

## Explanation

The frontend can be scaled independently because Kubernetes manages each service separately. The database tier does not need to be changed when increasing frontend replicas.

---

# 8. Exposing Application Using Service

A NodePort Service was created.

Manifest:

```
k8s/service-frontend.yaml
```

Apply Service:

```bash
kubectl apply -f k8s/service-frontend.yaml
```

View Services:

```bash
kubectl get services
```

Get application URL:

```bash
minikube service frontend --url
```

Open the generated URL in the browser.

## Task 5.1 Screenshot

```
screenshots/task5.1.png
```

## Explanation

Port forwarding provides temporary direct access to a Pod.

A Service provides a permanent networking endpoint and automatically routes traffic to available Pods even when Pods are replaced.

---

# 9. Rolling Updates and Rollbacks

Update nginx version:

```bash
kubectl set image deployment/frontend frontend=nginx:1.27-alpine
```

Check rollout:

```bash
kubectl rollout status deployment/frontend
```

Rollback:

```bash
kubectl rollout undo deployment/frontend
```

Verify rollback:

```bash
kubectl rollout status deployment/frontend
```

## Task 6.1 Screenshot

```
screenshots/task6.1.png
```

## Explanation

Docker Compose does not provide Kubernetes-level features such as automatic rolling updates, version tracking, and rollback management.

---

# 10. Deploying Full Multi-Tier Application

Application architecture:

```
Frontend
    |
    |
API
    |
    |
Cache
    |
    |
Database
```

---

# API Tier

Image:

```
kennethreitz/httpbin
```

Files:

```
api-deployment.yaml
api-service.yaml
```

Deploy:

```bash
kubectl apply -f k8s/api-deployment.yaml

kubectl apply -f k8s/api-service.yaml
```

---

# Cache Tier

Image:

```
redis:7-alpine
```

Files:

```
cache-deployment.yaml
cache-service.yaml
```

Deploy:

```bash
kubectl apply -f k8s/cache-deployment.yaml

kubectl apply -f k8s/cache-service.yaml
```

---

# Database Tier

Image:

```
postgres:16-alpine
```

Implemented using:

- StatefulSet
- PersistentVolumeClaim
- Headless Service

Files:

```
postgres-statefulset.yaml
postgres-service.yaml
```

Deploy:

```bash
kubectl apply -f k8s/postgres-statefulset.yaml

kubectl apply -f k8s/postgres-service.yaml
```

---

View all resources:

```bash
kubectl get all
```

## Task 7.1 Screenshot

```
screenshots/task7.1.png
```

---

# 11. Internal Connectivity Testing

Create debug Pod:

```bash
kubectl run debug --rm -it --image=busybox -- sh
```

Inside the debug container:

Test API:

```bash
wget -qO- http://api-service
```

Test Redis DNS:

```bash
nslookup cache-service
```

## Task 7.2 Screenshot

```
screenshots/task7.2.png
```

---

# 12. Database Persistence Verification

Connect to PostgreSQL:

```bash
kubectl exec -it postgres-0 -- psql -U postgres
```

Create test table:

```sql
CREATE TABLE demo (
id serial primary key,
note text
);

INSERT INTO demo(note)
VALUES ('lab6 test row');
```

Check data:

```sql
SELECT * FROM demo;
```

Delete PostgreSQL Pod:

```bash
kubectl delete pod postgres-0
```

Wait for recreation:

```bash
kubectl get pods -w
```

Check data again:

```sql
SELECT * FROM demo;
```

## Task 8.1 Screenshot

```
screenshots/task8.1.png
```

## Explanation

The data remained because PostgreSQL was deployed using StatefulSet with PersistentVolumeClaim.

If PostgreSQL was deployed as a normal Deployment without persistent storage, the database data would be lost after Pod deletion.

---

# 13. Monitoring and Troubleshooting

Enable metrics server:

```bash
minikube addons enable metrics-server
```

View Pod resources:

```bash
kubectl top pods
```

View node resources:

```bash
kubectl top nodes
```

---

# Broken Pod Testing

A broken Pod was created using an invalid nginx image tag.

Apply:

```bash
kubectl apply -f broken-pod.yaml
```

Check status:

```bash
kubectl get pods
```

View details:

```bash
kubectl describe pod broken-pod
```

## Task 9.1 Screenshot

```
screenshots/task9.1.png
```

## Explanation

The Pod showed an image pull failure because Kubernetes could not find the requested Docker image.

---

# 14. Cleanup

Delete Kubernetes resources:

```bash
kubectl delete -f k8s/
```

Verify:

```bash
kubectl get all
```

Stop Minikube:

```bash
minikube stop
```

## Task 10.1 Screenshot

```
screenshots/task10.1.png
```

---

# 15. Repository Structure

```
lab6/

│── README.md
│── answers.md
│
├── k8s/
│   ├── pod-frontend.yaml
│   ├── deployment-frontend.yaml
│   ├── service-frontend.yaml
│   ├── api-deployment.yaml
│   ├── api-service.yaml
│   ├── cache-deployment.yaml
│   ├── cache-service.yaml
│   ├── postgres-statefulset.yaml
│   └── postgres-service.yaml
│
└── screenshots/
    ├── task1.1.png
    ├── task2.1.png
    ├── task3.1.png
    ├── task4.1.png
    ├── task5.1.png
    ├── task6.1.png
    ├── task7.1.png
    ├── task7.2.png
    ├── task8.1.png
    ├── task9.1.png
    └── task10.1.png
```

---

# Conclusion

This laboratory successfully demonstrated Kubernetes container orchestration using Minikube.

The implementation showed:

- Container deployment using Pods
- Application management using Deployments
- Networking using Services
- Automatic recovery through self-healing
- Application scaling
- Rolling updates and rollback
- Persistent database storage using StatefulSet and PVC
- Kubernetes troubleshooting methods

Kubernetes provides advanced container management features that are not available in Docker Compose alone.
