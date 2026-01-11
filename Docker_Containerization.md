# Docker Containerization Documentation

## Overview

This document explains the Docker containerization strategy for the finch-frontend and finch-backend applications. Both applications are containerized using optimized multi-stage Dockerfiles following Docker best practices.

## Frontend Dockerfile

**Location**: `finch-frontend/Dockerfile`

### Architecture

The frontend Dockerfile uses a **multi-stage build** approach with two stages:

1. **Builder Stage** (node:18-alpine)
   - Installs dependencies
   - Builds the production bundle using Vite
   - Outputs optimized static files

2. **Production Stage** (nginx:alpine)
   - Uses lightweight nginx Alpine image
   - Serves static files from the build stage
   - Includes health check endpoint

### Key Features

- **Multi-stage build**: Reduces final image size by excluding build dependencies
- **Alpine Linux**: Minimal base image for smaller size
- **Health check**: Ensures container is running correctly
- **Optimized caching**: Layer caching for faster rebuilds

### Image Optimization Techniques

1. **Multi-stage builds**: Separates build and runtime environments
2. **Alpine base images**: Reduces image size significantly
3. **Layer caching**: Optimized layer order for better caching
4. **.dockerignore**: Excludes unnecessary files from build context

### Building the Image

```bash
cd finch-frontend
docker build -t finch-frontend:latest .
```

### Running the Container

```bash
docker run -d -p 5173:80 finch-frontend:latest
```

Access the application at `http://localhost:5173`

## Backend Dockerfile

**Location**: `finch-backend/Dockerfile`

### Architecture

The backend Dockerfile uses a **multi-stage build** approach:

1. **Dependencies Stage** (node:18-alpine)
   - Installs production dependencies only
   - Creates optimized node_modules layer

2. **Production Stage** (node:18-alpine)
   - Copies dependencies from previous stage
   - Copies application code
   - Runs as non-root user for security
   - Includes health check endpoint

### Key Features

- **Production-only dependencies**: Reduces image size
- **Non-root user**: Enhanced security
- **Health check**: Monitors application health
- **Optimized layer structure**: Better caching

### Image Optimization Techniques

1. **Separate dependency installation**: Better layer caching
2. **Production dependencies only**: Smaller node_modules
3. **Non-root user**: Security best practice
4. **Alpine base**: Minimal image size

### Building the Image

```bash
cd finch-backend
docker build -t finch-backend:latest .
```

### Running the Container

```bash
docker run -d -p 3000:3000 \
  -e DATABASE_URL=postgresql://user:pass@host:5432/db \
  -e REDIS_URL=redis://host:6379 \
  finch-backend:latest
```

## .dockerignore Files

Both applications include `.dockerignore` files to exclude unnecessary files from the Docker build context, reducing build time and image size.

### Excluded Files

- `node_modules`: Installed during build
- `.git`: Version control files
- `.env*`: Environment files (use build args or secrets)
- `*.log`: Log files
- IDE configuration files
- Test and coverage files
- Documentation files

## Docker Compose for Local Development

**Location**: `docker-compose.yml`

### Services

The docker-compose.yml file defines four services:

1. **PostgreSQL**
   - PostgreSQL 15 Alpine image
   - Persistent volume for data
   - Health checks enabled
   - Port: 5432

2. **Redis**
   - Redis 7 Alpine image
   - Persistent volume for data
   - AOF (Append Only File) enabled for durability
   - Port: 6379

3. **Backend**
   - Built from finch-backend Dockerfile
   - Connected to PostgreSQL and Redis
   - Port: 3000
   - Volume mounts for hot reloading

4. **Frontend**
   - Built from finch-frontend Dockerfile
   - Connected to backend service
   - Port: 5173 (mapped to container port 80)

### Networking

All services are connected via a bridge network (`finch-network`) allowing them to communicate using service names.

### Volumes

- `postgres_data`: PostgreSQL data persistence
- `redis_data`: Redis data persistence
- Application code volumes for development (hot reloading)

### Environment Variables

Services use environment variables for configuration:
- Database connection strings
- Redis URLs
- API endpoints
- Node environment

### Usage

#### Start all services:

```bash
docker-compose up -d
```

#### View logs:

```bash
docker-compose logs -f
```

#### Stop all services:

```bash
docker-compose down
```

#### Stop and remove volumes:

```bash
docker-compose down -v
```

#### Rebuild images:

```bash
docker-compose build --no-cache
```

## Best Practices Implemented

### Security

1. **Non-root user**: Backend runs as non-root user
2. **Minimal base images**: Alpine Linux reduces attack surface
3. **No secrets in images**: Environment variables used for sensitive data
4. **Health checks**: Ensures containers are functioning correctly

### Performance

1. **Layer caching**: Optimized layer order for better caching
2. **Multi-stage builds**: Smaller final images
3. **.dockerignore**: Faster builds by excluding unnecessary files
4. **Production dependencies only**: Smaller node_modules

### Maintainability

1. **Clear structure**: Easy to understand and modify
2. **Documentation**: Comments in Dockerfiles where needed
3. **Consistent naming**: Standard naming conventions
4. **Version pinning**: Specific base image versions

## Image Sizes

Expected image sizes (approximate):
- **Frontend**: ~50-70 MB (nginx + static files)
- **Backend**: ~150-200 MB (Node.js + dependencies)

## Troubleshooting

### Build Issues

1. **Build fails with "no space left"**
   - Clean up Docker system: `docker system prune -a`
   - Remove unused images and volumes

2. **Dependencies not found**
   - Ensure package.json and package-lock.json are present
   - Check .dockerignore doesn't exclude necessary files

3. **Permission denied errors**
   - Check file permissions in the build context
   - Ensure Docker has proper permissions

### Runtime Issues

1. **Container exits immediately**
   - Check logs: `docker logs <container-id>`
   - Verify environment variables are set
   - Check health check configuration

2. **Cannot connect to services**
   - Verify network configuration
   - Check service names in docker-compose
   - Ensure ports are not already in use

3. **Database connection fails**
   - Verify database service is running
   - Check connection string format
   - Ensure database credentials are correct

## Production Considerations

For production deployments:

1. **Use specific image tags**: Avoid `latest` tag
2. **Scan images for vulnerabilities**: Use tools like Trivy or Snyk
3. **Use secrets management**: Don't hardcode credentials
4. **Enable resource limits**: Set CPU and memory limits
5. **Use orchestration**: Deploy with Kubernetes or similar
6. **Monitor containers**: Set up logging and monitoring
7. **Regular updates**: Keep base images updated

## Next Steps

1. Set up image scanning in CI/CD pipeline
2. Configure image registry (Docker Hub, GCR, ECR, etc.)
3. Implement image signing for security
4. Set up automated security updates
5. Configure production environment variables
