# Running on Talos Linux

This fork tracks upstream `SynologyOpenSource/synology-csi` closely. It adds an
Alpine-based image, a release pipeline that signs the image and Helm chart, and
chart options needed on Talos. Driver code is only patched for concrete,
reproducible bugs.

## Artifacts

| Artifact | Location |
|---|---|
| Image | `ghcr.io/yoramvandevelde/synology-csi:<tag>` (linux/amd64) |
| Helm chart | `oci://ghcr.io/yoramvandevelde/charts/synology-csi` |

Versioning:

- The chart has its own SemVer; release tags (`vX.Y.Z`) are chart versions.
- The chart's `appVersion` is the upstream driver version.
- Images are tagged `<appVersion>-rN`, where N increases with every release
  of the same driver version, plus a floating `<appVersion>` tag.

Both are signed with cosign (keyless, GitHub OIDC). The image also carries an
SBOM and provenance attestation. A released chart pins the image built for the
same tag by digest (`images.plugin.digest`).

Verify before installing:

```sh
cosign verify ghcr.io/yoramvandevelde/charts/synology-csi:<version> \
  --certificate-identity-regexp '^https://github.com/yoramvandevelde/synology-csi/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
cosign verify ghcr.io/yoramvandevelde/synology-csi:<appVersion>-rN \
  --certificate-identity-regexp '^https://github.com/yoramvandevelde/synology-csi/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

## Talos prerequisites

- The `siderolabs/iscsi-tools` system extension, added through an Image Factory
  schematic:

  ```yaml
  customization:
    systemExtensions:
      officialExtensions:
        - siderolabs/iscsi-tools
  ```

- `iscsiadm` at `/usr/local/sbin/iscsiadm`. Talos v1.12.5 moved it (fixed in
  v1.12.6), so avoid that release and check the path after upgrades:

  ```sh
  talosctl -n <node> ls /usr/local/sbin | grep iscsiadm
  ```

- The node plugin runs privileged, while Talos enforces the `baseline` Pod
  Security level by default. Label the namespace:

  ```sh
  kubectl create namespace synology-csi
  kubectl label namespace synology-csi pod-security.kubernetes.io/enforce=privileged
  ```

The node plugin runs `iscsiadm` on the host through `--chroot-dir=/host`.
Talos has no `/usr/bin/env`, so the chart must pass an absolute
`--iscsiadm-path`; the driver then skips its `env` wrapper. Formatting,
resizing and NFS/SMB mounts run inside the container, which ships the needed
tools.

## DSM preparation

1. Create a storage pool and volume (for example `/volume1`).
2. Enable SAN Manager for iSCSI, and optionally the NFS service (NFSv4.1).
3. Create a dedicated administrator account for the driver. The driver does not
   support 2FA, so this account must not have it enabled.
4. Use HTTPS (port 5001). With a self-signed DSM certificate, put the CA in
   `tlsCACert` in the client info secret.
5. Restrict TCP/3260 on the NAS firewall to the node IPs. Targets are created
   without CHAP, so the network is the only access control.

## Client info secret

The chart does not create the secret when `clientInfoSecret.create: false`.
Provide it with any mechanism (Sealed Secrets, SOPS, External Secrets, plain
kubectl). It needs one key, `client-info.yml`:

```yaml
clients:
  - host: <dsm-ip>
    port: 5001
    https: true
    username: <user>
    password: <password>
    # tlsCACert: |
    #   -----BEGIN CERTIFICATE-----
```

```sh
kubectl -n synology-csi create secret generic client-info-secret \
  --from-file=client-info.yml=./client-info.yml
```

## Install

Start from [`deploy/example/values-talos.yaml`](../deploy/example/values-talos.yaml)
and set the DSM address and volume location:

```sh
helm install synology-csi oci://ghcr.io/yoramvandevelde/charts/synology-csi \
  --version <version> --namespace synology-csi -f values-talos.yaml
```

## Validation

1. Create a PVC on the iSCSI StorageClass and a pod that mounts it; write a file.
2. Delete the pod and let it reschedule to another node (cordon the first one if
   needed); check the file is still there.
3. Increase the PVC size and check the filesystem grows inside the pod.
4. Delete the PVC and confirm the LUN and target are removed in DSM (with
   `reclaimPolicy: Delete`).

## Known limitations

- **No CHAP.** iSCSI targets are created without authentication.
- **No 2FA** on the DSM account used by the driver.
- **NFS export rules use each node's `InternalIP`.** Mounts fail when the source
  IP a node uses toward the NAS differs from its `InternalIP` (NAT, another VLAN
  interface). Check with a host network pod:
  `kubectl -n synology-csi debug node/<node> -it --image=alpine -- ip route get <dsm-ip>`.
- **NFS share path parsing is fragile** (`NFSTODO` in `setNFSVolumePrivilege`).
  Plain share names work.
- **Snapshots and clones need Btrfs** on the DSM volume. Disable the
  VolumeSnapshotClass otherwise.

If the driver's NFS support misbehaves, `kubernetes-csi/csi-driver-nfs` against
a manually created export is a simpler alternative for RWX volumes.
