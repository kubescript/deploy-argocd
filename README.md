# Deploy to ArgoCD

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Deploy%20to%20ArgoCD-blue.svg?colorA=24292e&colorB=0366d6&style=flat&longCache=true&logo=github)](https://github.com/marketplace/actions/deploy-to-argocd)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

A GitHub Action to deploy applications to Kubernetes using ArgoCD's GitOps approach.

## Features

- **GitOps Deployment**: Declarative deployment using ArgoCD
- **Automatic CLI Installation**: Installs ArgoCD CLI if not present
- **Application Sync**: Wait for application to be healthy and synced
- **Flexible Configuration**: Support for multiple environments, namespaces, and clusters
- **Helm Support**: Works with Helm charts and environment-specific values files
- **Auto-Sync Options**: Configure auto-prune, self-heal, and namespace creation
- **Multi-Environment**: Deploy to different environments using different values files

## Prerequisites

### 1. ArgoCD Server

You need a running ArgoCD server:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. ArgoCD Authentication Token

Generate an authentication token:

```bash
# Login to ArgoCD
argocd login argocd.example.com

# Generate token (for a user or project)
argocd account generate-token --account github-actions

# Or use admin token
argocd account generate-token
```

Store the token as a GitHub secret (e.g., `ARGOCD_AUTH_TOKEN`).

### 3. Cluster Registration

Register your Kubernetes cluster with ArgoCD:

```bash
# Add cluster
argocd cluster add my-cluster-context

# List clusters
argocd cluster list
```

### 4. Repository Access

Ensure ArgoCD has access to your Git repository:

```bash
# Add repository
argocd repo add https://github.com/org/repo --username USERNAME --password TOKEN

# For private repos, use SSH or HTTPS with credentials
```

### 5. Helm Chart Structure

Your repository should have a Helm chart with environment-specific values:

```
my-app/
├── Chart.yaml
├── values.yaml              # Default values
├── values-dev.yaml          # Development environment
├── values-staging.yaml      # Staging environment
├── values-production.yaml   # Production environment
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ...
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `application-name` | ArgoCD application name | Yes | - |
| `argocd-server-url` | ArgoCD server URL (e.g., `argocd.example.com`) | Yes | - |
| `argocd-auth-token` | ArgoCD authentication token | Yes | - |
| `argocd-project` | ArgoCD project name | Yes | - |
| `argocd-destination-cluster` | Destination cluster name (as registered in ArgoCD) | Yes | - |
| `repository-url` | Git repository URL containing Helm chart | Yes | - |
| `chart-path` | Path to Helm chart in repository | Yes | - |
| `namespace` | Kubernetes namespace to deploy to | Yes | - |
| `app-namespace` | ArgoCD application namespace | Yes | - |
| `environment` | Environment name (for values file) | Yes | - |
| `image-tag` | Docker image tag to deploy | Yes | - |
| `revision` | Git revision to deploy (branch, tag, or SHA) | No | `HEAD` |
| `sync-policy` | Sync policy: `manual` or `auto` | No | `manual` |
| `argocd-wait` | Wait for application to be healthy | No | `true` |
| `argocd-wait-timeout` | Wait timeout in seconds | No | `300` |
| `auto-create-namespace` | Automatically create namespace | No | `false` |
| `auto-prune` | Enable automatic pruning of resources | No | `false` |
| `self-heal` | Enable self-healing | No | `false` |
| `grpc-keep-alive-min` | gRPC keep alive minimum time | No | `20s` |
| `argocd-wait-additional-args` | Additional args for wait command | No | `""` |
| `argocd-create-app-additional-args` | Additional args for create command | No | `""` |

## Usage

### Basic Example

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy to ArgoCD
        uses: kubescript/deploy-argocd@v1
        with:
          application-name: my-app
          argocd-server-url: argocd.example.com
          argocd-auth-token: ${{ secrets.ARGOCD_AUTH_TOKEN }}
          argocd-project: default
          argocd-destination-cluster: production-cluster
          repository-url: https://github.com/org/repo
          chart-path: helm/my-app
          namespace: my-app
          app-namespace: argocd
          environment: production
          image-tag: ${{ github.sha }}
```

### Multi-Environment Deployment

```yaml
name: Deploy to Environments

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - dev
          - staging
          - production

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy to ArgoCD
        uses: kubescript/deploy-argocd@v1
        with:
          application-name: my-app-${{ inputs.environment }}
          argocd-server-url: ${{ vars.ARGOCD_SERVER_URL }}
          argocd-auth-token: ${{ secrets.ARGOCD_AUTH_TOKEN }}
          argocd-project: ${{ inputs.environment }}
          argocd-destination-cluster: ${{ inputs.environment }}-cluster
          repository-url: ${{ github.repositoryUrl }}
          chart-path: helm/my-app
          namespace: my-app-${{ inputs.environment }}
          app-namespace: argocd
          environment: ${{ inputs.environment }}
          image-tag: ${{ github.sha }}
          auto-create-namespace: true
```

### Auto-Sync with Self-Heal

```yaml
name: Deploy with Auto-Sync

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy to ArgoCD
        uses: kubescript/deploy-argocd@v1
        with:
          application-name: my-app
          argocd-server-url: argocd.example.com
          argocd-auth-token: ${{ secrets.ARGOCD_AUTH_TOKEN }}
          argocd-project: default
          argocd-destination-cluster: production-cluster
          repository-url: https://github.com/org/repo
          chart-path: helm/my-app
          namespace: my-app
          app-namespace: argocd
          environment: production
          image-tag: ${{ github.sha }}
          sync-policy: auto
          auto-prune: true
          self-heal: true
          auto-create-namespace: true
```

### Deploy Specific Git Revision

```yaml
name: Deploy Tagged Release

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Extract version
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

      - name: Deploy to ArgoCD
        uses: kubescript/deploy-argocd@v1
        with:
          application-name: my-app
          argocd-server-url: argocd.example.com
          argocd-auth-token: ${{ secrets.ARGOCD_AUTH_TOKEN }}
          argocd-project: default
          argocd-destination-cluster: production-cluster
          repository-url: https://github.com/org/repo
          chart-path: helm/my-app
          namespace: my-app
          app-namespace: argocd
          environment: production
          image-tag: ${{ steps.version.outputs.VERSION }}
          revision: ${{ steps.version.outputs.VERSION }}
```

### Monorepo with Multiple Services

```yaml
name: Deploy Microservices

on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to deploy'
        required: true
        type: choice
        options:
          - api
          - web
          - worker

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy service to ArgoCD
        uses: kubescript/deploy-argocd@v1
        with:
          application-name: ${{ inputs.service }}
          argocd-server-url: argocd.example.com
          argocd-auth-token: ${{ secrets.ARGOCD_AUTH_TOKEN }}
          argocd-project: microservices
          argocd-destination-cluster: production-cluster
          repository-url: https://github.com/org/monorepo
          chart-path: services/${{ inputs.service }}/helm
          namespace: ${{ inputs.service }}
          app-namespace: argocd
          environment: production
          image-tag: ${{ github.sha }}
```

### Complete CI/CD Pipeline

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.build.outputs.image-tag }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build and push to ECR
        id: build
        uses: kubescript/build-to-ecr@v1
        with:
          aws-creds-role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
          ecr-url: ${{ secrets.ECR_URL }}
          ecr-repo: my-app
          docker-tag: ${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy to ArgoCD
        uses: kubescript/deploy-argocd@v1
        with:
          application-name: my-app
          argocd-server-url: argocd.example.com
          argocd-auth-token: ${{ secrets.ARGOCD_AUTH_TOKEN }}
          argocd-project: default
          argocd-destination-cluster: production-cluster
          repository-url: ${{ github.repositoryUrl }}
          chart-path: helm/my-app
          namespace: my-app
          app-namespace: argocd
          environment: production
          image-tag: ${{ needs.build.outputs.image-tag }}
          argocd-wait: true
          argocd-wait-timeout: 600
```

## Troubleshooting

### Error: "application not found"

**Problem**: ArgoCD application doesn't exist.

**Solution**: The action creates the application if it doesn't exist (using `--upsert` flag). Ensure the `app-namespace` and `application-name` are correct.

### Error: "cluster not found"

**Problem**: Destination cluster is not registered in ArgoCD.

**Solution**: Register the cluster:
```bash
argocd cluster add my-cluster-context
argocd cluster list
```

### Error: "repository not found"

**Problem**: ArgoCD doesn't have access to the Git repository.

**Solution**: Add the repository to ArgoCD:
```bash
argocd repo add https://github.com/org/repo --username USER --password TOKEN
```

### Application not syncing

**Problem**: Application created but not syncing.

**Solution**:
1. Check if `sync-policy` is set to `auto` or manually trigger sync
2. Verify the `chart-path` points to a valid Helm chart
3. Ensure the values file exists: `values-{environment}.yaml`
4. Check ArgoCD UI for specific errors

### Timeout waiting for application

**Problem**: Application health check times out.

**Solution**:
1. Increase `argocd-wait-timeout` value
2. Check pod logs for startup issues
3. Verify Kubernetes resources are valid
4. Use `argocd-wait: false` to skip health check

### Permission denied errors

**Problem**: ArgoCD token lacks permissions.

**Solution**: Ensure the token has appropriate permissions:
```bash
# Create service account with proper permissions
argocd proj role create my-project deployer
argocd proj role add-policy my-project deployer -a update -o '*' -p allow
```

### Values file not found

**Problem**: ArgoCD can't find `values-{environment}.yaml`.

**Solution**: Ensure your Helm chart includes the values file for the specified environment. The action uses `--values values-{environment}.yaml`.

## Best Practices

1. **Use Environments**: Leverage GitHub Environments for environment-specific secrets and approvals
2. **Separate Projects**: Create separate ArgoCD projects for different environments
3. **Health Checks**: Keep `argocd-wait: true` to verify deployments succeed
4. **Auto-Sync Carefully**: Only enable auto-sync, auto-prune, and self-heal in non-production environments initially
5. **Version Tracking**: Use image tags that are traceable (git SHA, semver tags)
6. **Namespace Management**: Use `auto-create-namespace: true` for new applications
7. **Token Security**: Store ArgoCD tokens in GitHub Secrets, rotate regularly

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

## Support

- Report issues: [GitHub Issues](https://github.com/kubescript/deploy-argocd/issues)
- Documentation: [GitHub Marketplace](https://github.com/marketplace/actions/deploy-to-argocd)
- ArgoCD Docs: [https://argo-cd.readthedocs.io](https://argo-cd.readthedocs.io)

---

Made with ❤️ by [KubeScript](https://github.com/kubescript)
