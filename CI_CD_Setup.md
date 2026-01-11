# CI/CD Setup Documentation

## Overview

This document describes the CI/CD pipeline architecture for the finch-frontend and finch-backend applications. The pipelines are implemented using GitHub Actions, providing automated build, test, and deployment workflows.

## CI/CD Tool: GitHub Actions

GitHub Actions was chosen as the CI/CD tool for the following reasons:
- Native integration with GitHub repositories
- Free tier for public repositories
- Extensive marketplace of actions
- Easy configuration via YAML files
- Support for Docker builds and container registry pushes

## Pipeline Architecture

### Frontend CI/CD Pipeline

**Location**: `finch-frontend/.github/workflows/frontend-ci.yml`

#### Pipeline Stages

1. **Checkout Code**
   - Checks out the repository code
   - Uses the `finch-frontend` path for the workflow

2. **Setup Node.js**
   - Configures Node.js version 18
   - Enables npm caching for faster builds

3. **Install Dependencies**
   - Runs `npm ci` for clean, reproducible installs
   - Uses package-lock.json for version consistency

4. **Code Quality Checks**
   - Runs ESLint to check code quality
   - Identifies potential issues and enforces coding standards

5. **Unit Tests**
   - Executes unit tests using Vitest
   - Fails the build if tests fail

6. **Build Production Bundle**
   - Runs `npm run build` to create optimized production bundle
   - Generates static assets ready for deployment

7. **Docker Image Build**
   - Builds Docker image using Docker Buildx
   - Tags images with branch name, commit SHA, and latest (for main branch)
   - Uses registry cache for faster builds

8. **Push to Container Registry**
   - Pushes Docker image to Docker Hub (or configured registry)
   - Only executes on push events (not pull requests)

#### Triggers

- Push to `main` or `master` branch
- Pull requests to `main` or `master` branch
- Only triggers when files in `finch-frontend/` directory change

### Backend CI/CD Pipeline

**Location**: `finch-backend/.github/workflows/backend-ci.yml`

#### Pipeline Stages

1. **Checkout Code**
   - Checks out the repository code
   - Uses the `finch-backend` path for the workflow

2. **Setup PostgreSQL Service**
   - Starts PostgreSQL 15 container for integration tests
   - Configures test database credentials

3. **Setup Node.js**
   - Configures Node.js version 18
   - Enables npm caching

4. **Install Dependencies**
   - Runs `npm ci` for clean installs

5. **Code Quality Checks**
   - Runs ESLint to check code quality

6. **Unit and Integration Tests**
   - Executes unit tests
   - Runs integration tests against PostgreSQL service
   - Uses test database connection string

7. **Docker Image Build**
   - Builds Docker image using Docker Buildx
   - Tags images with branch name, commit SHA, and latest

8. **Push to Container Registry**
   - Pushes Docker image to Docker Hub
   - Only executes on push events

#### Triggers

- Push to `main` or `master` branch
- Pull requests to `main` or `master` branch
- Only triggers when files in `finch-backend/` directory change

## Required Secrets

To use these pipelines, you need to configure the following GitHub Secrets:

1. **DOCKER_USERNAME**: Your Docker Hub username
2. **DOCKER_PASSWORD**: Your Docker Hub password or access token

### Setting Up Secrets

1. Go to your GitHub repository
2. Navigate to Settings → Secrets and variables → Actions
3. Click "New repository secret"
4. Add each secret with the appropriate name and value

## Image Tagging Strategy

Images are tagged with:
- **Branch name**: For branch-specific builds
- **Commit SHA**: For traceability (format: `{branch}-{sha}`)
- **Latest**: Only for the default branch (main/master)

Example tags:
- `your-username/finch-frontend:main`
- `your-username/finch-frontend:main-abc1234`
- `your-username/finch-frontend:latest`

## Monitoring Pipelines

### Viewing Pipeline Status

1. Go to your GitHub repository
2. Click on the "Actions" tab
3. Select the workflow you want to view
4. Click on a specific run to see detailed logs

### Pipeline Notifications

GitHub Actions sends notifications for:
- Pipeline failures
- Successful deployments
- Pull request status updates

## Troubleshooting

### Common Issues

1. **Docker Login Fails**
   - Verify DOCKER_USERNAME and DOCKER_PASSWORD secrets are set correctly
   - Ensure Docker Hub account has proper permissions

2. **Tests Fail**
   - Check test logs in the Actions tab
   - Verify all dependencies are installed correctly
   - Ensure test database is accessible (for backend)

3. **Build Fails**
   - Check build logs for specific errors
   - Verify Dockerfile syntax
   - Ensure all required files are present

4. **Image Push Fails**
   - Verify registry credentials
   - Check if image name follows Docker Hub naming conventions
   - Ensure you have push permissions to the repository

## Best Practices

1. **Always test locally** before pushing to trigger CI/CD
2. **Use feature branches** and create pull requests for code review
3. **Monitor pipeline execution** to catch issues early
4. **Keep dependencies updated** to avoid security vulnerabilities
5. **Use semantic versioning** for production releases

## Next Steps

After setting up CI/CD:
1. Configure Docker Hub repository names
2. Set up GitHub Secrets
3. Test the pipeline with a test commit
4. Monitor first successful build
5. Configure deployment automation (if needed)
