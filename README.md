# encrypted_storage_pool

A generic storage-pool consumer for LUKS devices unlocked by
[`clevis_encryption`](https://github.com/alc-kit/clevis-encryption-role). It
assembles a **btrfs** or **LVM** pool on the `crypt-*` mappers, consumes
`clevis-luks-unlocked.target`, and emits `encrypted-storage-ready.target`.

It is the **filesystem-agnostic, non-ZFS** counterpart of proxmox-install's
`proxmox_encrypted_storage` role — same seam contract, mainline filesystems (no
DKMS). Its first job is to give `clevis_encryption`'s own test harness a real
downstream consumer to validate boot-time unlock + ordering without pulling ZFS
into clevis's CI.

## Where it fits

```
clevis_encryption  (NBDE)            LUKS2 + Clevis/Tang; opens crypt-* mappers
   └─ publishes  clevis-luks-unlocked.target   ← the seam (unlock has run)
        │
        ▼
encrypted_storage_pool  (this role)
   btrfs mkfs / LVM vgcreate → mount → write-probe check
   └─ publishes  encrypted-storage-ready.target  ← the barrier
```

## Boot chain

```
clevis-luks-unlocked.target             (from clevis_encryption)
  → encrypted-storage-assemble.service   btrfs device scan / vgchange -ay, then mount
  → encrypted-storage-check.service      mounted + writable → ok, else exit 1
  → encrypted-storage-ready.target       synchronization barrier (WantedBy=multi-user)
```

The filesystem's fstab entry is **noauto** so the early `local-fs` chain never
touches it — this late, network-ordered `assemble` unit owns the mount, avoiding
the systemd ordering cycle that ordering a filesystem after a network-bound unlock
would otherwise create.

`assemble` orders `Wants=`/`After=` the seam (not `Requires=`): the unlock is
fail-degraded, and fail-**closed** lives in the `check → ready` `Requires=` chain
(a partial unlock can still bring up a `-o degraded` btrfs raid1; the check gate
decides viability).

## Backends

| `encrypted_storage_pool_backend` | create | topology `mirror` | topology `stripe` |
|---|---|---|---|
| `btrfs` | `mkfs.btrfs` across the mappers | `-d raid1 -m raid1` | `-d single -m single` |
| `lvm` | pvcreate + vgcreate + lvcreate + `mkfs.ext4` | `--type raid1 -m1` | `--type striped -i N` |

## Role variables

| Variable | Default | Description |
|---|---|---|
| `encrypted_storage_pool_enabled` | `true` | Set `false` to make the role a no-op. |
| `encrypted_storage_pool_backend` | `btrfs` | `btrfs` or `lvm`. |
| `encrypted_storage_pool_name` | `data` | btrfs LABEL / LVM volume-group name. |
| `encrypted_storage_pool_topology` | `mirror` | `mirror` or `stripe`. |
| `encrypted_storage_pool_mountpoint` | `/srv/{{ name }}` | Mount point (fstab noauto). |
| `encrypted_storage_pool_devices` | *(derived)* | Bare disk names (e.g. `[vdb, vdc]`). Unset → derived from `crypt-*` in `/etc/crypttab`. |
| `encrypted_storage_pool_ensure` | `true` | Create/assemble the pool on every run. |
| `encrypted_storage_pool_install_packages` | `true` | Install `btrfs-progs` / `lvm2`. |
| `encrypted_storage_pool_destroy_existing` | `false` | **Destructive.** Destroy an existing pool first. |

## Usage

Runs after `clevis_encryption` (NBDE-only). Requires `clevis_encryption` **>= v1.2.0**
(publishes `clevis-luks-unlocked.target` + supports `clevis_deploy_storage_units`).

```yaml
roles:
  - role: clevis_encryption
    clevis_ensure_pool: false
    clevis_install_zfs_packages: false
    clevis_deploy_storage_units: false
  - role: encrypted_storage_pool
    encrypted_storage_pool_backend: btrfs   # or lvm
    encrypted_storage_pool_topology: mirror
```

## Testing

A `vm` Molecule scenario (Vagrant + libvirt/KVM) boots two VMs — an external Tang
server and an encrypted target — applies `clevis_encryption` (NBDE-only) + this
role, **reboots**, and asserts the real cross-role boot chain: mappers open,
`clevis-luks-unlocked.target` reached, seam ordering held
(`unlock ≤ unlocked.target ≤ assemble`), pool mounted, and real I/O round-trips.
The `vm-tests` CI workflow runs it for **both backends** (btrfs + lvm) via a
matrix (needs nested KVM). No DKMS — btrfs/lvm are mainline.

## License

MIT
