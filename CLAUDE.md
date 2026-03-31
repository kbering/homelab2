# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a GitOps-managed Kubernetes homelab running on Talos Linux. The cluster uses Flux CD for continuous deployment from this repository.

## Architecture

**Core Infrastructure:**
- **OS**: Talos Linux (immutable Kubernetes OS)
- **CNI**: Cilium 1.16.6+ with kubeProxy replacement, eBPF datapath
- **Storage**: OpenEBS Mayastor 4.4.0 (distributed block storage)
- **GitOps**: Flux CD v2 (10-30 minute reconciliation intervals)
- **Secrets**: External Secrets Operator with Azure Key Vault backend

**Directory Structure:**
- `apps/` - Application deployments (Linkding, etc.)
- `clusters/prod/` - Production cluster configuration
  - `flux-system/` - Flux bootstrap and orchestration
  - `cilium/` - LB-IPAM pool and L2 announcements
  - `openebs/` - OpenEBS Helm release
  - `openebs-config/` - DiskPools, node labels, and init helpers
  - `elastic-stack/` - Elasticsearch, Kibana, Fleet
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

Recommended Talos patches before bootstrap:
```bash
# Apply to all nodes if you want Cilium-only networking from first boot
talosctl patch machineconfig --mode reboot --patch @talos-cilium-gitops-patch.yaml

# Apply to storage workers before OpenEBS/Mayastor install
talosctl patch machineconfig --mode reboot --patch @talos-openebs-storage-patch.yaml
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

# OpenEBS status
kubectl -n openebs get pods
kubectl -n openebs get diskpools.openebs.io

# Elasticsearch
kubectl -n elastic get pods
kubectl -n elastic run -it es-shell --rm --image=curlimages/curl -- \
  curl -k -u 'elastic:PASSWORD' https://elasticsearch-es-http:9200/_cluster/health
```

### Encrypt Secrets with SOPS

```bash
sops --encrypt --in-place clusters/prod/elastic/secret-elastic-credentials.yaml
sops --encrypt --in-place clusters/prod/elastic/secret-es-license.yaml
```

## Key Configuration Files

- `infrastructure/controllers/prod/cilium/values.yaml` - Cilium CNI configuration
- `clusters/prod/cilium/ip-pool.yaml` - LoadBalancer IP pool (10.17.0.200/29)
- `talos-cilium-gitops-patch.yaml` - Optional Talos patch to disable flannel/kube-proxy
- `talos-openebs-storage-patch.yaml` - Talos patch for hugepages and `/var/mnt/storage`
- `clusters/prod/openebs/helmrelease.yaml` - OpenEBS installation
- `clusters/prod/openebs-config/diskpools.yaml` - Mayastor DiskPool definitions
- `clusters/prod/elastic-stack/elasticsearch.yaml` - Elasticsearch settings

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
- **CiliumNetworkPolicies** can protect sensitive components

## Namespaces

- `kube-system` - Cilium CNI
- `openebs` - Storage (privileged PSA)
- `elastic` - Elasticsearch cluster
- `linkding` - Applications
- `flux-system` - GitOps controllers
- `external-secrets` - Secret management
