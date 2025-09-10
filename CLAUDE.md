# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository demonstrates container image signing enforcement through admission control policies. It contains GitHub Actions workflows that deploy identical applications to EKS with two paths: **signed images** (using Cosign/Fulcio) and **unsigned images**. The demo showcases how admission controllers can differentiate and block unsigned container deployments while allowing signed ones to proceed.

## Demo Architecture

The project implements a **dual deployment strategy** to highlight admission control enforcement:

1. **Signed Deployment Path**: Images signed with Cosign keyless signing via Fulcio
2. **Unsigned Deployment Path**: Identical images without cryptographic signatures  
3. **Admission Controller**: Out-of-band component that enforces signature verification

This setup demonstrates real-world container supply chain security where admission controllers act as the enforcement point for signature policies.

## Architecture

The repository is organized around three primary GitHub Actions workflows:

### Workflow Architecture

**Signed Deployment Workflows:**
1. **cosign-keyless-wiz-demo.yml** - Build, push, sign with Cosign, and deploy signed images
2. **sign-and-deploy-eks.yml** - Sign existing images with Cosign and deploy to EKS
3. **ecr-copy-sign-deploy.yml** - Copy upstream images to ECR, sign with Cosign, and deploy

**Future Unsigned Deployment Workflow:**
- Will deploy identical applications without Cosign signatures to demonstrate admission controller blocking

### Key Components
- **Cosign Integration**: Uses Sigstore/Cosign installer for keyless container signing
- **AWS Integration**: ECR (Elastic Container Registry) and EKS (Elastic Kubernetes Service) deployment
- **OIDC Authentication**: GitHub OIDC token for keyless signing via Fulcio
- **Admission Controller**: External component (configured out-of-band) that enforces signature verification
- **Dual Deployment**: Parallel signed/unsigned paths to demonstrate policy enforcement
- **Security Scanning**: Integration placeholder for Wiz CLI security scanning
- **Kubernetes Deployment**: Template-based manifest rendering using envsubst

## Workflow Operations

### Running Workflows
All workflows are triggered via `workflow_dispatch` (manual trigger) through GitHub Actions UI:

1. **cosign-keyless-wiz-demo** - Also triggers on version tags (v*.*.*)
2. **sign-and-deploy-eks** - Manual dispatch with image and cluster parameters
3. **ecr-copy-sign-deploy** - Manual dispatch with source image and ECR parameters

### Key Environment Variables
- `AWS_REGION` - AWS region for ECR/EKS operations
- `CLUSTER_NAME` - EKS cluster name for deployments
- `NAMESPACE` - Kubernetes namespace for deployments
- `IMAGE_REF` - Container image reference to sign/deploy
- `ECR_REPO` - ECR repository name

### Required Secrets and Variables
- `AWS_DEPLOY_ROLE_ARN` (repository variable) - IAM role for AWS operations
- `WIZ_CLIENT_ID`, `WIZ_CLIENT_SECRET`, `WIZ_TENANT_URL` (secrets) - Wiz security scanning
- `GITHUB_TOKEN` - Automatically provided for GHCR operations

## Cosign Operations

### Keyless Signing Process
```bash
# Install cosign
cosign install

# Sign with keyless (requires OIDC token)
COSIGN_YES=true cosign sign IMAGE_REF

# Verify signature
cosign verify \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp "^https://github.com/REPO/.*$" \
  IMAGE_REF
```

### Verification Requirements
- OIDC issuer must be GitHub Actions token endpoint
- Certificate identity must match repository pattern
- Signatures are stored in transparency log (Rekor)

## Kubernetes Deployment

### Manifest Templates
Workflows reference Kubernetes manifest templates in `kavedarr/deploy/eks/k8s/`:
- `namespace.yaml` - Namespace definition
- `deployment.yaml` - Application deployment
- `service.yaml` - Service definition

### Template Variables
Templates use environment variable substitution via `envsubst`:
- `$NAMESPACE` - Target namespace
- `$DEPLOYMENT_NAME` - Deployment name
- `$CONTAINER_NAME` - Container name in deployment
- `$IMAGE_REF` - Signed container image reference

## Demo Execution Strategy

### Testing Signed Images (Success Path)
1. Run any of the three existing workflows to deploy signed images
2. Admission controller allows deployment due to valid Cosign signatures
3. Pods start successfully, demonstrating policy compliance

### Testing Unsigned Images (Blocked Path)
1. Deploy identical images without Cosign signatures
2. Admission controller rejects deployment due to missing signatures
3. Deployment fails, demonstrating policy enforcement

### Local Development Commands
```bash
# Verify AWS and GitHub CLI are authenticated
aws sts get-caller-identity
gh auth status

# Manually trigger workflows
gh workflow run cosign-keyless-wiz-demo.yml
gh workflow run sign-and-deploy-eks.yml --input image_ref=IMAGE --input cluster_name=CLUSTER
gh workflow run ecr-copy-sign-deploy.yml --input source_image=IMAGE --input cluster_name=CLUSTER

# Check EKS cluster and admission controller status
kubectl get pods -n demo
kubectl get events -n demo
```

## Security Model

### Trust Chain
1. GitHub OIDC token provides identity
2. Fulcio issues short-lived certificate
3. Signature stored in Rekor transparency log
4. Admission controller verifies OIDC issuer and identity before allowing deployment

### Admission Control Enforcement
- **Policy**: Only signed images from trusted sources allowed
- **Verification**: Cosign signature validation at admission time
- **Enforcement**: Unsigned deployments rejected before pod creation

### Permissions Required
- `id-token: write` - For OIDC token generation
- `contents: read` - For repository access
- `packages: write` - For container registry push (GHCR workflow)

### AWS IAM Requirements
The AWS role must have permissions for:
- ECR repository creation and push/pull
- EKS cluster access and kubectl operations
- Kubernetes namespace and deployment management