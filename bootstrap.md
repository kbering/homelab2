


#!/usr/bin/env bash
set -euo pipefail

### ---------- Config via ENV (kan overrides før kørsel) ----------
: "${CLIENT_ID:?Sæt CLIENT_ID i miljøet}"
: "${CLIENT_SECRET:?Sæt CLIENT_SECRET i miljøet}"
: "${CLUSTER_NAME:?Sæt CLUSTER_NAME i miljøet}"
: "${KUBECONFIG:?Sæt KUBECONFIG til din kubeconfig sti}"

# Cilium
CILIUM_VERSION="${CILIUM_VERSION:-1.18.0}"
CILIUM_NAMESPACE="${CILIUM_NAMESPACE:-kube-system}"
CILIUM_RELEASE_NAME="${CILIUM_RELEASE_NAME:-cilium}"

# Sti til dine Cilium værdier
CILIUM_VALUES_PATH="${CILIUM_VALUES_PATH:-infrastructure/controllers/prod/cilium/values.yaml}"

# Flux / GitHub
GITHUB_OWNER="${GITHUB_OWNER:-mischavandenburg}"
GITHUB_REPO="${GITHUB_REPO:-homelab}"
GITHUB_BRANCH="${GITHUB_BRANCH:-main}"
GITHUB_PATH="${GITHUB_PATH:-./clusters/${CLUSTER_NAME}}"
: "${GITHUB_TOKEN:?Sæt GITHUB_TOKEN i miljøet til flux bootstrap}"

### ---------- Preflight checks ----------
need() { command -v "$1" >/dev/null 2>&1 || { echo "Mangler $1 på PATH"; exit 1; }; }
need kubectl
need helm
need flux

if [ ! -f "$CILIUM_VALUES_PATH" ]; then
  echo "❌ Filen $CILIUM_VALUES_PATH findes ikke. Ret stien eller sæt CILIUM_VALUES_PATH."
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

### ---------- Cilium via Helm (upgrade --install) ----------
echo "==> Tilføjer/opfresher Cilium Helm repo"
helm repo add cilium https://helm.cilium.io >/dev/null 2>&1 || true
helm repo update >/dev/null

echo "==> Installerer/opgraderer Cilium ${CILIUM_VERSION} med værdier fra ${CILIUM_VALUES_PATH}"
helm upgrade --install "${CILIUM_RELEASE_NAME}" cilium/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace "${CILIUM_NAMESPACE}" --create-namespace \
  -f "${CILIUM_VALUES_PATH}"

echo "==> Venter på at Cilium pods bliver Ready"
kubectl -n "${CILIUM_NAMESPACE}" rollout status ds/cilium --timeout=3m || true
kubectl -n "${CILIUM_NAMESPACE}" get pods -l k8s-app=cilium


cilium install-cli

### ---------- Flux bootstrap til GitHub ----------
echo "==> Bootstrapper Flux til GitHub: ${GITHUB_OWNER}/${GITHUB_REPO} (${GITHUB_BRANCH}) path=${GITHUB_PATH}"
flux bootstrap github \
  --owner="${GITHUB_OWNER}" \
  --repository="${GITHUB_REPO}" \
  --branch="${GITHUB_BRANCH}" \
  --path="${GITHUB_PATH}" \
  --personal

echo "✅ Alt færdigt!"