# Production-Ready FastAPI Application

This repository contains a containerized FastAPI application designed for a production-like deployment on Kubernetes. It includes a full CI/CD pipeline, a complete monitoring stack (Prometheus & Grafana), and robust security configurations (RBAC, Pod Security Standards).

## Project Structure

```
.
├── .github/workflows/            
│   └── ci-cd.yaml
├── main.py                         
├── Dockerfile                      
├── requirements.txt               
├── requirements-dev.txt            
├── .gitignore                     
├── Runbook.md                      
├── k8s/                           
│   ├── configmap.yaml             
│   ├── deployment.yaml             
│   ├── hpa.yaml                   
│   ├── ingress.yaml               
│   ├── networkpolicy.yaml          
│   ├── secret.yaml                 
│   ├── service.yaml                
│   └── serviceaccount.yaml        
├── monitoring/                     
│   ├── alertmanager-config.yaml    
│   ├── alertmanager-deployment.yaml 
│   ├── alerts.yaml                 
│   ├── grafana-deployment.yaml            
│   ├── node-exporter.yaml         
│   ├── prometheus-configmap.yaml   
│   ├── prometheus-deployment.yaml 
├── scripts/                        
│   ├── deploy.sh                  
│   ├── healthcheck.sh              
│   └── rollback.sh                
├── security/                       
│   ├── pod-security.yaml          
│   └── rbac.yaml   
├── tests/                        
│   ├── init.py         
│   └── test_main.py               
└── README.md                     
```

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- `kubectl`
- A Docker Hub account

## Setup and Deployment

### 1. Start Local Kubernetes Cluster
Start Minikube to get your local Kubernetes environment running.
```bash
minikube start
```

### 2. Create Namespaces
The application and its monitoring stack are deployed in separate namespaces for better organization and security.
```bash
kubectl create namespace fastapi-app
kubectl create namespace monitoring
```

### 3. Deploy the Application
The `deploy.sh` script applies all necessary Kubernetes manifests and can deploy a specific version if a Docker image tag is provided.

**Deploy the `:latest` version:**
```bash
./scripts/deploy.sh
```

**Deploy a specific version (using commit SHA):**
```bash
./scripts/deploy.sh <commit-sha-tag>
```

### 4. CI/CD Pipeline (`ci-cd.yaml`)

The CI/CD pipeline is configured to run on every push to the `main` branch for changes within the .github/workflows directory, the application code(main.py), the requirements.txt, the Dockerfile, and the tests folder. 

**Pipeline Stages:**
1.  **Test:** Runs `pytest` for unit tests.
2.  **Build & Push:** Builds the Docker image and pushes it to Docker Hub with two tags: `:latest` and the commit SHA.

**Note:** The pipeline requires `DOCKER_USERNAME` and `DOCKER_PASSWORD` to be configured as secrets in the GitHub repository settings.

## Management Scripts

### Health Check
The `healthcheck.sh` script checks the status of the deployment and verifies that the application is accessible via its Ingress endpoint.
```bash
./scripts/healthcheck.sh
```

### Rollback
The `rollback.sh` script performs a rollback to the previous deployment version.
```bash
./scripts/rollback.sh
```

## Monitoring

The monitoring stack includes Prometheus for metrics collection and Grafana for visualization.

### 1. Deploy the Monitoring Stack
```bash
kubectl apply -f monitoring/
```

### 2. Access Grafana
Forward the Grafana port to your local machine:
```bash
kubectl port-forward -n monitoring svc/grafana-service 3000:3000
```
- **URL:** `http://localhost:3000`
- **Username:** `admin`
- **Password:** `admin123` (retrieved from the `grafana-secret`)

### 3. Access Prometheus
Forward the Prometheus port to your local machine:
```bash
kubectl port-forward -n monitoring svc/prometheus-service 9091:9090
```
- **URL:** `http://localhost:9090`

### 4. Add Prometheus as a Grafana Data Source
- **URL:** `http://prometheus-service.monitoring.svc.cluster.local:9090`
- Navigate to Configuration -> Data Sources -> Add Prometheus.
- Use the URL http://prometheus-service.monitoring.svc.cluster.local:9090 and click "Save & test".

## Security

- **RBAC:** A dedicated `ServiceAccount` (`fastapi-service-account`) is used for the application. It is bound to a `Role` with zero permissions, following the principle of least privilege.
- **Pod Security:** The `fastapi-app` namespace is enforced with the `restricted` Pod Security Standard, requiring pods to follow current pod hardening best practices and preventing them from running with privileged settings.
- **Network Policies:** Network policies are in place to control ingress and egress traffic to the application pods.
- **Secrets:** Application secrets (like API keys) and configuration are externalized using Kubernetes `Secrets` and `ConfigMaps`.



Perfect — your workflow is clean, modular, and already uses good DevSecOps practices 👏
Below is a **ready-to-use `README.md`** that clearly documents your GitHub Actions CI/CD pipeline — explaining the structure, each stage’s purpose, and how to run it manually or automatically.

---


##  Continuous Integration (CI) Pipeline — *FastAPI Microservice*

This repository is a GitHub Actions–based CI/CD pipeline that automates the testing, building, and security scanning of a FastAPI microservice before deployment to container registries or Kubernetes environments.

The pipeline enforces **code quality**, ensures **test coverage**, and integrates **vulnerability scanning** to maintain security and compliance standards.

---

## ⚙️ Pipeline Structure

The workflow file is defined in:

```
.github/workflows/ci-cd.yaml
```

It consists of **two main jobs** that run sequentially:

1.  **Test and Linting**
2.  **Build and Push Docker Image**

Below is a breakdown of each stage:

---

### **Job 1 — Test and Linting**

| Step                     | Purpose                                                                                                                                                |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Checkout code**        | Clones the repository into the GitHub Actions runner so that all subsequent steps can access the code.                                                 |
| **Set up Python**        | Installs Python 3.11 on the runner environment.                                                                                                        |
| **Install dependencies** | Installs development dependencies listed in `requirements-dev.txt` (e.g., `pytest`, `flake8`, etc.).                                                   |
| **Run lint checks**      | Executes **flake8** to check for PEP8/code style issues. <br>• Non-blocking: the build continues even if warnings are found.                           |
| **Run tests**            | Runs the project’s test suite using **pytest** with code coverage tracking. Generates a coverage report in XML format for visibility and integrations. |

**Purpose:**
This stage ensures that the codebase adheres to style guidelines, runs without syntax errors, and passes all tests before containerization.
Only when this job passes does the next stage execute.

---

### **Job 2 — Build and Push Docker Image**

| Step                             | Purpose                                                                                                                                                                                                              |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Checkout code**                | Fetches the repository again for the build stage.                                                                                                                                                                    |
| **Set up Docker Buildx**         | Enables advanced Docker build capabilities like multi-platform builds and caching.                                                                                                                                   |
| **Log in to Docker Hub**         | Authenticates with Docker Hub using secrets stored in the repository (`DOCKER_USERNAME` and `DOCKER_PASSWORD`).                                                                                                      |
| **Build and push Docker image**  | Builds the application image from the `Dockerfile` and pushes it to Docker Hub with two tags:<br>• `latest` – for the newest stable build<br>• `${{ github.sha }}` – for traceability of specific commits            |
| **Run Trivy vulnerability scan** | Scans the built Docker image using **Trivy** for OS and library vulnerabilities. <br>Findings are reported but **do not fail the pipeline** (`exit-code: 0`), allowing builds to continue while highlighting issues. |

**Purpose:**
This stage automates container image creation and security verification, ensuring that every pushed image is tested, versioned, and scanned for vulnerabilities.

---

## Secrets and Environment Variables

To make the pipeline work, configure the following secrets in your GitHub repository:

| Secret            | Description                           |
| ----------------- | ------------------------------------- |
| `DOCKER_USERNAME` | Your Docker Hub username              |
| `DOCKER_PASSWORD` | Docker Hub access token (or password) |

Go to:
**Repository → Settings → Secrets and variables → Actions → New repository secret**

---

## Workflow Triggers

The pipeline can be triggered in two ways:

1. **Manual Trigger**

   * The workflow uses `workflow_dispatch`, allowing you to start a run manually from the **Actions** tab on GitHub.

2. *(Optional)* **Automatic Trigger**

   * You can uncomment the `push` and `pull_request` sections in the YAML file to run the pipeline automatically on code updates to specific branches (e.g., `master`).

Example:

```yaml
on:
  push:
    branches: [ master ]
```

---

## ▶️ How to Run the Pipeline

1. **Trigger the workflow manually**

   * Go to **GitHub → Actions → CI Pipeline → Run workflow**
   * Select the desired branch and click **Run workflow**

2. **Monitor progress**

   * Each job (Test/Lint → Build/Push) will run in sequence.
   * View logs for every step directly in the **Actions** console.

3. **Check the Docker image**

   * Once successful, the built image will appear in Docker Hub:

     ```
     docker pull emmaxyz/fastapi-service:latest
     docker pull emmaxyz/fastapi-service:<commit_sha>
     ```

---

## 📊 Pipeline Summary

```
flowchart TD
    A[Checkout Code] --> B[Set Up Python 3.11]
    B --> C[Install Dependencies]
    C --> D[Run Flake8 (Linting)]
    D --> E[Run Pytest (Tests & Coverage)]
    E --> F[Build Docker Image]
    F --> G[Push Image to Docker Hub]
    G --> H[Trivy Vulnerability Scan]
```



