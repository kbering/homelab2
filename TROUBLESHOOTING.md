# Troubleshooting Guide - Homelab2 Setup

Dette dokument beskriver de problemer vi stødte på under opsætningen og hvordan de blev løst.

## Problem 1: Longhorn CSI Pods CrashLoopBackOff

**Symptom:**
- Alle Longhorn CSI pods (attacher, provisioner, resizer, snapshotter, csi-plugin) i CrashLoopBackOff
- 3800+ restarts over flere dage
- Kunne ikke tilgå Kubernetes API: `dial tcp 10.96.0.1:443: i/o timeout`

**Root Cause:**
Cilium var konfigureret med `kubeProxyReplacement: true` men uden `socketLB.enabled: true`.
- Regular pods (ikke hostNetwork) kunne ikke nå Kubernetes services
- På Talos bruges `localhost:7445` proxy, men det virker kun for hostNetwork pods
- CSI pods skal kunne nå Kubernetes API via ClusterIP service (10.96.0.1:443)

**Løsning:**
```yaml
# infrastructure/controllers/prod/cilium/values.yaml
kubeProxyReplacement: true

# Enable socket LB for pods to reach Kubernetes services
socketLB:
  enabled: true
```

**Commands:**
```bash
# Upgrade Cilium
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
  -f infrastructure/controllers/prod/cilium/values.yaml

# Wait for Cilium rollout
kubectl -n kube-system rollout status ds/cilium --timeout=2m

# Restart failed Longhorn pods
kubectl delete pods -n longhorn-system -l app=csi-attacher
kubectl delete pods -n longhorn-system -l app=csi-provisioner
kubectl delete pods -n longhorn-system -l app=csi-resizer
kubectl delete pods -n longhorn-system -l app=csi-snapshotter
kubectl delete pods -n longhorn-system -l app=longhorn-csi-plugin
kubectl delete pods -n longhorn-system -l app=longhorn-manager
```

**Resultat:**
✅ Alle Longhorn CSI pods kører nu healthy
✅ 37/37 cluster pods managed by Cilium (før: 19/37)

---

## Problem 2: Longhorn Ingen Diske Tilgængelige

**Symptom:**
- Longhorn nodes havde `disks: {}`
- Ingen storage tilgængelig
- `diskStatus: {}`

**Root Cause 1: Manglende node labels**
Longhorn konfiguration: `createDefaultDiskLabeledNodes: true`
Ingen nodes havde `node.longhorn.io/create-default-disk=true` label

**Root Cause 2: Forkert disk path**
- Talos havde mounted diske på `/var/mnt/longhorn`
- Longhorn var konfigureret til `/mnt/disk1`
- Path mismatch: `open /mnt/disk1/longhorn-disk.cfg: no such file or directory`

**Løsning:**
```bash
# Verificer Talos disk mounts
talosctl get mounts --nodes talos-auk-e5p,talos-qip-m4e,talos-wlj-5t6 | grep longhorn
# Output: /dev/sdb1   /var/mnt/longhorn   xfs

# Patch Longhorn nodes med korrekt path
kubectl patch nodes.longhorn.io talos-auk-e5p -n longhorn-system --type=json \
  -p='[{"op": "replace", "path": "/spec/disks/disk-1/path", "value": "/var/mnt/longhorn"},
       {"op": "replace", "path": "/spec/disks/disk-1/allowScheduling", "value": true}]'

kubectl patch nodes.longhorn.io talos-qip-m4e -n longhorn-system --type=json \
  -p='[{"op": "replace", "path": "/spec/disks/disk-2/path", "value": "/var/mnt/longhorn"},
       {"op": "replace", "path": "/spec/disks/disk-2/allowScheduling", "value": true}]'

kubectl patch nodes.longhorn.io talos-wlj-5t6 -n longhorn-system --type=json \
  -p='[{"op": "replace", "path": "/spec/disks/disk-3/path", "value": "/var/mnt/longhorn"},
       {"op": "replace", "path": "/spec/disks/disk-3/allowScheduling", "value": true}]'
```

**Resultat:**
✅ 3 nodes med ~100GB storage hver (total ~300GB)
✅ Alle diske Ready og Schedulable

---

## Problem 3: Elastic ECK Deployment Fejl

**Symptom:**
```
kustomize build failed: may not add resource with an already registered id:
Namespace.v1.[noGrp]/elastic-system.[noNs]
```

**Root Cause:**
- Duplikerede namespace definitions
- Gamle `elastic-eck/` directory med kustomization.yaml
- Flux prøvede at validere Agent CRD før ECK operator var installeret
- `helmrepository-elastic.yaml` konflikt mellem flux-system og elastic-eck

**Løsning:**

1. **Split operator og stack i separate kustomizations:**
```
clusters/prod/
├── elastic-eck-operator/
│   ├── namespace.yaml          (elastic-system)
│   ├── helmrepository.yaml
│   ├── helmrelease.yaml        (ECK operator)
│   └── kustomization.yaml
└── elastic-stack/
    ├── elasticsearch.yaml
    ├── kibana.yaml
    ├── fleet-server.yaml       (disabled initially)
    └── kustomization.yaml
```

2. **Dependency chain:**
```yaml
# kustomization-elastic-stack.yaml
spec:
  dependsOn:
    - name: elastic-eck-operator
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: elastic-operator
      namespace: elastic-system
```

3. **Fjern duplikater:**
```bash
# Fjern gamle filer
rm -r clusters/prod/elastic-eck/
rm clusters/prod/flux-system/helmrepository-elastic.yaml
```

4. **Manuel deployment (workaround for Flux validation issue):**
```bash
# Apply operator først
kubectl apply -f clusters/prod/flux-system/kustomization-elastic-eck-operator.yaml

# Vent på operator er klar
kubectl wait --for=condition=ready pod -l control-plane=elastic-operator \
  -n elastic-system --timeout=300s

# Apply stack
kubectl apply -f clusters/prod/flux-system/kustomization-elastic-stack.yaml
```

**Resultat:**
✅ ECK operator deployed
✅ Elasticsearch cluster (3 nodes, green, 9.2.1)
✅ Kibana (green, LoadBalancer)
✅ 3x 50GB Longhorn PVCs

---

## Problem 4: Network Access til Services

**Symptom:**
- Longhorn UI og Hubble UI kun tilgængelige via port-forward
- Ingen LoadBalancer IPs

**Løsning:**
Opret LoadBalancer services med Cilium LB-IPAM:

```yaml
# Longhorn UI
apiVersion: v1
kind: Service
metadata:
  name: longhorn-frontend-lb
  namespace: longhorn-system
spec:
  type: LoadBalancer
  selector:
    app: longhorn-ui
  ports:
    - port: 80
      targetPort: 8000

# Hubble UI
apiVersion: v1
kind: Service
metadata:
  name: hubble-ui-lb
  namespace: kube-system
spec:
  type: LoadBalancer
  selector:
    k8s-app: hubble-ui
  ports:
    - port: 80
      targetPort: 8081
```

**Resultat:**
- Longhorn UI: http://10.17.0.201
- Hubble UI: http://10.17.0.202
- Kibana: http://10.17.0.203:5601

---

## Lessons Learned

### 1. Cilium Kube-Proxy Replacement
Når `kubeProxyReplacement: true`:
- **SKAL** have `socketLB.enabled: true` for regular pods
- `localhost:7445` proxy virker KUN for hostNetwork pods på Talos
- Verificer med: `cilium config view | grep bpf-lb-sock`

### 2. Longhorn på Talos
- Brug Talos machine config patches til disk mounting
- Verificer mount path med `talosctl get mounts`
- Patch Longhorn nodes med korrekt path
- Enable `allowScheduling: true`

### 3. Flux + ECK Deployment
- Split operator og stack i separate kustomizations
- Brug `dependsOn` og `healthChecks` for dependencies
- Undgå duplikerede namespace definitions
- Manuel apply kan være nødvendig for CRD validation issues

### 4. GitOps Structure
```
clusters/prod/
├── flux-system/
│   ├── kustomization.yaml                      (main)
│   ├── kustomization-longhorn.yaml             (→ longhorn/)
│   ├── kustomization-cilium-config.yaml        (→ cilium/)
│   ├── kustomization-elastic-eck-operator.yaml (→ elastic-eck-operator/)
│   └── kustomization-elastic-stack.yaml        (→ elastic-stack/)
├── longhorn/
├── cilium/
├── elastic-eck-operator/
└── elastic-stack/
```

---

## Verification Commands

```bash
# Cilium status
cilium status
kubectl -n kube-system get pods -l k8s-app=cilium

# Longhorn status
kubectl get nodes.longhorn.io -n longhorn-system
kubectl get pods -n longhorn-system

# Elastic status
kubectl get elasticsearch -n elastic
kubectl get kibana -n elastic
kubectl get pods -n elastic

# LoadBalancer IPs
kubectl get svc --all-namespaces --field-selector spec.type=LoadBalancer

# Flux status
flux get kustomizations
flux get helmreleases -A
```

---

## Current Working State

| Service | Status | Access | Storage |
|---------|--------|--------|---------|
| Cilium | ✅ Running | N/A | N/A |
| Longhorn | ✅ Healthy | http://10.17.0.201 | ~300GB (3x100GB) |
| Hubble UI | ✅ Running | http://10.17.0.202 | N/A |
| Elasticsearch | ✅ Green (3 nodes) | Internal | 150GB (3x50GB) |
| Kibana | ✅ Green | http://10.17.0.203:5601 | N/A |
| ECK Operator | ✅ Running | N/A | N/A |

**Credentials:**
- Kibana: `elastic` / (se secret: `elasticsearch-es-elastic-user`)

---

## Problem 5: Fleet Server Image Not Found (version 9.2.1)

**Symptom:**
```
Failed to pull image "docker.elastic.co/beats/elastic-agent:9.2.1": not found
ImagePullBackOff
```

**Root Cause:**
ECK operator bruger det gamle registry path `docker.elastic.co/beats/elastic-agent`, men Elastic Agent version 9.x er flyttet til det nye path: `docker.elastic.co/elastic-agent/elastic-agent`

**Løsning:**
Tilføj explicit `image` field i Agent spec for at override default image path:

```yaml
apiVersion: agent.k8s.elastic.co/v1alpha1
kind: Agent
metadata:
  name: fleet-server
  namespace: elastic
spec:
  version: 9.2.1
  image: docker.elastic.co/elastic-agent/elastic-agent:9.2.1  # ✅ Korrekt registry path
  kibanaRef:
    name: kibana
  elasticsearchRefs:
  - name: elasticsearch
  mode: fleet
  fleetServerEnabled: true
  policyID: fleet-server-policy
```

**Resultat:**
✅ Fleet Server kan nu hente korrekt image
