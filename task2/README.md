# Task 2: FastAPI + MySQL on Kubernetes

## Overview

In this task, I deployed a FastAPI Guestbook application connected to a MySQL database on a Kubernetes cluster using Minikube.

The application uses Kubernetes ConfigMaps and Secrets for configuration and credentials. MySQL stores its data using Persistent Volumes and Persistent Volume Claims. Users access the application through an NGINX Ingress Controller.

## Architecture

```text
                    Browser
                       |
                       v
                NGINX Ingress
                       |
                       v
                webapp-ingress
                       |
                       v
                webapp-api Service
                    Port 80
                       |
             ---------------------
             |                   |
             v                   v
        FastAPI Pod 1       FastAPI Pod 2
             |                   |
             ---------------------
                       |
                       v
                 MySQL Service
                   Port 3306
                       |
                       v
                  MySQL Pod
                       |
                       v
                 MySQL PVC
                       |
                       v
                  MySQL PV
```

## Technologies Used

- Kubernetes
- Minikube
- Docker
- FastAPI
- MySQL 8.4
- ConfigMap
- Secrets
- Persistent Volume (PV)
- Persistent Volume Claim (PVC)
- Deployments and Services
- NGINX Ingress Controller

## Project Structure

```text
Task-2/
│
├── webapp/
│   ├── Dockerfile
│   └── application files
│
├── mysql/
│   ├── Dockerfile
│   ├── conf/
│   │   └── mysql.cnf
│   ├── initdb/
│   │   └── 01-setup.sh
│   └── schema/
│       └── schema.sql
│
├── namespace.yaml
├── secret.yaml
├── configmap.yaml
├── storage.yaml
├── mysql.yaml
├── webapp.yaml
├── ingress.yaml
└── README.md
```

## Implementation Steps

### 1. Build Docker Images

Build the FastAPI and MySQL images:

```bash
docker build -t webapp-api:task2 ./webapp

docker build -t webapp-mysql:task2 ./mysql
```

Load both images into Minikube:

```bash
minikube image load webapp-api:task2

minikube image load webapp-mysql:task2
```

Verify the images:

```bash
minikube image ls
```

### 2. Create Kubernetes Namespace

Create the namespace:

```bash
kubectl apply -f namespace.yaml
```

Verify:

```bash
kubectl get namespaces
```

Namespace: `webapp-stack`

### 3. Create Secret

The Secret stores MySQL credentials:

| Key | Value |
|---|---|
| root-password | rootpass123 |
| app-user | webapp |
| app-password | webapp123 |

Apply:

```bash
kubectl apply -f secret.yaml
```

Verify:

```bash
kubectl get secret mysql-secret -n webapp-stack
```

### 4. Create ConfigMap

The ConfigMap stores application configuration:

| Key | Value |
|---|---|
| APP_TITLE | hello in MLOPS MENA COMMUNITY |
| APP_ENV | dev |
| MYSQL_HOST | mysql |
| MYSQL_PORT | 3306 |
| MYSQL_DATABASE | webapp |

Apply:

```bash
kubectl apply -f configmap.yaml
```

Verify:

```bash
kubectl get configmap webapp-config -n webapp-stack
```

### 5. Create Persistent Volume and Persistent Volume Claim

Created a Persistent Volume with:

- Name: `mysql-pv`
- Storage: 2Gi
- Access Mode: ReadWriteOnce
- Storage Class: manual
- Host Path: `/data/webapp-mysql`
- Reclaim Policy: Retain

The PV is pinned to the worker node `minikube-m02` using node affinity.

Created a PVC named `mysql-pvc` in the `webapp-stack` namespace.

Apply:

```bash
kubectl apply -f storage.yaml
```

Verify:

```bash
kubectl get pv

kubectl get pvc -n webapp-stack
```

Expected status: `Bound`.

### 6. Deploy MySQL

Created a MySQL Deployment with:

- One replica.
- Image: `webapp-mysql:task2`
- Image Pull Policy: `IfNotPresent`
- Credentials from Kubernetes Secrets.
- Database configuration from ConfigMap.
- Persistent storage mounted at `/var/lib/mysql`.
- Readiness and Liveness probes.
- Recreate deployment strategy.

Created a ClusterIP Service named `mysql` on port 3306.

Apply:

```bash
kubectl apply -f mysql.yaml
```

Check the Pod:

```bash
kubectl get pods -n webapp-stack
```

Check MySQL logs:

```bash
kubectl logs deploy/mysql -n webapp-stack
```

Check the Service:

```bash
kubectl get svc -n webapp-stack
```

### 7. Deploy FastAPI Application

Created a FastAPI Deployment with:

- Two replicas.
- Image: `webapp-api:task2`
- Container port: 8000.
- Configuration loaded from ConfigMap using `envFrom`.
- MySQL username and password loaded from Secrets.
- Readiness and Liveness HTTP probes using `/health`.

Created a ClusterIP Service:

- Name: `webapp-api`
- Service port: 80
- Target port: 8000

Apply:

```bash
kubectl apply -f webapp.yaml
```

Verify:

```bash
kubectl get deployments -n webapp-stack

kubectl get pods -n webapp-stack

kubectl get endpoints webapp-api -n webapp-stack
```

The application should have two ready replicas.

### 8. Enable NGINX Ingress Controller

Enable the Minikube Ingress addon:

```bash
minikube addons enable ingress
```

Verify:

```bash
kubectl get pods -n ingress-nginx

kubectl get ingressclass
```

### 9. Create Ingress

Created an Ingress resource named `webapp-ingress`.

It routes incoming requests from `/` to the `webapp-api` Service on port 80.

Apply:

```bash
kubectl apply -f ingress.yaml
```

Verify:

```bash
kubectl get ingress -n webapp-stack
```

### 10. Access the Application

Start port forwarding to the Ingress Controller:

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
```

Open the application:

```text
http://localhost:8080
```

The application displays:

- Application Pod name.
- Image version.
- Environment configuration.
- MySQL server information.
- Guestbook messages.

## Database Initialization

The MySQL Docker image includes:

- `initdb/01-setup.sh`: Creates the database and application user.
- `schema/schema.sql`: Creates the `messages` table and inserts default messages.

These initialization scripts run only when MySQL starts with an empty data directory.

**Important:** During this task, the existing Persistent Volume already contained MySQL data, so the initialization scripts were not executed again.

To resolve this, the application database and user were created using MySQL root credentials, and the schema was applied manually without deleting the Persistent Volume:

```bash
kubectl exec -i -n webapp-stack deploy/mysql -- sh -c \
'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" "$APP_DB_NAME"' \
< mysql/schema/schema.sql
```

The database was then verified, and the application successfully read and wrote messages.

## Verification and Testing

### 1. Verify Docker Images

```bash
minikube image ls
```

Expected:
- `webapp-api:task2`
- `webapp-mysql:task2`

### 2. Verify Kubernetes Resources

```bash
kubectl get all -n webapp-stack
```

### 3. Verify PV and PVC

```bash
kubectl get pv

kubectl get pvc -n webapp-stack
```

Expected: Both resources are `Bound`.

### 4. Verify Application Configuration

```bash
kubectl exec -n webapp-stack deploy/webapp-api -- env
```

Verified application environment variables from ConfigMap and Secrets.

### 5. Verify API Information

```bash
curl http://localhost:8080/api/info
```

Verified:
- Application version: `task2`
- Application title: `hello in MLOPS MENA COMMUNITY`
- Environment: `dev`
- MySQL connection information.

### 6. Write a Message (POST)

```bash
curl -X POST http://localhost:8080/api/messages \
-H "Content-Type: application/json" \
-d '{"author":"student","content":"hello"}'
```

Expected response:

```json
{
  "id": 3,
  "written_to": "mysql-..."
}
```

The message was successfully inserted into MySQL.

### 7. Read Messages (GET)

```bash
curl http://localhost:8080/api/messages
```

Expected messages:

| ID | Author | Content |
|---|---|---|
| 3 | student | hello |
| 2 | k8s-labs | This message was created when the database was initialized. |
| 1 | admin | Welcome to the WebApp guestbook! |

### 8. Test Data Persistence

To verify that database data survives a MySQL Pod restart, delete the MySQL Pod:

```bash
kubectl delete pod -n webapp-stack -l app=mysql
```

Wait for the new Pod to become Ready:

```bash
kubectl wait --for=condition=Ready pod \
-l app=mysql \
-n webapp-stack \
--timeout=180s
```

Check the Pods:

```bash
kubectl get pods -n webapp-stack
```

Read the messages again:

```bash
curl http://localhost:8080/api/messages
```

**Result:** The message `student: hello` was still available after the MySQL Pod was deleted and recreated.

The MySQL Pod name changed, but the stored messages remained unchanged.

This confirms that the Persistent Volume successfully preserves database data independently of the Pod lifecycle.

## Final Results

All main task requirements were successfully implemented and tested.

| Component | Result |
|---|---|
| Docker Images | Passed |
| Kubernetes Namespace | Passed |
| ConfigMap and Secrets | Passed |
| Persistent Volume and PVC | Passed |
| MySQL Deployment and Service | Passed |
| FastAPI Deployment (2 replicas) | Passed |
| Readiness and Liveness Probes | Passed |
| NGINX Ingress | Passed |
| API POST and GET | Passed |
| MySQL Pod Restart | Passed |
| Data Persistence | Passed |

## Screenshots

Add screenshots here to document the implementation and verification:

<img width="1486" height="1007" alt="1" src="https://github.com/user-attachments/assets/6734d957-0d9d-419a-8769-af31327806cb" />

<img width="1161" height="1001" alt="2" src="https://github.com/user-attachments/assets/00d908cd-016a-4bbe-8049-b1632dbaa6ef" />

<img width="1030" height="407" alt="3" src="https://github.com/user-attachments/assets/2a843b23-ff55-4c3c-afb2-9a654b505f15" />

<img width="1052" height="288" alt="4" src="https://github.com/user-attachments/assets/503e4c53-f713-4124-ab48-59f7dad0ba09" />

<img width="1888" height="807" alt="5" src="https://github.com/user-attachments/assets/4d3f1d2c-06f6-4f21-9eb2-415b0444db24" />

<img width="1110" height="341" alt="6" src="https://github.com/user-attachments/assets/c44d67f1-e30e-4863-8fe3-183ad1106e49" />

<img width="1413" height="401" alt="7" src="https://github.com/user-attachments/assets/b36b3530-b2c7-4eb2-bbe8-b31bc6853763" />

<img width="1907" height="995" alt="8" src="https://github.com/user-attachments/assets/81210077-e980-48c6-be3c-fefbcadb4757" />

<img width="1883" height="775" alt="9" src="https://github.com/user-attachments/assets/44b786aa-9cf0-4f91-bf53-4e5bb58e1008" />


## Conclusion

In this task, I deployed a FastAPI application connected to MySQL on a multi-node Kubernetes cluster using Minikube.

I practiced Kubernetes configuration management, Secrets, persistent storage, Deployments, Services, health probes, and Ingress routing.

The final persistence test confirmed that database data remains available even after the MySQL Pod is deleted and recreated.

