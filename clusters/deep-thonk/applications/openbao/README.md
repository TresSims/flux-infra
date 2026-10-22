# OpenBao on deep-thonk

OpenBao uses Helm chart `0.30.4` (OpenBao `2.7.1`) at
<https://openbao.k8s.deep-thonk.com>. The namespace, HelmRelease, Helm release,
StatefulSet, and services are named `openbao`; the data PVC is `data-openbao-0`.
The old `vault` namespace and hostname have been removed.

External Secrets in `o11y`, `tailscale`, and `technitium` use `openbao-store`.
ESO's provider field must still be named `vault` for API compatibility.
Each namespace's existing `approle-creds` Secret now uses `openbao-role-id` and
`openbao-role-secret` keys. The credential values, AppRoles, KV v2 mount, and
remote secret paths were preserved.

## Unsealing

The original 3-of-5 Shamir shares still apply. Run this command three times,
entering a different share at each hidden prompt:

```sh
kubectl --context admin@k8s-deep-thonk -n openbao exec -it openbao-0 -- \
  bao operator unseal
```

Do not initialize a new store. `bao status` should report `initialized: true`
and storage type `pebbledb`. Server updates use `OnDelete`, so chart upgrades
need a deliberate pod restart followed by unsealing.

## Migration record

The original deployment was Vault `1.21.2` with file storage on
`vault/data-vault-0`. OpenBao's
[Vault migration guide](https://openbao.org/docs/guides/migration/) tests older
Vault versions; retain the stopped-storage backups for rollback.

The cutover first converted 113 encrypted entries from file storage to
PebbleDB on the existing PVC, then renamed the deployment by rebinding its
retained PV to `openbao/data-openbao-0`. The backing volume remains
`pvc-e7b14290-42ab-446e-9b34-8f0e633c7aa2`; its reclaim policy is `Retain`.
The store is mounted at `/openbao/data/pebbledb`, and the original file store
is retained at `/openbao/data/legacy-file`.

The original cluster ID, `576d8d83-1b83-e8f0-4b0a-80a315d956d6`, was preserved.
The internal cluster display name remains the original generated
`vault-cluster-1f333981`; it is stored identity, not a Kubernetes resource name.

Two one-shot manifests are intentionally excluded from Kustomize:

- [`migration/data-pvc.yaml`](migration/data-pvc.yaml) prebinds the new PVC to
  the existing PV. Stop the old server, confirm `Retain`, take a backup, remove
  all pods mounting the old claim, and delete the old claim before rebinding
  the PV's `claimRef` to the new claim's namespace, name, and UID. Wait for
  `Bound` before installing OpenBao. Deleting a suspended HelmRelease may leave
  its Helm release installed; explicitly uninstall that old release before
  starting the replacement to avoid duplicate ingress and injector resources.
- [`migration/file-to-pebbledb.yaml`](migration/file-to-pebbledb.yaml) converts
  an **unmigrated** file store on that PVC. Run it only while the server is
  stopped. It refuses to overwrite an existing migration. Keep its pinned
  2.7 image: file-source migration support is scheduled for removal in 2.8.

The live migration is already complete; do not rerun these manifests.
See the [upstream storage migration guide](https://openbao.org/docs/guides/migrations/k8s-pebbledb/).

## Backups and rollback

Backups were saved outside Git at `/tmp/opencode/openbao-migration/backup/`:

- `vault-file-storage.tar.gz`: original stopped Vault file store.
- `openbao-pebbledb-pre-rename.tar.gz`: stopped OpenBao store before rebinding.
- Matching `.sha256` files, original Helm values/manifests, PV binding details,
  and pre-migration target-secret fingerprints.

Copy that directory to durable backup storage; `/tmp` is temporary. The retained
`legacy-file` directory is also a pre-cutover copy and is no longer updated.

To roll back, suspend reconciliation and stop OpenBao first. Preserve its current
data separately. To return to Vault, replace the volume contents with the original
file-store archive (do not overlay it on PebbleDB), rebind the retained PV to the
old claim if restoring the old namespace, and restore Vault chart `0.32.0` with
the saved configuration. Recreate the old credential-key aliases and SecretStores
before reverting ExternalSecret references. Unseal with the original shares and
verify fresh secret synchronization. Restoring an archive loses subsequent writes;
changing the image/chart alone does not reverse the storage migration.

## GitOps handoff

`applications` was suspended for the live migration. Keep it suspended until
this branch is merged to `main`, the Ship workflow publishes its OCI artifact,
and Flux fetches that artifact. The old artifact would reinstall Vault.

After verifying the OCI source revision contains this migration:

```sh
flux --context admin@k8s-deep-thonk reconcile source oci flux-system
flux --context admin@k8s-deep-thonk resume kustomization applications
```

Verify the OpenBao HelmRelease and all three SecretStores/ExternalSecrets are
Ready after reconciliation. Publish matching manifests before resuming
reconciliation following a rollback as well.
