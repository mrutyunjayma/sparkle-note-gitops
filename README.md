# Sparkle Note GitOps

This repository stores Kubernetes manifests for deploying the Sparkle Note application with a GitOps workflow.

Argo CD watches this repository and syncs the YAML files into the Kubernetes cluster. The repo is split into application manifests, shared infrastructure manifests, and Argo CD application definitions.

## Project Structure

```text
.
├── apps/
│   ├── backend/
│   └── frontend/
├── argocd/
│   └── applications/
└── infra/
    ├── cluster/
    ├── controllers/
    └── networking/
```

## Folder Overview

### `apps/`

Application-level Kubernetes resources live here.

#### `apps/backend/`

- `deployment.yaml`: Backend Deployment
- `service.yaml`: Backend Service
- `external-secrets.yaml`: ExternalSecret that pulls backend secrets from AWS Secrets Manager

#### `apps/frontend/`

- `deployment.yaml`: Frontend Deployment
- `service.yaml`: Frontend Service
- `configmap.yaml`: Frontend configuration values such as API URL

### `infra/`

Shared cluster resources and controller-related manifests live here.

#### `infra/cluster/`

- `namespace.yaml`: Creates required namespaces
- `cluster-secrets-store.yaml`: Defines a cluster-wide connection to AWS Secrets Manager

#### `infra/controllers/`

- `external-secrets/eso-serviceaccount.yaml`: ServiceAccount for External Secrets Operator using IRSA
- `alb/alb-serviceaccount.yaml`: ServiceAccount for AWS Load Balancer Controller using IRSA

#### `infra/networking/`

- `ingress.yaml`: Ingress resource for exposing the frontend through AWS ALB

### `argocd/`

Argo CD `Application` resources live here.

- `applications/backend.yaml`: Tells Argo CD to deploy `apps/backend`
- `applications/frontend.yaml`: Tells Argo CD to deploy `apps/frontend`

## How Deployment Works

1. Argo CD reads the manifests from this Git repository.
2. `argocd/applications/*.yaml` tells Argo CD which folder to sync.
3. Resources in `infra/` prepare shared cluster components such as namespaces, networking, and secret access.
4. Resources in `apps/` deploy the backend and frontend workloads.
5. Argo CD keeps the cluster aligned with what is stored in Git.

## External Secrets Flow

This repository uses External Secrets so secret values are not stored directly in Git.

Flow:

1. `infra/controllers/external-secrets/eso-serviceaccount.yaml` provides IAM access through IRSA.
2. `infra/cluster/cluster-secrets-store.yaml` defines how to connect to AWS Secrets Manager.
3. `apps/backend/external-secrets.yaml` requests a specific secret from AWS.
4. External Secrets Operator creates a Kubernetes `Secret`.
5. The backend Deployment reads that Kubernetes secret as an environment variable.

## AWS / EKS Notes

- Replace placeholder AWS account IDs with your real account ID.
- IRSA annotations must use a full IAM role ARN, for example:

```yaml
eks.amazonaws.com/role-arn: arn:aws:iam::055679187412:role/aws-load-balancer-controller-role
```

- These annotations must point to IAM roles, not IAM users.

## Useful Concepts

- `destination` in Argo CD: where Argo CD deploys the manifests
- `ignoreDifferences`: fields Argo CD should not treat as drift
- `ExternalSecret`: asks for secret data from an external provider
- `ClusterSecretStore`: shared provider configuration used by one or more ExternalSecrets

## Current Notes

Some manifests appear to need cleanup for label and naming consistency. For example, service selectors, deployment labels, and secret key names should all match exactly for the app to work correctly.
