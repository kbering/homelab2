# Elasticsearch via Flux + Helm

This bundle contains manifests to deploy Elasticsearch with Helm under GitOps (Flux), including:
- Namespace
- HelmRepository for Elastic charts
- HelmRelease for Elasticsearch
- Values (as ConfigMap via Kustomize generator)
- Secrets for bootstrap credentials and Enterprise license (recommend SOPS encryption)
- A Job to upload the license after the cluster is ready

## How to use

1. Copy the folder `clusters/prod/elastic/` into your repository.
2. Replace secrets:
   - Edit `secret-elastic-credentials.yaml` and set a strong password.
   - Replace `secret-es-license.yaml` content with your official license JSON.
3. (Recommended) Encrypt secrets with SOPS:
   ```bash
   sops --encrypt --in-place clusters/prod/elastic/secret-elastic-credentials.yaml
   sops --encrypt --in-place clusters/prod/elastic/secret-es-license.yaml
   ```
4. Commit and push. Reconcile Flux:
   ```bash
   flux reconcile kustomization flux-system --with-source
   ```

Verify:
```bash
kubectl -n elastic get pods
kubectl -n elastic run -it es-shell --rm --image=curlimages/curl --   curl -k -u 'elastic:YOUR_PASSWORD' https://elasticsearch-master:9200/_license
```
