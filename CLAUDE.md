# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Knative PoC (Proof of Concept) on GKE — a blog article reference implementation demonstrating Knative Serving and Eventing on Google Kubernetes Engine, with monitoring via Prometheus/Grafana.

## Versions

- Knative Serving: v1.21.2 (with net-kourier v1.20.1 as ingress)
- Knative Eventing: v1.21.1
- GKE: v1.35.1 (Kubernetes)

## Infrastructure Setup (Terraform)

GKE cluster is provisioned via `terraform/main.tf` using the `terraform-google-modules/kubernetes-engine/google` module (~44.0). Single-zone, 1–3 nodes (e2-standard-2).

```bash
cd terraform
terraform init
terraform apply
```

## Knative Install Order

CRDs must be applied before core manifests. Always wait for CRDs to establish before proceeding.

```bash
# 1. Serving CRDs
kubectl apply -k k8s/kustomize/base/knative/crds/
kubectl wait --for=condition=Established --all crd --timeout=60s

# 2. Serving Core + Kourier (ingress-class patch applied automatically)
kubectl apply -k k8s/kustomize/base/knative/core/

# 3. Magic DNS via sslip.io
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.21.2/serving-default-domain.yaml

# 4. Eventing CRDs
kubectl apply -k k8s/kustomize/base/knative-eventing/crds/
kubectl wait --for=condition=Established --all crd --timeout=60s

# 5. Eventing Core
kubectl apply -k k8s/kustomize/base/knative-eventing/core/
```

## Deploying the Sample App

Base manifest (`k8s/kustomize/base/nginx-sample/ksvc.yaml`) uses `nginx:alpine`. Overlays swap the image tag for environment-specific builds (`gcr.io/PROJECT_ID/nginx-sample`).

```bash
# Base (nginx:alpine)
kubectl apply -k k8s/kustomize/base/nginx-sample/

# Dev overlay (gcr.io/PROJECT_ID/nginx-sample:dev)
kubectl apply -k k8s/kustomize/overlays/dev/

# Prod overlay (gcr.io/PROJECT_ID/nginx-sample:latest)
kubectl apply -k k8s/kustomize/overlays/prod/
```

Before using overlays, replace `PROJECT_ID` with your actual GCP project ID in the overlay `kustomization.yaml` files.

## Monitoring (Helmfile)

```bash
cd k8s

# Install kube-prometheus-stack (Prometheus + Grafana)
helmfile -e dev apply    # or prod
```

Base values in `helm-values/kube-prometheus-stack/base.yaml`; environment overrides in `dev.yaml` / `prod.yaml`. Grafana is ClusterIP by default — use `kubectl port-forward` to access.

## Architecture

Traffic flow: `External → svc/kourier (LoadBalancer) → 3scale-kourier-gateway → [activator if scaled-to-zero] → pod/nginx-sample`

Knative Serving creates 4 resource types per `ksvc`:
- **Service (ksvc)** — top-level manager
- **Configuration** — image template
- **Revision** — immutable snapshot per deploy
- **Route** — traffic split between revisions

Scale-to-zero is enabled by default. The activator buffers requests while pods spin up.

## Key Verification Commands

```bash
kubectl get pods -n knative-serving
kubectl get pods -n knative-eventing
kubectl get ksvc -n default
kubectl get revisions -n default
```
