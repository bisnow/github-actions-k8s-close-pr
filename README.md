# Cleanup k8s PR Stack Composite Action

This composite action safely cleans up all resources associated with a Kubernetes PR stack deployment, including Helm releases, Kubernetes resources, and CloudFormation stacks.

## Features

- **Safe Deletion**: Multi-layer verification ensures only the correct PR stack is deleted
- **Comprehensive Cleanup**: Removes Helm releases, K8s resources, and CloudFormation stacks
- **Namespace Safety**: Uses both `pr-number` and `app` labels to prevent collisions
- **Stack Verification**: Validates CloudFormation stack parameters before deletion
- **Reusable**: Generic enough to work across different services

## Safety Mechanisms

1. **Pattern Matching**: Verifies stack name matches expected pattern (e.g., `leads-pr123`)
2. **Parameter Verification**: Confirms `EnvName` and `ServiceName` match expectations
3. **Conditional Execution**: Only deletes stack if verification passes
4. **Precise Queries**: Uses both `pr-number` and `app-name` labels for K8s resource queries

## Usage

```yaml
- name: Cleanup PR Stack
  uses: ./.github/actions/cleanup-pr-stack
  with:
    pr-number: ${{ github.event.pull_request.number }}
    app-name: leads-test
    service-name: biscred-leads
    cloudformation-stack-prefix: leads-pr
```

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pr-number` | Yes | - | Pull request number |
| `app-name` | Yes | - | Application name for K8s resource queries |
| `service-name` | Yes | - | Service name for CloudFormation verification |
| `namespace` | No | `biscred-pr-stacks` | Kubernetes namespace |
| `cloudformation-stack-prefix` | Yes | - | Stack name prefix (e.g., `leads-pr` for `leads-pr123`) |
| `aws-region` | No | `us-east-1` | AWS region |
| `eks-cluster-name` | No | `bisnow-non-prod-eks` | EKS cluster name |

### Outputs

| Output | Description |
|--------|-------------|
| `cleanup-status` | Status of cleanup operation (`success` or `failed`) |

## What Gets Cleaned Up

1. **Helm Release**: Deletes `pr-{number}` release from namespace
2. **Kubernetes Resources**: Verifies all pods with matching labels are removed
3. **CloudFormation Stack**: Deletes verified stack and waits for completion

## Moving to a Separate Repository

When this action is ready for production use across multiple repositories:

1. Create a new repository: `bisnow/github-actions-k8s-close-pr`
2. Move `action.yml` and `README.md` to the repository root
3. Update workflow to use: `uses: bisnow/github-actions-k8s-close-pr@main`
4. Tag releases for version pinning: `uses: bisnow/github-actions-k8s-close-pr@v1`

## Example Workflow

```yaml
name: PR Cleanup Workflow

on:
  pull_request:
    types: [closed]

env:
  APP_NAME: leads-test
  SERVICE_NAME: biscred-leads

jobs:
  cleanup-pr-stack:
    name: Cleanup PR Stack
    runs-on: arc-runners-bisnow
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v5

      - name: Assume AWS Role
        uses: bisnow/github-actions-assume-role-for-environment@main
        with:
          aws-account: bisnow

      - name: Cleanup PR Stack
        uses: ./.github/actions/cleanup-pr-stack
        with:
          pr-number: ${{ github.event.pull_request.number }}
          app-name: ${{ env.APP_NAME }}
          service-name: ${{ env.SERVICE_NAME }}
          cloudformation-stack-prefix: leads-pr

      - name: Summary
        run: |
          echo "## 🧹 PR Environment Cleaned Up" >> $GITHUB_STEP_SUMMARY
          echo "**Service:** \`${{ env.SERVICE_NAME }}\` (\`${{ env.APP_NAME }}\`)" >> $GITHUB_STEP_SUMMARY
          echo "**PR #${{ github.event.pull_request.number }}** resources deleted" >> $GITHUB_STEP_SUMMARY
```

## Notes

- Requires AWS credentials to be configured (via OIDC or access keys)
- Requires kubectl to be configured for the target EKS cluster
- Helm must be available in the runner environment
- CloudFormation stack verification prevents accidental deletion of wrong stacks
