# dsti-devops

## Summary
DevOps project delivering a Task Manager web app with CRUD, MariaDB storage, automated tests, CI, local VM provisioning (Vagrant + Ansible), Docker images, and Kubernetes manifests for Minikube. No public cloud deployment is required.

## Repository Setup

Clone the main repository with submodules:
```bash
git clone --recurse-submodules <main-repository-url>
```
If already cloned:
```bash
git submodule update --init --recursive
```

## Part 1 — Web Application

Simple Task Management app with a backend REST API, a frontend dashboard, and a relational database. Includes health checks and automated tests.

### Architecture Overview
- Backend: Task Manager API (Spring Boot / Java)
- Frontend: Task Dashboard (Flask / Python)
- Database: MariaDB (Docker container)

### Backend — Task Manager API

#### Technology Stack
- Language: Java
- Framework: Spring Boot
- Database: MariaDB (production), H2 (tests)
- Build tool: Maven

#### Features
- REST API with CRUD operations on tasks
- OpenAPI/Swagger documentation
- Automated unit tests

#### API Endpoints
| Method | Endpoint           | Description      |
|-------:|--------------------|------------------|
| POST   | `/api/tasks`       | Create a task    |
| GET    | `/api/tasks`       | List all tasks   |
| GET    | `/api/tasks/{id}`  | Get task by ID   |
| PUT    | `/api/tasks/{id}`  | Update a task    |
| DELETE | `/api/tasks/{id}`  | Delete a task    |
| GET    | `/actuator/health` | Health check     |

#### Testing
- Type: Unit tests
- DB (tests): H2 (in-memory)
- Framework: JUnit + Spring Boot Test
- Coverage: service, API, and repository layers

#### Running Tests (Backend)
From the `task-manager` backend root:
```bash
./mvnw test
```

### Frontend — Task Dashboard

#### Technology Stack
- Language: Python
- Framework: Flask

#### Features
- Web dashboard for managing tasks
- Communicates with the backend REST API
- Displays task title, status, and description

#### Frontend Endpoints
| Method | Endpoint  | Description           |
|-------:|-----------|-----------------------|
| GET    | `/`       | Task dashboard UI     |
| GET    | `/health` | Frontend health check |

## Running the Web Application (Local)

- Backend (Java 17+, Maven):
    ```bash
    ./mvnw spring-boot:run
    ```
    Starts on port 8080 by default.

- Frontend (Python 3.10+, pip):
    ```bash
    export MANAGER_HOST=localhost
    export MANAGER_PORT=8080
    pip install -r requirements.txt
    flask run --port 5000
    ```
    Starts on port 5000 by default.

For a full stack, see Part 4 — Docker Compose.

## Part 2 — CI/CD

GitHub Actions pipelines for both repositories: `task-manager` (backend) and `task-dashboard` (frontend).

### Task Manager (task-manager)
- Branches: develop, main
- On push/pull request:
    - Run unit tests (Maven)
    - Build Docker image (no push)
- On manual trigger (`workflow_dispatch`):
    - Input: tag (e.g., `v1.2.0`)
    - Login with GitHub Secrets
    - Build and push to Docker Hub

### Task Dashboard (task-dashboard)
- Branches: develop, main
- On push/pull request:
    - Docker build only (no tests)
- On manual trigger (`workflow_dispatch`):
    - Input: tag (e.g., `v1.2.0`)
    - Login with GitHub Secrets
    - Build and push to Docker Hub
- Notes:
    - Build on push/PR validates Dockerfile and context
    - Manual publishing gates releases

### Workflow References
- Task Manager:
    https://github.com/<your-org-or-username>/task-manager/tree/main/.github/workflows
- Task Dashboard:
    https://github.com/<your-org-or-username>/task-dashboard/tree/main/.github/workflows

### Docker Hub Images
- Backend: `docker.io/<your-org-or-username>/task-manager:<tag>`
- Frontend: `docker.io/<your-org-or-username>/task-dashboard:<tag>`

## Part 3 — Infrastructure as Code (IaC)

Vagrant provisions a VM; Ansible (local mode) configures runtimes and deploys applications.

### VM Configuration
- Provisioning: Vagrant + Ansible (local)
- Port forwarding:
    | VM Port | Host Port | Application                  |
    |--------:|-----------|------------------------------|
    |    8080 | 8080      | Task Manager API (Backend)   |
    |    5000 | 5000      | Task Dashboard (Frontend)    |

### Running the VM
From the directory with `Vagrantfile` and playbooks:
```bash
vagrant up
```

### Accessing the Applications
- Backend: http://localhost:8080
- Frontend: http://localhost:5000

## Part 4 — Docker Compose

Docker Compose starts the full stack.

### Services
- MariaDB (Database)
- Task Manager (Backend API)
- Task Dashboard (Frontend UI)

### Default Services, Ports, and Environment Variables
| Service        | Description  | Host      | Default Port | Environment Variables                                                                                   | Example Value                                 |
|----------------|--------------|-----------|--------------|----------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| MariaDB        | Database     | localhost | 3306         | `DB_PASSWORD`, `DATABASE_NAME`, `DATABASE_USER`, `DATABASE_PASSWORD`                                    | `rootpass`, `tasks`, `user`, `test`           |
| Task Manager   | Backend API  | localhost | 8080         | `MANAGER_PORT`, `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_NAME`, `DATABASE_USER`, `DATABASE_PASSWORD` | `8080`, `mariadb`, `3306`, `tasks`, `user`, `test` |
| Task Dashboard | Frontend UI  | localhost | 5000         | `DASHBOARD_PORT`, `MANAGER_HOST`, `MANAGER_PORT`                                                        | `5000`, `manager`, `8080`                     |

Ensure ports are free before starting.

### Running the Application

- Default configuration:
    ```bash
    docker-compose up -d
    ```

- Custom configuration via inline environment variables:
    ```bash
    DB_PASSWORD=myrootpass \
    DATABASE_NAME=mytasks \
    DATABASE_USER=myuser \
    DATABASE_PASSWORD=mypass \
    MANAGER_PORT=9090 \
    DASHBOARD_PORT=6000 \
    docker-compose up -d
    ```

- Custom configuration via `.env` file:
    Create `.env.custom`:
    ```bash
    DB_PASSWORD=myrootpass
    DATABASE_NAME=mytasks
    DATABASE_USER=myuser
    DATABASE_PASSWORD=mypass
    MANAGER_PORT=9090
    DASHBOARD_PORT=6000
    ```
    Start with:
    ```bash
    docker-compose --env-file .env.custom up -d
    ```

Got it! Here’s the rewritten Part V with the Minikube cluster startup first, followed by Istio installation, configuration, and application deployment.

## Part V — Container Orchestration using Minikube & Istio

This section describes how to deploy the Task Manager and Task Dashboard applications locally using **Kubernetes (Minikube)** and **Istio**.

---

### 1. Prerequisites

Before deploying the applications, ensure the following tools are installed:

- **Minikube** (local Kubernetes cluster)  
  - Installation guide: [https://minikube.sigs.k8s.io/docs/start/](https://minikube.sigs.k8s.io/docs/start/)  
  - On **Windows**, it is recommended to use **WSL2** for a Linux-like environment.

- **Istioctl** (Istio CLI for installation and management)  
  - Installation guide: [https://istio.io/latest/docs/setup/getting-started/#download](https://istio.io/latest/docs/setup/getting-started/#download)

---

### 2. Start Minikube Cluster

1. Start the Minikube cluster:

```bash
minikube start


Verify the cluster is running:

kubectl get nodes


Make sure Minikube is running successfully before proceeding to Istio installation.

3. Install Istio

Download Istio and add istioctl to your PATH following the Istio installation guide.

Verify the installation:

istioctl version


Install Istio in the cluster (using the demo profile) — applications will be deployed in the default namespace:

istioctl install --set profile=demo -y

4. Enable Istio in Default Namespace

To automatically inject Istio sidecars into pods in the default namespace:

kubectl label namespace default istio-injection=enabled


Verify the namespace label:

kubectl get namespace -L istio-injection

5. Deploy Applications to Kubernetes

Apply the Kubernetes manifests for the backend and frontend applications:

kubectl apply -f k8s/


Check deployments and pods:

kubectl get deployments
kubectl get pods


Check services:

kubectl get svc


Access services locally using Minikube:

minikube service <service-name>


Replace <service-name> with the name of the service you want to open (e.g., task-manager or task-dashboard).

6. Notes

All applications are deployed in the default namespace.

Istio sidecar injection ensures traffic is routed through Istio, enabling observability and service mesh features.

Minikube allows you to test Kubernetes and Istio locally without a cloud provider.

References:

Minikube: https://minikube.sigs.k8s.io/docs/start/

Istio: https://istio.io/latest/docs/setup/getting-started/#download