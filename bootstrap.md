


#!/usr/bin/env bash
set -euo pipefail

### ---------- Config via ENV (kan overrides før kørsel) ----------
: "${CLIENT_ID:?Sæt CLIENT_ID i miljøet}"
: "${CLIENT_SECRET:?Sæt CLIENT_SECRET i miljøet}"
: "${CLUSTER_NAME:?Sæt CLUSTER_NAME i miljøet}"
: "${KUBECONFIG:?Sæt KUBECONFIG til din kubeconfig sti}"
: "${GITHUB_TOKEN:?Sæt GITHUB_TOKEN i miljøet til flux bootstrap}"

GITHUB_OWNER="${GITHUB_OWNER:-kbering}"
GITHUB_REPO="${GITHUB_REPO:-homelab2}"
GITHUB_BRANCH="${GITHUB_BRANCH:-main}"
GITHUB_PATH="${GITHUB_PATH:-./clusters/${CLUSTER_NAME}}"

### ---------- Preflight checks ----------
need() { command -v "$1" >/dev/null 2>&1 || { echo "Mangler $1 på PATH"; exit 1; }; }
need kubectl
need flux

if [ ! -d "${GITHUB_PATH}/flux-system" ]; then
  echo "❌ Forventer Flux manifests i ${GITHUB_PATH}/flux-system"
  exit 1
fi

echo "==> Kube context: $(kubectl config current-context || echo 'ukendt')"
kubectl get nodes -o wide >/dev/null

### ---------- External Secrets: namespace + Azure creds secret ----------
echo "==> Sikrer namespace 'external-secrets' findes"
kubectl get ns external-secrets >/dev/null 2>&1 || kubectl create namespace external-secrets

echo "==> Anvender/opfresher Secret 'azure-creds' i external-secrets"
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: azure-creds
  namespace: external-secrets
type: Opaque
stringData:
  ClientID: "${CLIENT_ID}"
  ClientSecret: "${CLIENT_SECRET}"
EOF

### ---------- Flux bootstrap til GitHub ----------
echo "==> Bootstrapper Flux til GitHub: ${GITHUB_OWNER}/${GITHUB_REPO} (${GITHUB_BRANCH}) path=${GITHUB_PATH}"
flux bootstrap github \
  --owner="${GITHUB_OWNER}" \
  --repository="${GITHUB_REPO}" \
  --branch="${GITHUB_BRANCH}" \
  --path="${GITHUB_PATH}" \
  --personal

### ---------- Seed repo-defined Kustomizations immediately ----------
echo "==> Anvender Flux manifests fra repoet"
kubectl apply -k "${GITHUB_PATH}/flux-system"

echo "==> Reconciler Flux source og root kustomization"
flux reconcile source git flux-system -n flux-system
flux reconcile kustomization flux-system -n flux-system --with-source || true

echo "==> Venter på Cilium"
kubectl -n kube-system rollout status ds/cilium --timeout=5m || true
kubectl -n kube-system get pods -l k8s-app=cilium

echo "==> Fjerner kube-flannel og kube-proxy efter Cilium er landet"
kubectl -n kube-system delete daemonset kube-flannel --ignore-not-found=true
kubectl -n kube-system delete daemonset kube-proxy --ignore-not-found=true

echo "==> Reconciler Cilium-config efter CRDs er klar"
flux reconcile kustomization cilium -n flux-system --with-source || true
flux reconcile kustomization cilium-config -n flux-system --with-source || true

echo "✅ Bootstrap færdig. Verificer med:"
echo "   kubectl get kustomizations -n flux-system"
echo "   kubectl get svc -A | grep LoadBalancer"
