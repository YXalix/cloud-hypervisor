# Mmap Memory Restore

Restore VM memory by mapping a file directly into guest RAM with copy-on-write
semantics. Multiple VMs mapping the same file share physical pages via the Linux
page cache, reducing memory usage in high-density deployments.

## Usage

```bash
# Map from the snapshot's memory-ranges file
./cloud-hypervisor --api-socket /tmp/ch.sock \
    --restore "source_url=file:///snapshots/vm1,memory_restore_mode=mmap"

# Map from a custom file or device (e.g. ublk)
./cloud-hypervisor --api-socket /tmp/ch.sock \
    --restore "source_url=file:///snapshots/vm1,memory_restore_mode=mmap,mmap_file=/dev/ublkb0"

# Auto-resume after restore
./cloud-hypervisor --api-socket /tmp/ch.sock \
    --restore "source_url=file:///snapshots/vm1,memory_restore_mode=mmap,mmap_file=/dev/ublkb0,resume=true"
```

Two-step via API:

```bash
# Terminal 1
./cloud-hypervisor --api-socket /tmp/ch.sock

# Terminal 2
./ch-remote --api-socket=/tmp/ch.sock \
    restore source_url=file:///snapshots/vm1,memory_restore_mode=mmap,mmap_file=/dev/ublkb0
```

HTTP API:

```bash
curl -X PUT http://localhost/api/v1/vm/my-vm/restore \
    -H "Content-Type: application/json" \
    -d '{"source_url":"file:///snapshots/vm1","memory_restore_mode":"Mmap","mmap_file":"/dev/ublkb0","resume":true}'
```

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `source_url` | (required) | Snapshot directory URL |
| `memory_restore_mode` | `copy` | `copy`, `ondemand`, or `mmap` |
| `mmap_file` | snapshot's `memory-ranges` | File/device to mmap from |
| `resume` | `false` | Auto-resume after restore |
| `prefault` | `off` | **Incompatible with mmap mode** |

## How It Works

1. Guest RAM is allocated as anonymous memory.
2. Each saved memory region is replaced with `mmap(MAP_FIXED | MAP_PRIVATE)` backed by the file.
3. No data is copied at restore time -- pages load on first access via `filemap_fault`.
4. Writes trigger copy-on-write -- shared pages are unaffected.

## Restore Modes

| | `copy` | `ondemand` | `mmap` |
|---|---|---|---|
| Restore time | Slow | Fast | Fast |
| First access | None | ~100-300us | ~50-200us |
| Page sharing | No | No | Yes |
| Writes | N/A | Copy from snapshot | COW |

## Multi-VM Sharing

Multiple VMs mapping the same file share physical pages. Only written pages use extra memory.

```bash
# VM-1 and VM-2 share /dev/ublkb0 pages
./cloud-hypervisor --api-socket /tmp/ch-1.sock \
    --restore "source_url=file:///snap/base,memory_restore_mode=mmap,mmap_file=/dev/ublkb0,resume=true"
./cloud-hypervisor --api-socket /tmp/ch-2.sock \
    --restore "source_url=file:///snap/base,memory_restore_mode=mmap,mmap_file=/dev/ublkb0,resume=true"
```

## Monitoring

```bash
# Check shared vs private pages
cat /proc/$(pidof cloud-hypervisor)/smaps | grep -E "Shared_Clean|Shared_Dirty"

# Verify file-backed VMAs
cat /proc/$(pidof cloud-hypervisor)/maps
```
