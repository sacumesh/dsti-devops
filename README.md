# dsti-devops

## Overview
DevOps project delivering a Task Manager web app with CRUD, MariaDB storage, automated tests, CI, local VM provisioning (Vagrant + Ansible), Docker images, and Kubernetes manifests for Minikube. No public cloud deployment is required.

## Repository Setup
Clone with submodules:
```bash
git clone --recurse-submodules <this-repository-URL>
# To get the URL:
git remote get-url origin
```

If already cloned:
```bash
git submodule update --init --recursive
```

## Apps as Submodules
List submodules:
```bash
git submodule status
```

## Part I — Task Management Web Application

Simple Task Management app with a backend REST API, a frontend dashboard, and a relational database. Includes health checks and automated tests.

### Architecture
- Backend: Task Manager API (Spring Boot / Java)
- Frontend: Task Dashboard (Flask / Python)
- Database: MariaDB (Docker container)

---

### Backend — Task Manager API

![Backend — Task Manager API](./assets/backend.png)

#### Technology
- Language: Java
- Framework: Spring Boot
- Database: MariaDB (production), H2 (tests)
- Build tool: Maven

#### Features
- REST API with CRUD operations on tasks
- OpenAPI/Swagger documentation
- Automated unit tests

#### API Endpoints
| Method | Endpoint           | Description   |
|-------:|--------------------|---------------|
| POST   | `/api/tasks`       | Create a task |
| GET    | `/api/tasks`       | List tasks    |
| GET    | `/api/tasks/{id}`  | Get by ID     |
| PUT    | `/api/tasks/{id}`  | Update task   |
| DELETE | `/api/tasks/{id}`  | Delete task   |
| GET    | `/actuator/health` | Health check  |

#### Testing
- Type: Unit tests
- DB (tests): H2 (in-memory)
- Framework: JUnit + Spring Boot Test
- Coverage: service, API, and repository layers

#### Run Tests (Backend)
```bash
# Unix/macOS
./mvnw test

# Windows (PowerShell/CMD)
mvnw.cmd test
```

---

### Frontend — Task Dashboard

![Frontend — Task Dashboard](./assets/frontend.png)

#### Technology
- Language: Python
- Framework: Flask

#### Features
- Web dashboard for managing tasks
- Communicates with the backend REST API
- Displays task title, status, and description

#### Endpoints
| Method | Endpoint  | Description           |
|-------:|-----------|-----------------------|
| GET    | `/`       | Task dashboard UI     |
| GET    | `/health` | Frontend health check |

## Part II — CI/CD Pipelines

GitHub Actions for both repositories: `task-manager` (backend) and `task-dashboard` (frontend).

### task-manager
- Branches: `develop`, `main`
- Triggers:
    - Push or pull request:
        - Run unit tests (Maven)
        - Build Docker image (no push)
    - Manual (`workflow_dispatch`):
        - Input: tag (e.g., `v1.2.0`)
        - Login with GitHub Secrets
        - Build and push to Docker Hub

### task-dashboard
- Branches: `develop`, `main`
- Triggers:
    - Push or pull request:
        - Build Docker image (no tests)
    - Manual (`workflow_dispatch`):
        - Input: tag (e.g., `v1.2.0`)
        - Login with GitHub Secrets
        - Build and push to Docker Hub

### Notes
- Push/PR builds validate Dockerfiles and build contexts
- Manual publishing gates releases

### Workflow References
- Task Manager:
    https://github.com/sacumesh/devops-task-manager/tree/main/.github/workflows
- Task Dashboard:
    https://github.com/sacumesh/devops-task-dashboard/tree/main/.github/workflows

### Docker Hub Images
- Backend: https://hub.docker.com/layers/sacumesh/devops-task-manager/1.0.0
- Frontend: https://hub.docker.com/layers/sacumesh/devops-task-dashboard/1.0.0

## Part III — Infrastructure as Code (IaC)

### Deployed Components
- Java Spring Boot application (Task Manager API — backend)
- Python Flask application (Task Dashboard — frontend)
- MariaDB database (Docker)

Vagrant provisions the VM; Ansible (local mode) configures runtimes and deploys the applications.

### VM Configuration
- Provisioning: Vagrant + Ansible (local)
- Port forwarding:
    | Host Port | VM Port | Service                    |
    |----------:|--------:|----------------------------|
    | 8080      | 8080    | Task Manager API (backend) |
    | 5000      | 5000    | Task Dashboard (frontend)  |

### Run the VM
From the `iac` directory:
```bash
vagrant up
```

### Access the Applications
- Backend: http://localhost:8080
- Frontend: http://localhost:5000

### Cleanup
```bash
# Stop the VM
vagrant halt

# Destroy the VM and remove resources
vagrant destroy -f
```

## Part IV — Docker Compose

Start the full stack with Docker Compose.

### Services
- MariaDB (Database)
- Task Manager (Backend API)
- Task Dashboard (Frontend UI)

### Defaults: Services, Ports, Environment
| Service        | Description  | Host      | Port | Environment Variables                                                                                   | Example Value                                  |
|----------------|--------------|-----------|------|----------------------------------------------------------------------------------------------------------|------------------------------------------------|
| MariaDB        | Database     | localhost | 3306 | `DB_PASSWORD`, `DATABASE_NAME`, `DATABASE_USER`, `DATABASE_PASSWORD`                                    | `rootpass`, `tasks`, `user`, `test`            |
| Task Manager   | Backend API  | localhost | 8080 | `MANAGER_PORT`, `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_NAME`, `DATABASE_USER`, `DATABASE_PASSWORD` | `8080`, `mariadb`, `3306`, `tasks`, `user`, `test` |
| Task Dashboard | Frontend UI  | localhost | 5000 | `DASHBOARD_PORT`, `MANAGER_HOST`, `MANAGER_PORT`                                                        | `5000`, `manager`, `8080`                      |

Ensure ports are free before starting.

### Run
- Default:
    ```bash
    docker compose up -d
    # If using older Docker:
    # docker-compose up -d
    ```

- Access (default):
    - Dashboard: http://localhost:5000
    - Manager API: http://localhost:8080
    - MariaDB: localhost:3306

Sample output:
![Compose up output](./images/part4/image1.png)
![Services running](./images/part4/image2.png)

- Custom (inline env):
    ```bash
    DB_PASSWORD=myrootpass \
    DATABASE_NAME=mytasks \
    DATABASE_USER=myuser \
    DATABASE_PASSWORD=mypass \
    MANAGER_PORT=9090 \
    DASHBOARD_PORT=6000 \
    docker compose up -d
    ```

- Custom (`.env` file):
    Create `.env.custom`:
    ```bash
    DB_PASSWORD=myrootpass
    DATABASE_NAME=mytasks
    DATABASE_USER=myuser
    DATABASE_PASSWORD=mypass
    MANAGER_PORT=9090
    DASHBOARD_PORT=6000
    ```
    Start:
    ```bash
    docker compose --env-file .env.custom up -d
    ```

- Stop:
    ```bash
    docker compose down
    ```

## Part V — Container Orchestration with Minikube and Istio

Deploy Task Dashboard (frontend) and Task Manager (backend) locally on Kubernetes (Minikube) with Istio service mesh.

### Overview
- Components:
    - task-dashboard (v1): frontend/UI
    - task-manager (v1, v2): Spring Boot API
    - mariadb (v1): database
- Flow:
    - Browser → task-dashboard → task-manager Service → mariadb
    - Istio sidecar injection enabled; traffic flows inside the mesh

### Traffic Management (Istio)
- DestinationRule for task-manager with subsets v1 and v2 (`version: v1` / `version: v2`)
- VirtualService for task-manager: 80% to v1, 20% to v2
- No external Ingress/Gateway; use `minikube service --url` for access

### Observability
- task-manager exposes metrics at `/actuator/prometheus`
- Prometheus (Istio demo profile in `istio-system`) scrapes application metrics
- Envoy sidecars emit telemetry for Prometheus and Kiali

### Data Persistence
- mariadb runs as a Deployment/Service
- task-manager connects to mariadb for CRUD
- DB traffic remains within the namespace

### 1) Prerequisites
- Windows users: use WSL2
- Install:
    - kubectl: https://kubernetes.io/docs/tasks/tools/
    - Minikube: https://minikube.sigs.k8s.io/docs/start/
    - istioctl: https://istio.io/latest/docs/setup/getting-started/#download

### 2) Start Minikube
```bash
minikube start
kubectl get nodes
```
Sample output:
![command output](./images/part5/image1.png)

### 3) Install Istio (demo profile)
```bash
istioctl version
istioctl install --set profile=demo -y
```
Sample output:
![command output](./images/part5/image2.png)

Enable sidecar injection:
```bash
kubectl label namespace default istio-injection=enabled
kubectl get namespace -L istio-injection
```
Sample output:
![command output](./images/part5/image3.png)

### 4) Deploy Applications
Apply manifests:
```bash
kubectl apply -k k8s/
```
Sample output:
![command output](./images/part5/image4.png)

Verify:
```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```
Note: Pods have 2 containers (app + Istio sidecar).
Sample output:
![command output](./images/part5/image5.png)

### 5) Configure Istio (traffic rules)
Apply mesh resources:
```bash
kubectl apply -k istio/
```
Sample output:
![command output](./images/part5/image6.png)

Verify:
```bash
kubectl get virtualservice
kubectl get destinationrule
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```
Sample output:
![command output](./images/part5/image7.png)

### 6) Access Services (minikube service --url)
```bash
minikube service <service-name> --url
```
Sample outputs:
![command output](./images/part5/image8.png)
![command output](./images/part5/image9.png)
![command output](./images/part5/image10.png)
![command output](./images/part5/image11.png)

### 7) Observability

#### Prometheus
```bash
minikube service -n istio-system prometheus --url
```
Sample output:
![command output](./images/part5/image17.png)
![command output](./images/part5/image16.png)

#### Kiali
```bash
minikube service -n istio-system kiali --url
```

Get Task Dashboard URL:
```bash
minikube service task-dashboard --url
```

Generate traffic:
```bash
DASHBOARD_URL="<paste the URL>"
while true; do
    echo "$(date) - $(curl -s -o /dev/null -w "%{http_code}" "$DASHBOARD_URL")"
    sleep 1
done
```
Sample output:
![command output](./images/part5/image14.png)
![command output](./images/part5/image12.png)
![command output](./images/part5/image13.png)
