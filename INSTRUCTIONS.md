# Handoff: synology-csi thin fork for Talos homelab

## Goal
Dynamic provisioning of 1-5 iSCSI LUNs (primary) and optionally NFS (secondary) from a
Synology DS216 into a Talos Kubernetes cluster, fully declarative after a one-time DSM bootstrap.

## Decisions
- Use upstream `SynologyOpenSource/synology-csi`, tag `v1.4.0`. Do NOT use
  `delta-whiplash/synology-csi-ng` (code quality rejected, single maintainer).
- Thin fork: upstream tag + own Dockerfile + Kustomize patches. No driver source patches
  unless a concrete, reproducible bug shows up in this setup.
- Build, sign (cosign) and SBOM the image in our own pipeline. Renovate on upstream tags.
- RWX fallback if synology-csi NFS misbehaves: `kubernetes-csi/csi-driver-nfs` against one
  manually bootstrapped export.

## Verified facts (from reading upstream v1.4.0 source)
- `main.go` already has `--chroot-dir` and `--iscsiadm-path` (also multipath/multipathd/nvme paths).
- `pkg/utils/hostexec/hostexec.go`: `wrapEnv` skips the `/usr/bin/env -i PATH=...` wrapper when
  the command contains `/`. Talos has no `/usr/bin/env`, so an absolute `--iscsiadm-path` is the fix.
  Config only, no code change.
- Upstream `Dockerfile` final stage is `ubi9/ubi-minimal` and installs e2fsprogs, xfsprogs,
  nfs-utils, cifs-utils, which are not in free UBI repos -> requires RHEL entitlement. This is
  why no official v1.4.0 image exists. Builder is `golang:1.21.4-alpine` (outdated).
- `mkfs` and NFS `mount` run inside the container (NodeStageVolume / NodePublishVolume), not via
  chroot -> the image must ship e2fsprogs, xfsprogs, util-linux, nfs-utils.
- NFS in v1.4.0 already has: retry on DSM error 2370 (`nfsPrivilegeRequest`, `pkg/dsm/webapi/share.go`)
  and a re-save of the share privilege + one mount retry when mount fails with
  "No such file or directory" (`pkg/driver/nodeserver.go`).
- NFS export rules are set to the `InternalIP` of ALL nodes (`getNodeAddress`, nodeserver.go).
  Breaks if the source IP toward the NAS differs from InternalIP (NAT, other VLAN interface).
- `setNFSVolumePrivilege` contains `NFSTODO: fix the parsing rule` on share path parsing. Fragile
  but fine with plain names.
- CHAP is not wired: iSCSI targets are created without auth.

## Environment constraints
- NAS: DS216, Armada 385 (32-bit ARM), 512MB RAM, single 1GbE, DSM 7.2. No Btrfs -> expect no
  LUN snapshots/clones. Features that matter: provision, delete, resize, mount.
- Talos needs the `siderolabs/iscsi-tools` extension (Image Factory schematic).
- Talos v1.12.5 regression moved iscsiadm out of `/usr/local/sbin` (fixed in v1.12.6).
  Avoid 1.12.5; verify path with `talosctl ls /usr/local/sbin/` after upgrades.
- DSM 2FA is not supported by the driver -> dedicated admin account for the CSI, no 2FA.

## One-time DSM bootstrap (manual, document in repo)
Storage pool + volume, SAN Manager enabled, CSI service account, (optional) NFS service enabled.

## Tasks
1. Replace Dockerfile final stage with Alpine or Wolfi: e2fsprogs xfsprogs util-linux iproute2
   bash ca-certificates nfs-utils cifs-utils nvme-cli. Bump Go builder to current. Keep
   `USER 1000` for controller; node DaemonSet stays privileged/root.
2. CI: multi-arch build (amd64 at minimum; check node arches), cosign sign, SBOM, push.
3. Kustomize overlay on `deploy/kubernetes` (or helm in `deploy/helm`):
   - node plugin args: `--chroot-dir=/host`, `--iscsiadm-path=/usr/local/sbin/iscsiadm`
   - image ref -> our registry, pinned by digest
   - client-info secret via existing secret management (SOPS/ExternalSecrets), not plaintext
   - namespace label `pod-security.kubernetes.io/enforce=privileged`
4. StorageClass iSCSI: `protocol: iscsi`, `dsm: <ip>`, `location: /volume1`, `fsType: ext4|xfs`,
   `allowVolumeExpansion: true`.
5. Optional NFS StorageClass: `protocol: nfs`, mountOptions `nfsvers=4.1`.
6. Network: restrict TCP/3260 on the NAS to node IPs (compensates for missing CHAP).
7. Validation: create PVC, mount, write, reschedule pod to other node, resize, delete PVC and
   confirm LUN/target removed in DSM.

## Open checks
- Does each Talos node's InternalIP equal its source IP toward the NAS?
- Node architectures for the image build.
- Current Talos version (not 1.12.5)
