# GitOps Infrastructure Config

This repository contains the Kubernetes manifests that deploy the Product Catalog API from the companion application repository: [dotnet-microservice-gitops](../dotnet-microservice-gitops/README.md).

The goal of this repo is to keep cluster configuration versioned, reviewable, and separate from application source code. It holds the runtime contract for how the service is deployed, exposed, and probed in Kubernetes.

## What is in this repo

The repository currently includes three core manifests:

- `deployment.yaml` defines the Product Catalog API Deployment.
- `service.yaml` defines the internal Kubernetes Service that routes traffic to the pods.
- `secret.yaml` stores sensitive configuration needed by the deployment.

## Deployment model

The Deployment runs the API as a replicated workload with rolling updates enabled.

Key characteristics:

- Deployment name: `product-catalog-api`
- Namespace: `default`
- Replica count: `3`
- Update strategy: `RollingUpdate`
- Container port: `8080`
- Image currently referenced: `shubd0cker/product-catalog-api:11`

The rollout settings are tuned for zero-downtime updates:

- `maxSurge: 1` allows one extra pod during an update.
- `maxUnavailable: 0` keeps all existing pods serving until replacements are healthy.

## Service model

The Service exposes the API internally inside the cluster.

- Service name: `product-catalog-service`
- Service type: `ClusterIP`
- Service port: `80`
- Target port: `8080`

Traffic is forwarded to pods labeled `app: product-catalog-api`.

## Health checks

The deployment depends on the application exposing the following endpoints:

- `GET /healthz/live` for liveness
- `GET /healthz/ready` for readiness

These match the .NET service implementation and should remain in sync with the app repository.

## Repository relationship

This repository is the GitOps companion to the application repository.

The intended flow is:

1. The application repository builds and publishes a new container image.
2. The image tag in `deployment.yaml` is updated here.
3. Kubernetes reconciles the manifest and performs the rollout.
4. The Service keeps routing cluster traffic to the updated pods.

Keeping these concerns separate makes it easier to review infrastructure changes independently from code changes.

## Applying the manifests

If you want to apply the resources manually, use `kubectl` from the repo root:

```bash
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

In a GitOps workflow, a controller such as Argo CD or Flux would normally watch this repository and apply changes automatically.

## Updating the image

When the application image changes, update the `image` field in `deployment.yaml` to the new tag, commit the change, and let the GitOps pipeline reconcile the cluster.

If the application port changes, update all of the following together:

- `deployment.yaml`
- `service.yaml`
- the Dockerfile in the application repository
- the application's health probe endpoints if needed

## Operational notes

- Keep manifest comments aligned with the actual runtime behavior.
- Keep secrets out of the Git history if this repo ever transitions to real production credentials.
- If you add ingress, config maps, or autoscaling later, document those resources here as well.
# gitops-infra-config