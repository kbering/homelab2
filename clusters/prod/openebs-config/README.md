OpenEBS/Mayastor notes

- Storage workers must have `vm.nr_hugepages=1024`.
- Storage workers must mount the extra data disk on `/var/mnt/storage`.
- `mayastor-etcd-localpv` is patched to use `/var/mnt/storage/localpv-hostpath/etcd`.
- `mayastor-node-init` pre-creates the localpv base path and `diskpool.img` on each storage worker.
- Disk pools are currently file-backed AIO pools at `aio:///var/local/openebs/io-engine/diskpool.img`.
- `diskpools.yaml` and `storage-workers.yaml` use explicit node names and must be updated if Omni reprovisions workers with new names.
- For a faster fresh install, apply `/Users/bering/Dropbox/code/homelab2/talos-openebs-storage-patch.yaml` to the storage workers before Flux bootstrap.
- If you want Cilium-only networking from first boot, also apply `/Users/bering/Dropbox/code/homelab2/talos-cilium-gitops-patch.yaml` to the cluster before bootstrap.
