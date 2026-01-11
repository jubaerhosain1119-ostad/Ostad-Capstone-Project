# Ostad Capstone Project

## End-to-End CI/CD, Containerization, and Kubernetes Deployment

This project implements a complete DevOps infrastructure for the **finch-frontend** (React) and **finch-backend** (Node.js) applications, covering CI/CD automation, Docker containerization, Kubernetes orchestration, secret management, and comprehensive monitoring.

## Project Structure

```
Ostad-Capstone-Project/
├── finch-frontend/              # React frontend application
│   ├── .github/workflows/       # CI/CD pipeline definitions
│   ├── Dockerfile               # Multi-stage Docker build
│   └── .dockerignore           # Docker build exclusions
├── finch-backend/               # Node.js backend application
│   ├── .github/workflows/       # CI/CD pipeline definitions
│   ├── Dockerfile               # Optimized Docker build
│   └── .dockerignore           # Docker build exclusions
├── kubernetes/                  # Kubernetes manifests
│   ├── frontend-deployment.yaml
│   ├── backend-deployment.yaml
│   ├── redis-deployment.yaml
│   ├── postgresql-deployment.yaml
│   ├── ingress.yaml
│   ├── secrets/                 # Kubernetes secrets
│   └── monitoring/              # Prometheus, Loki, Grafana
├── grafana/
│   └── dashboards/              # Grafana dashboard JSON files
├── docker-compose.yml           # Local development setup
├── CI_CD_Setup.md              # CI/CD documentation
├── Docker_Containerization.md   # Docker documentation
├── Kubernetes_Architecture.md   # K8s architecture & diagram
├── Secret_Management.md        # Secrets documentation
└── Monitoring_Setup.md         # Monitoring documentation
```

## Features

### Phase 1: CI/CD Configuration
- ✅ GitHub Actions workflows for frontend and backend
- ✅ Automated build, test, and Docker image push
- ✅ Code quality checks (ESLint)
- ✅ Unit and integration tests
- ✅ Container registry integration

### Phase 2: Docker Containerization
- ✅ Multi-stage Dockerfiles for optimized images
- ✅ Production-ready container configurations
- ✅ Docker Compose for local development
- ✅ Health checks and security best practices

### Phase 3: Kubernetes Architecture
- ✅ Complete K8s manifests for all services
- ✅ Frontend and backend deployments with scaling
- ✅ PostgreSQL and Redis with persistent storage
- ✅ Ingress configuration for external access
- ✅ Resource limits and health checks

### Phase 4: Secret Management
- ✅ Kubernetes Secrets for sensitive data
- ✅ Database credentials management
- ✅ API keys management
- ✅ Documentation for external secret managers

### Phase 5: Monitoring Stack
- ✅ Prometheus for metrics collection
- ✅ Loki for log aggregation
- ✅ Grafana dashboards for visualization
- ✅ Pre-configured dashboards for applications, K8s, databases, and logs

## Quick Start

### Prerequisites

- Docker and Docker Compose
- Kubernetes cluster (minikube, kind, or cloud)
- kubectl configured
- Docker Hub account (for CI/CD)

### Local Development with Docker Compose

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down
```

### Kubernetes Deployment

1. **Create secrets**:
   ```bash
   # Update secret values in kubernetes/secrets/ first!
   kubectl apply -f kubernetes/secrets/
   ```

2. **Deploy databases**:
   ```bash
   kubectl apply -f kubernetes/postgresql-deployment.yaml
   kubectl apply -f kubernetes/redis-deployment.yaml
   ```

3. **Deploy applications**:
   ```bash
   kubectl apply -f kubernetes/backend-deployment.yaml
   kubectl apply -f kubernetes/frontend-deployment.yaml
   ```

4. **Deploy ingress**:
   ```bash
   kubectl apply -f kubernetes/ingress.yaml
   ```

5. **Deploy monitoring** (optional):
   ```bash
   kubectl apply -f kubernetes/monitoring/
   ```

### Accessing Services

**Local Development (Docker Compose)**:
- Frontend: http://localhost:5173
- Backend API: http://localhost:3000
- PostgreSQL: localhost:5432
- Redis: localhost:6379

**Kubernetes (with minikube)**:
```bash
# Enable ingress
minikube addons enable ingress

# Get ingress IP
minikube ip

# Add to /etc/hosts
echo "$(minikube ip) finch.local" | sudo tee -a /etc/hosts

# Access application
curl http://finch.local
```

**Monitoring (if deployed)**:
```bash
# Grafana
kubectl port-forward svc/grafana-service 3000:3000
# Access at http://localhost:3000 (admin/admin)

# Prometheus
kubectl port-forward svc/prometheus-service 9090:9090
# Access at http://localhost:9090
```

## Documentation

- **[CI/CD Setup](CI_CD_Setup.md)**: GitHub Actions pipeline configuration
- **[Docker Containerization](Docker_Containerization.md)**: Dockerfile optimization and best practices
- **[Kubernetes Architecture](Kubernetes_Architecture.md)**: K8s deployment architecture with diagrams
- **[Secret Management](Secret_Management.md)**: Secure secret handling strategies
- **[Monitoring Setup](Monitoring_Setup.md)**: Prometheus, Loki, and Grafana configuration

## CI/CD Configuration

### Required GitHub Secrets

Configure these secrets in your GitHub repository:

- `DOCKER_USERNAME`: Your Docker Hub username
- `DOCKER_PASSWORD`: Your Docker Hub password or access token

### Pipeline Triggers

- Push to `main` or `master` branch
- Pull requests to `main` or `master` branch
- Only triggers when application files change

## Important Notes

### Before Deployment

1. **Update Docker image names** in Kubernetes manifests:
   - Replace `your-docker-username` with your actual Docker Hub username
   - Files: `kubernetes/frontend-deployment.yaml`, `kubernetes/backend-deployment.yaml`

2. **Update secret values**:
   - Encode your actual secrets in base64
   - Update files in `kubernetes/secrets/`
   - See [Secret Management](Secret_Management.md) for details

3. **Configure storage classes**:
   - Update `storageClassName` in PVC manifests if needed
   - Default is `standard` (works with minikube)

4. **Update ingress host**:
   - Change `finch.local` in `kubernetes/ingress.yaml` if needed
   - Configure DNS or /etc/hosts accordingly

## Technology Stack

- **Frontend**: React 18, Vite
- **Backend**: Node.js 18, Express
- **Databases**: PostgreSQL 15, Redis 7
- **Containerization**: Docker, Docker Compose
- **Orchestration**: Kubernetes
- **CI/CD**: GitHub Actions
- **Monitoring**: Prometheus, Loki, Grafana

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Ensure CI/CD pipelines pass
5. Submit a pull request

## License

This project is part of the Ostad Capstone Project.

## Support

For issues and questions:
1. Check the relevant documentation files
2. Review Kubernetes and Docker logs
3. Verify configuration files
4. Check GitHub Actions workflow logs

---

**Project Status**: ✅ Complete - All phases implemented and documented
