# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a GitOps-managed Kubernetes homelab running on Talos Linux. The cluster uses Flux CD for continuous deployment from this repository.

## Architecture

**Core Infrastructure:**
- **OS**: Talos Linux (immutable Kubernetes OS)
- **CNI**: Cilium 1.16.6+ with kubeProxy replacement, eBPF datapath
- **Storage**: Longhorn 1.7.0 (distributed block storage)
- **GitOps**: Flux CD v2 (10-30 minute reconciliation intervals)
- **Secrets**: External Secrets Operator with Azure Key Vault backend

**Directory Structure:**
- `apps/` - Application deployments (Linkding, etc.)
- `clusters/prod/` - Production cluster configuration
  - `flux-system/` - Flux bootstrap and orchestration
  - `cilium/` - LB-IPAM pool and L2 announcements
  - `longhorn/` - Storage controller
  - `elastic/` - Elasticsearch 8.x cluster
- `infrastructure/controllers/` - Infrastructure controller definitions (Cilium Helm values)

## Common Commands

### Bootstrap a New Cluster

Set required environment variables:
```bash
export CLIENT_ID="..."           # Azure service principal
export CLIENT_SECRET="..."       # Azure service principal secret
export CLUSTER_NAME="prod"
export KUBECONFIG="~/.kube/config"
export GITHUB_TOKEN="..."        # GitHub PAT for Flux
export GITHUB_OWNER="kbering"
export GITHUB_REPO="homelab2"
```

Run bootstrap:
```bash
./bootstrap.md
```

### Flux Operations

```bash
# Force reconciliation
flux reconcile kustomization flux-system --with-source

# Check status
flux get kustomizations
flux get helmreleases -A

# View Flux logs
flux logs
```

### Verify Components

```bash
# Cilium status
cilium status
kubectl -n kube-system get pods -l k8s-app=cilium

# Longhorn status
kubectl -n longhorn-system get pods

# Elasticsearch
kubectl -n elastic get pods
kubectl -n elastic run -it es-shell --rm --image=curlimages/curl -- \
  curl -k -u 'elastic:PASSWORD' https://elasticsearch-master:9200/_license
```

### Encrypt Secrets with SOPS

```bash
sops --encrypt --in-place clusters/prod/elastic/secret-elastic-credentials.yaml
sops --encrypt --in-place clusters/prod/elastic/secret-es-license.yaml
```

## Key Configuration Files

- `infrastructure/controllers/prod/cilium/values.yaml` - Cilium CNI configuration
- `clusters/prod/cilium/ip-pool.yaml` - LoadBalancer IP pool (10.17.0.200/29)
- `clusters/prod/elastic/values-elasticsearch.yaml` - Elasticsearch settings (3 replicas, 50Gi storage)
- `clusters/prod/longhorn/values.yml` - Longhorn storage configuration

## Deployment Patterns

**All deployments follow GitOps:**
1. Edit YAML files in this repository
2. Commit and push to main branch
3. Flux automatically reconciles within 10-30 minutes
4. Or force with: `flux reconcile kustomization flux-system --with-source`

**HelmRelease pattern:** Charts are deployed via Flux HelmRelease CRs. Values are loaded from Kustomize-generated ConfigMaps.

**Secrets pattern:** Use ExternalSecret resources that reference Azure Key Vault via ClusterSecretStore.

## Network Configuration

- **LoadBalancer IPs**: 10.17.0.200/29 (managed by Cilium LB-IPAM)
- **L2 announcements** enabled for all LoadBalancer services
- **CiliumNetworkPolicies** protect sensitive components (Longhorn UI, etc.)

## Namespaces

- `kube-system` - Cilium CNI
- `longhorn-system` - Storage (privileged PSA)
- `elastic` - Elasticsearch cluster
- `linkding` - Applications
- `flux-system` - GitOps controllers
- `external-secrets` - Secret management
