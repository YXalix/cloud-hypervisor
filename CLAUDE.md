# VM Memory Snapshot Sharing via ublk Page Cache

## Project Overview

Implementation of a VM memory snapshot sharing system for secure container high-density deployment on ARM64 (Linux 6.6). The system uses ublk (userspace block device) page cache to share memory across multiple VMs, achieving ~79% memory savings.

**Design doc**: `/root/kernel/docs/design/vm-memory-snapshot-sharing-v2.rst`

## Architecture

### Core Mechanism

Multiple QEMU processes `mmap /dev/ublkbN` (MAP_PRIVATE) → share physical pages via inode `address_space` page cache → writes trigger COW (`wp_page_copy`) → shared pages unaffected.

### Snapshot Chain Model

```
V1 (base) ─── V2 (diff on V1) ─── V3 (diff on V2)
```

Each VM maps GPA ranges to different ublk devices based on which snapshot version owns the data for that range. Example VM-B (V2+V1):

```
GPA:  [0x0000_0000 ─── ublk_b0 (V1) ─── 0x2000_0000 ─── ublk_b1 (V2 diff) ─── 0x3000_0000 ─── ublk_b0 (V1) ─── 0x8000_0000]
```

### Two Operating Modes

1. **Static mode (basic)**: Pre-analyze snapshot chain → compute VMA layout → `mmap` + `MAP_FIXED` all regions before VM start. Zero runtime overhead, no dynamic changes.
2. **Dynamic mode (advanced, Chapter 11)**: Anonymous mapping + `uffd MISSING` → handler dynamically `mmap(MAP_FIXED, ublk_fd)` on first fault. Supports runtime snapshot switching and access tracing.
3. **Hybrid mode**: Hot regions (BIOS, kernel) use static mmap; remaining regions use uffd dynamic. Best of both worlds.

## Key Data Structures

```c
struct snapshot_region {
    uint64_t gpa_start;    /* guest physical address */
    uint64_t len;          /* region length */
    int      ublk_fd;      /* /dev/ublkbN fd */
    uint64_t file_offset;  /* offset within ublk device */
};

struct uffd_ctx {
    int uffd_fd;
    void *base;
    struct snapshot_region *regions;
    int num_regions;
};
```

## Implementation Phases

### Phase 1: Validation Prototype (1-2 weeks)
- Set up ublk devices with test data
- Verify multi-process mmap `/dev/ublkbN` shares page cache
- Verify MAP_PRIVATE COW isolation correctness
- Verify `mmap(MAP_FIXED)` VMA replacement
- Performance baselines: filemap_fault latency, COW overhead, sharing efficiency

### Phase 2: Integration (2-3 weeks)
- Snapshot chain analysis tool (page bitmap parsing + mapping table generation)
- QEMU memory initialization integration (`build_guest_ram`)
- Management plane integration (snapshot management + ublk device allocation)
- KVM end-to-end testing

### Phase 3: Production (2-3 weeks)
- Error handling and graceful degradation
- Prefetch strategy optimization
- Monitoring metrics (sharing ratio, fault latency)
- Huge page support (optional: hugetlbfs alternative)

## Critical Code Paths (Linux Kernel)

| Function | File | Line | Purpose |
|----------|------|------|---------|
| `blkdev_mmap` | `block/fops.c` | 845 | Block device mmap entry |
| `generic_file_mmap` | `mm/filemap.c` | 3824 | Installs `generic_file_vm_ops` |
| `filemap_fault` | `mm/filemap.c` | 3399 | File page fault → page cache lookup/fill |
| `filemap_map_pages` | `mm/filemap.c` | 3553 | Batch map existing page cache pages |
| `do_wp_page` | `mm/memory.c` | 3757 | Write protection fault → COW decision |
| `wp_page_copy` | `mm/memory.c` | 3425 | COW: alloc + copy + replace PTE |
| `__mmap_region` | `mm/mmap.c` | 2818 | `mmap MAP_FIXED` core |
| `__split_vma` | `mm/mmap.c` | 2482 | VMA split |
| `vma_merge` | `mm/mmap.c` | 937 | VMA merge (checks vm_file, uffd ctx) |
| `user_mem_abort` | `arch/arm64/kvm/mmu.c` | 1472 | ARM64 stage-2 fault handler |
| `__gfn_to_pfn_memslot` | `virt/kvm/kvm_main.c` | ~2600 | GPA→HVA→PFN conversion |

## Key Implementation Details

### Static Mode: VMA Layout Construction
1. `mmap(MAP_PRIVATE, ublk_v1_fd, 0)` as base covering entire guest RAM
2. For each V2+ diff region: `mmap(MAP_FIXED | MAP_PRIVATE, ublk_diff_fd, offset)` to replace
3. Different `vm_file` VMA segments don't merge (vma_merge checks vm_file pointer)

### Dynamic Mode: uffd Flow
1. `mmap(MAP_ANONYMOUS | MAP_PRIVATE)` → `uffd_register(MISSING mode)`
2. Fault → `handle_userfault()` → releases `mmap_lock` → sleeps
3. Handler: `read(uffd_fd)` → lookup region → `mmap(MAP_FIXED, ublk_fd)` → `UFFDIO_WAKE`
4. Faulting thread wakes → `VM_FAULT_RETRY` → `lock_mm_and_find_vma()` → finds new ublk VMA → `filemap_fault()`

### uffd Constraints
- `vma_can_userfault()` only allows anonymous/shmem/hugetlb VMA — NOT block device VMA
- That's why dynamic mode uses anonymous mapping first, then replaces with file VMA
- After MAP_FIXED replacement: new file VMA has NO uffd, subsequent faults go directly to `filemap_fault`
- `UFFDIO_WAKE` does NOT check VMA state — it only operates on wait queue

### VMA Fragmentation Control
- Always replace at region granularity (pre-merged contiguous same-source pages), never per-page
- Typical region count < 100, VMA count stays manageable
- Adjacent same-attribute VMA segments may be auto-merged by `vma_merge()`

### Prefetch Strategies
- `MADV_POPULATE_READ`: Synchronous, blocks until pages loaded (for hot regions like BIOS/kernel)
- `MADV_WILLNEED`: Async readahead hint, non-blocking
- `MADV_RANDOM`: Limit readahead for random access patterns

## Performance Targets

| Scenario | Latency |
|----------|---------|
| First read (page cache miss) | ~50-200us (static), ~100-300us (dynamic) |
| Subsequent read (page cache hit) | ~100ns (TLB miss) |
| First write (COW) | ~5-10us |
| Subsequent write | ~100ns |

## Monitoring & Debugging

- **Sharing efficiency**: `cat /proc/$(pidof qemu)/smaps` → `Shared_Clean / Total_RSS`
- **VMA layout**: `cat /proc/$(pidof qemu)/maps`
- **Page cache**: `fincore /dev/ublkbN`

## Error Handling

- `mmap(MAP_FIXED)` failure: Roll back previously replaced VMAs, or degrade to V1-only
- Page cache eviction: Increases latency but no correctness impact; mitigate with `mlock()` or `MADV_WILLNEED`
- ublk daemon crash: VM hangs; recover by restarting daemon or killing VM
- uffd handler must ALWAYS call `UFFDIO_WAKE`, even on error, to prevent faulting thread from permanent block

## Safety

- Isolation between VMs: MAP_PRIVATE COW, independent page tables, KVM stage-2 isolation
- Snapshot files should be opened read-only by ublk daemon
- Restrict ublk device access: `chmod 0600 /dev/ublkbN`
