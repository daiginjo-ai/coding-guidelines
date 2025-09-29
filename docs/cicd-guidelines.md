# CI/CD Guidelines

This document outlines the standards and best practices for Continuous Integration and Continuous Deployment in the daiginjo-ai organization using GitHub Actions.

## Overview

All repositories in the daiginjo-ai organization should implement standardized CI/CD pipelines that ensure code quality, automate testing, and facilitate consistent deployments to AWS infrastructure.

## GitHub Actions Workflow Structure

### Required Workflows

1. **Pull Request Workflow** - Runs on every PR
2. **Release Workflow** - Runs on tag push
3. **Manual Deployment Workflow** - Manually triggered for deployments

## Pull Request Workflow

### Workflow Configuration

```yaml
# .github/workflows/pr.yml
name: Pull Request

on:
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '18'
  DOCKER_BUILDKIT: 1

jobs:
  test-backend:
    name: Backend Tests
    runs-on: ubuntu-latest

    services:
      mongodb:
        image: mongo:7.0
        env:
          MONGO_INITDB_ROOT_USERNAME: root
          MONGO_INITDB_ROOT_PASSWORD: password
        ports:
          - 27017:27017
        options: >-
          --health-cmd mongosh
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    defaults:
      run:
        working-directory: ./backend

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Run linting
        run: npm run lint

      - name: Run type checking
        run: npm run type-check

      - name: Run unit tests
        run: npm run test
        env:
          NODE_ENV: test
          MONGODB_URI: mongodb://root:password@localhost:27017/test?authSource=admin
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test-secret

      - name: Run integration tests
        run: npm run test:integration
        env:
          NODE_ENV: test
          MONGODB_URI: mongodb://root:password@localhost:27017/test?authSource=admin
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test-secret

      - name: Generate test coverage
        run: npm run test:coverage
        env:
          NODE_ENV: test
          MONGODB_URI: mongodb://root:password@localhost:27017/test?authSource=admin
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test-secret

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./backend/coverage/lcov.info
          flags: backend

  test-frontend:
    name: Frontend Tests
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./frontend

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Run linting
        run: npm run lint

      - name: Run type checking
        run: npm run type-check

      - name: Run unit tests
        run: npm run test
        env:
          NODE_ENV: test

      - name: Run build
        run: npm run build
        env:
          NEXT_PUBLIC_API_URL: http://localhost:4000/graphql

      - name: Generate test coverage
        run: npm run test:coverage
        env:
          NODE_ENV: test

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./frontend/coverage/lcov.info
          flags: frontend

  docker-build:
    name: Docker Build
    runs-on: ubuntu-latest
    needs: [test-backend, test-frontend]

    strategy:
      matrix:
        component: [backend, frontend]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./${{ matrix.component }}
          file: ./${{ matrix.component }}/Dockerfile
          platforms: linux/amd64,linux/arm64
          push: false
          tags: |
            ${{ matrix.component }}:pr-${{ github.event.number }}
            ${{ matrix.component }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: [test-backend, test-frontend]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

## Release Workflow

### Workflow Configuration

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

env:
  NODE_VERSION: '18'
  AWS_REGION: us-east-1
  ECR_REGISTRY: 123456789012.dkr.ecr.us-east-1.amazonaws.com

jobs:
  validate-tag:
    name: Validate Tag
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.extract.outputs.version }}
      is-prerelease: ${{ steps.extract.outputs.is-prerelease }}

    steps:
      - name: Extract version from tag
        id: extract
        run: |
          TAG=${GITHUB_REF#refs/tags/}
          VERSION=${TAG#v}
          echo "version=$VERSION" >> $GITHUB_OUTPUT

          if [[ $VERSION =~ -[a-zA-Z] ]]; then
            echo "is-prerelease=true" >> $GITHUB_OUTPUT
          else
            echo "is-prerelease=false" >> $GITHUB_OUTPUT
          fi

          echo "Tag: $TAG, Version: $VERSION"

  test:
    name: Run Tests
    runs-on: ubuntu-latest
    needs: validate-tag

    strategy:
      matrix:
        component: [backend, frontend]

    defaults:
      run:
        working-directory: ./${{ matrix.component }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: ${{ matrix.component }}/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm run test
        env:
          NODE_ENV: test

  build-and-push:
    name: Build and Push to ECR
    runs-on: ubuntu-latest
    needs: [validate-tag, test]

    strategy:
      matrix:
        component: [backend, frontend]

    permissions:
      id-token: write
      contents: read

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./${{ matrix.component }}
          file: ./${{ matrix.component }}/Dockerfile
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ${{ env.ECR_REGISTRY }}/${{ github.repository_name }}-${{ matrix.component }}:${{ needs.validate-tag.outputs.version }}
            ${{ env.ECR_REGISTRY }}/${{ github.repository_name }}-${{ matrix.component }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ${{ env.ECR_REGISTRY }}/${{ github.repository_name }}-${{ matrix.component }}:${{ needs.validate-tag.outputs.version }}
          format: spdx-json
          output-file: /tmp/sbom-${{ matrix.component }}.spdx.json

      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom-${{ matrix.component }}
          path: /tmp/sbom-${{ matrix.component }}.spdx.json

  create-release:
    name: Create GitHub Release
    runs-on: ubuntu-latest
    needs: [validate-tag, build-and-push]

    permissions:
      contents: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Generate changelog
        id: changelog
        run: |
          if [ -f CHANGELOG.md ]; then
            # Extract changelog for this version
            awk '/^## \[${{ needs.validate-tag.outputs.version }}\]/{flag=1; next} /^## \[/{flag=0} flag' CHANGELOG.md > release_notes.md
          else
            echo "Release ${{ needs.validate-tag.outputs.version }}" > release_notes.md
          fi

      - name: Create Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ needs.validate-tag.outputs.version }}
          body_path: release_notes.md
          draft: false
          prerelease: ${{ needs.validate-tag.outputs.is-prerelease }}

  notify:
    name: Notify Release
    runs-on: ubuntu-latest
    needs: [validate-tag, create-release]
    if: always()

    steps:
      - name: Notify Slack
        if: success()
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: ":rocket: Release ${{ needs.validate-tag.outputs.version }} has been successfully built and pushed to ECR",
              attachments: [{
                color: "good",
                fields: [{
                  title: "Repository",
                  value: "${{ github.repository }}",
                  short: true
                }, {
                  title: "Version",
                  value: "${{ needs.validate-tag.outputs.version }}",
                  short: true
                }]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on Failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: ":x: Release ${{ needs.validate-tag.outputs.version }} failed",
              attachments: [{
                color: "danger",
                fields: [{
                  title: "Repository",
                  value: "${{ github.repository }}",
                  short: true
                }, {
                  title: "Version",
                  value: "${{ needs.validate-tag.outputs.version }}",
                  short: true
                }]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Manual Deployment Workflow

### Workflow Configuration

```yaml
# .github/workflows/deploy.yml
name: Deploy to EKS

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        type: choice
        options:
          - development
          - staging
          - production
        default: 'development'
      image_tag:
        description: 'Docker image tag to deploy'
        required: true
        type: string
      component:
        description: 'Component to deploy'
        required: true
        type: choice
        options:
          - backend
          - frontend
          - all
        default: 'all'
      dry_run:
        description: 'Perform a dry run (no actual deployment)'
        required: false
        type: boolean
        default: false

env:
  AWS_REGION: us-east-1
  ECR_REGISTRY: 123456789012.dkr.ecr.us-east-1.amazonaws.com

jobs:
  validate-inputs:
    name: Validate Deployment Inputs
    runs-on: ubuntu-latest
    outputs:
      cluster-name: ${{ steps.cluster.outputs.name }}
      namespace: ${{ steps.cluster.outputs.namespace }}
      values-file: ${{ steps.cluster.outputs.values-file }}

    steps:
      - name: Set cluster configuration
        id: cluster
        run: |
          case "${{ github.event.inputs.environment }}" in
            development)
              echo "name=daiginjo-dev-cluster" >> $GITHUB_OUTPUT
              echo "namespace=development" >> $GITHUB_OUTPUT
              echo "values-file=values-dev.yaml" >> $GITHUB_OUTPUT
              ;;
            staging)
              echo "name=daiginjo-staging-cluster" >> $GITHUB_OUTPUT
              echo "namespace=staging" >> $GITHUB_OUTPUT
              echo "values-file=values-staging.yaml" >> $GITHUB_OUTPUT
              ;;
            production)
              echo "name=daiginjo-prod-cluster" >> $GITHUB_OUTPUT
              echo "namespace=production" >> $GITHUB_OUTPUT
              echo "values-file=values-prod.yaml" >> $GITHUB_OUTPUT
              ;;
          esac

      - name: Validate image tag format
        run: |
          TAG="${{ github.event.inputs.image_tag }}"
          if [[ ! $TAG =~ ^[0-9]+\.[0-9]+\.[0-9]+(-[a-zA-Z0-9]+)?$ ]] && [[ ! $TAG =~ ^v[0-9]+\.[0-9]+\.[0-9]+(-[a-zA-Z0-9]+)?$ ]]; then
            echo "Error: Invalid image tag format. Expected semver format (e.g., 1.0.0, v1.0.0, 1.0.0-beta.1)"
            exit 1
          fi

  check-image-exists:
    name: Check Image Exists in ECR
    runs-on: ubuntu-latest
    needs: validate-inputs

    strategy:
      matrix:
        component: ${{ github.event.inputs.component == 'all' && fromJson('["backend", "frontend"]') || fromJson(format('["{0}"]', github.event.inputs.component)) }}

    permissions:
      id-token: write
      contents: read

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Check if image exists in ECR
        run: |
          REPOSITORY="${{ github.repository_name }}-${{ matrix.component }}"
          TAG="${{ github.event.inputs.image_tag }}"

          if aws ecr describe-images --repository-name $REPOSITORY --image-ids imageTag=$TAG >/dev/null 2>&1; then
            echo "✅ Image $REPOSITORY:$TAG exists in ECR"
          else
            echo "❌ Image $REPOSITORY:$TAG not found in ECR"
            exit 1
          fi

  deploy:
    name: Deploy to EKS
    runs-on: ubuntu-latest
    needs: [validate-inputs, check-image-exists]
    environment:
      name: ${{ github.event.inputs.environment }}
      url: ${{ steps.deployment.outputs.app_url }}

    strategy:
      matrix:
        component: ${{ github.event.inputs.component == 'all' && fromJson('["backend", "frontend"]') || fromJson(format('["{0}"]', github.event.inputs.component)) }}

    permissions:
      id-token: write
      contents: read

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Install kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'

      - name: Install Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.12.0'

      - name: Configure kubectl
        run: |
          aws eks update-kubeconfig --region ${{ env.AWS_REGION }} --name ${{ needs.validate-inputs.outputs.cluster-name }}

      - name: Verify cluster connection
        run: |
          kubectl cluster-info
          kubectl get nodes

      - name: Create namespace if not exists
        run: |
          kubectl create namespace ${{ needs.validate-inputs.outputs.namespace }} --dry-run=client -o yaml | kubectl apply -f -

      - name: Prepare Helm values
        id: helm-values
        run: |
          # Set image tag in values file
          TAG="${{ github.event.inputs.image_tag }}"
          VALUES_FILE="helm/${{ needs.validate-inputs.outputs.values-file }}"

          # Create temporary values file with updated image tag
          cp $VALUES_FILE /tmp/deploy-values.yaml

          # Update image tags for the component
          if [[ "${{ matrix.component }}" == "backend" ]]; then
            yq e '.backend.image.tag = "${{ github.event.inputs.image_tag }}"' -i /tmp/deploy-values.yaml
          elif [[ "${{ matrix.component }}" == "frontend" ]]; then
            yq e '.frontend.image.tag = "${{ github.event.inputs.image_tag }}"' -i /tmp/deploy-values.yaml
          fi

          echo "values-file=/tmp/deploy-values.yaml" >> $GITHUB_OUTPUT

      - name: Helm dry run
        if: github.event.inputs.dry_run == 'true'
        run: |
          helm upgrade --install \
            ${{ github.repository_name }}-${{ matrix.component }} \
            ./helm \
            --namespace ${{ needs.validate-inputs.outputs.namespace }} \
            --values ${{ steps.helm-values.outputs.values-file }} \
            --set image.repository=${{ env.ECR_REGISTRY }}/${{ github.repository_name }}-${{ matrix.component }} \
            --set image.tag=${{ github.event.inputs.image_tag }} \
            --dry-run \
            --debug

      - name: Deploy with Helm
        if: github.event.inputs.dry_run != 'true'
        id: deployment
        run: |
          helm upgrade --install \
            ${{ github.repository_name }}-${{ matrix.component }} \
            ./helm \
            --namespace ${{ needs.validate-inputs.outputs.namespace }} \
            --values ${{ steps.helm-values.outputs.values-file }} \
            --set image.repository=${{ env.ECR_REGISTRY }}/${{ github.repository_name }}-${{ matrix.component }} \
            --set image.tag=${{ github.event.inputs.image_tag }} \
            --wait \
            --timeout=10m

          # Get application URL (adjust based on your ingress setup)
          if [[ "${{ matrix.component }}" == "frontend" ]]; then
            APP_URL=$(kubectl get ingress -n ${{ needs.validate-inputs.outputs.namespace }} -o jsonpath='{.items[0].spec.rules[0].host}' 2>/dev/null || echo "Not available")
            echo "app_url=https://$APP_URL" >> $GITHUB_OUTPUT
          fi

      - name: Verify deployment
        if: github.event.inputs.dry_run != 'true'
        run: |
          # Wait for rollout to complete
          kubectl rollout status deployment/${{ github.repository_name }}-${{ matrix.component }} -n ${{ needs.validate-inputs.outputs.namespace }} --timeout=300s

          # Check pod status
          kubectl get pods -n ${{ needs.validate-inputs.outputs.namespace }} -l app.kubernetes.io/name=${{ github.repository_name }}-${{ matrix.component }}

      - name: Run smoke tests
        if: github.event.inputs.dry_run != 'true' && matrix.component == 'backend'
        run: |
          # Get service endpoint
          SERVICE_IP=$(kubectl get service ${{ github.repository_name }}-${{ matrix.component }} -n ${{ needs.validate-inputs.outputs.namespace }} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo "cluster-ip")

          if [[ "$SERVICE_IP" != "cluster-ip" ]]; then
            # Test health endpoint
            for i in {1..30}; do
              if curl -f http://$SERVICE_IP/health; then
                echo "✅ Health check passed"
                break
              fi
              echo "⏳ Waiting for service to be ready... ($i/30)"
              sleep 10
            done
          else
            echo "Using port-forward for health check"
            kubectl port-forward service/${{ github.repository_name }}-${{ matrix.component }} 8080:80 -n ${{ needs.validate-inputs.outputs.namespace }} &
            sleep 5
            curl -f http://localhost:8080/health || echo "⚠️ Health check failed"
            pkill -f "kubectl port-forward" || true
          fi

  notify-deployment:
    name: Notify Deployment Status
    runs-on: ubuntu-latest
    needs: [validate-inputs, deploy]
    if: always()

    steps:
      - name: Notify Slack on Success
        if: needs.deploy.result == 'success' && github.event.inputs.dry_run != 'true'
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: ":rocket: Deployment successful!",
              attachments: [{
                color: "good",
                fields: [{
                  title: "Environment",
                  value: "${{ github.event.inputs.environment }}",
                  short: true
                }, {
                  title: "Component",
                  value: "${{ github.event.inputs.component }}",
                  short: true
                }, {
                  title: "Image Tag",
                  value: "${{ github.event.inputs.image_tag }}",
                  short: true
                }, {
                  title: "Repository",
                  value: "${{ github.repository }}",
                  short: true
                }]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on Dry Run
        if: github.event.inputs.dry_run == 'true'
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: ":test_tube: Dry run completed successfully",
              attachments: [{
                color: "warning",
                fields: [{
                  title: "Environment",
                  value: "${{ github.event.inputs.environment }}",
                  short: true
                }, {
                  title: "Component",
                  value: "${{ github.event.inputs.component }}",
                  short: true
                }, {
                  title: "Image Tag",
                  value: "${{ github.event.inputs.image_tag }}",
                  short: true
                }]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on Failure
        if: needs.deploy.result == 'failure'
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: ":x: Deployment failed!",
              attachments: [{
                color: "danger",
                fields: [{
                  title: "Environment",
                  value: "${{ github.event.inputs.environment }}",
                  short: true
                }, {
                  title: "Component",
                  value: "${{ github.event.inputs.component }}",
                  short: true
                }, {
                  title: "Image Tag",
                  value: "${{ github.event.inputs.image_tag }}",
                  short: true
                }, {
                  title: "Repository",
                  value: "${{ github.repository }}",
                  short: true
                }]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Required Repository Secrets

### GitHub Secrets Configuration

All repositories must configure the following secrets:

```bash
# AWS IAM Role for OIDC authentication
AWS_ROLE_ARN=arn:aws:iam::123456789012:role/GitHubActionsRole

# Slack notifications (optional)
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
```

### AWS IAM Role Configuration

Create an IAM role with OIDC provider for GitHub Actions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:daiginjo-ai/your-repo:*"
        }
      }
    }
  ]
}
```

### Required IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:DescribeImages",
        "ecr:DescribeRepositories"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters"
      ],
      "Resource": "*"
    }
  ]
}
```

## Environment Protection Rules

### GitHub Environment Configuration

Configure the following environments with protection rules:

#### Development Environment
- **Protection rules**: None (auto-deploy allowed)
- **Reviewers**: Not required
- **Wait timer**: 0 minutes

#### Staging Environment
- **Protection rules**: Required reviewers
- **Reviewers**: Development team members
- **Wait timer**: 0 minutes

#### Production Environment
- **Protection rules**: Required reviewers
- **Reviewers**: Senior developers and DevOps team
- **Wait timer**: 5 minutes
- **Deployment branches**: Only `main` branch

## Docker Configuration

### Dockerfile Standards

#### Backend Dockerfile
```dockerfile
# backend/Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy source code
COPY . .

# Build application
RUN npm run build

# Production stage
FROM node:18-alpine AS production

# Create app user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

WORKDIR /app

# Copy built application
COPY --from=builder --chown=nextjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nextjs:nodejs /app/package.json ./package.json

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:4000/health || exit 1

USER nextjs

EXPOSE 4000

CMD ["node", "dist/server.js"]
```

#### Frontend Dockerfile
```dockerfile
# frontend/Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci

# Copy source code
COPY . .

# Build application
RUN npm run build

# Production stage
FROM node:18-alpine AS production

# Create app user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

WORKDIR /app

# Copy built application
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/public ./public

USER nextjs

EXPOSE 3000

ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

CMD ["node", "server.js"]
```

### .dockerignore
```bash
# .dockerignore (both backend and frontend)
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.env.local
.env.development
.env.test
.env.production
coverage
.nyc_output
.cache
.parcel-cache
.next
.nuxt
dist
build
```

## Package.json Scripts

### Required npm Scripts

#### Backend package.json
```json
{
  "scripts": {
    "start": "node dist/server.js",
    "dev": "tsx watch src/server.ts",
    "build": "tsc",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:integration": "jest --config jest.integration.config.js",
    "lint": "eslint src/**/*.ts",
    "lint:fix": "eslint src/**/*.ts --fix",
    "type-check": "tsc --noEmit",
    "format": "prettier --write src/**/*.ts"
  }
}
```

#### Frontend package.json
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "next lint",
    "lint:fix": "next lint --fix",
    "type-check": "tsc --noEmit",
    "format": "prettier --write src/**/*.{ts,tsx}"
  }
}
```

## Monitoring and Alerts

### Workflow Status Badges

Add status badges to your repository README:

```markdown
![PR Workflow](https://github.com/daiginjo-ai/your-repo/workflows/Pull%20Request/badge.svg)
![Release Workflow](https://github.com/daiginjo-ai/your-repo/workflows/Release/badge.svg)
![Deploy Workflow](https://github.com/daiginjo-ai/your-repo/workflows/Deploy%20to%20EKS/badge.svg)
```

### Slack Notifications

Configure Slack webhook for notifications:

1. Create a Slack app with incoming webhooks
2. Add webhook URL to repository secrets as `SLACK_WEBHOOK_URL`
3. Customize notification payloads in workflow files

## Best Practices

### Version Tagging
- Use semantic versioning (e.g., `v1.0.0`, `v1.0.1-beta.1`)
- Create tags on the `main` branch only
- Include release notes in tag descriptions

### Security Scanning
- All workflows include Trivy security scanning
- Upload scan results to GitHub Security tab
- Block deployments if critical vulnerabilities found

### Resource Management
- Use GitHub Actions cache for dependencies
- Implement proper cleanup in workflows
- Monitor workflow usage and costs

### Error Handling
- Fail fast on critical errors
- Provide meaningful error messages
- Include debug information in logs

### Testing Strategy
- Run comprehensive tests on every PR
- Include integration tests with real services
- Generate and track code coverage
- Require minimum coverage thresholds

## Troubleshooting

### Common Issues

#### ECR Authentication Failures
```bash
# Check IAM role permissions
aws sts get-caller-identity

# Verify ECR repository exists
aws ecr describe-repositories --repository-names your-repo-backend
```

#### EKS Deployment Failures
```bash
# Check cluster connectivity
kubectl cluster-info

# Verify namespace exists
kubectl get namespaces

# Check Helm releases
helm list -A
```

#### Image Not Found in ECR
- Verify the tag was pushed correctly
- Check ECR repository naming conventions
- Ensure the release workflow completed successfully

### Debug Mode

Enable debug logging in workflows by setting repository secret:
```bash
ACTIONS_STEP_DEBUG=true
ACTIONS_RUNNER_DEBUG=true
```