# Cleanup k8s PR Stack

Cleans up all resources associated with a Kubernetes PR stack: Helm releases, Kubernetes resources, and CloudFormation stacks.


## Usage

```yaml
name: PR Cleanup Workflow

permissions:
  id-token: write
  contents: read
  pull-requests: read

env:
  APP_NAME: leads-test
  SERVICE_NAME: biscred-leads

on:
  pull_request:
    branches:
      - main
    types: [closed]

concurrency:
  group: pr-cleanup-${{ github.event.pull_request.number }}

jobs:
  cleanup-pr-stack:
    name: Cleanup PR Stack
    runs-on: arc-runners-bisnow
    steps:
      - name: Cleanup PR Stack
        uses: bisnow/github-actions-k8s-close-pr@main
        with:
          pr-number: ${{ github.event.pull_request.number }}
          app-name: ${{ env.APP_NAME }}
          service-name: ${{ env.SERVICE_NAME }}
          cloudformation-stack-prefix: leads-pr
          namespace: "biscred-pr-stacks"


```

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pr-number` | Yes | - | Pull request number |
| `app-name` | Yes | - | Application name for K8s resource queries |
| `service-name` | Yes | - | Service name for CloudFormation verification |
| `namespace` | Yes | - | Kubernetes namespace where PR stack is deployed |
| `cloudformation-stack-prefix` | Yes | - | Stack name prefix (e.g., `leads-pr` for `leads-pr123`) |
| `aws-account` | No | `bisnow` | AWS account name for role assumption |
| `aws-region` | No | `us-east-1` | AWS region |
| `eks-cluster-name` | No | `bisnow-non-prod-eks` | EKS cluster name |

### Outputs

| Output | Description |
|--------|-------------|
| `cleanup-status` | Status of cleanup operation (`success` or `failed`) |

## What It Does

1. Assumes AWS role for the specified account
2. Configures kubectl for the EKS cluster
3. Deletes the Helm release `pr-{number}` from the namespace
4. Verifies all Kubernetes pods are removed
5. Verifies the CloudFormation stack matches the PR (checks stack name pattern and parameters)
6. Deletes the CloudFormation stack and waits for completion

## Requirements

- OIDC permissions (`id-token: write`) for AWS role assumption
- The action uses `bisnow/github-actions-assume-role-for-environment` for authentication
