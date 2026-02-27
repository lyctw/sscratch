---
title: VIRQ Mapping
tags:
  - virq
  - implementation
---

# VIRQ Mapping

The VIRQ mapping system provides bidirectional translation between physical hardware interrupts and virtual interrupt numbers.

## Mapping Model

```
(chip_uid, hwirq) ←→ VIRQ
```

- **chip_uid**: Unique identifier for the irqchip device
- **hwirq**: Hardware interrupt number (e.g., APLIC source ID)
- **VIRQ**: Virtual interrupt number exposed to S-mode

## Data Structures

### Forward Mapping: (chip_uid, hwirq) → VIRQ

```c
struct fwd_entry {
    u32 chip_uid;
    u32 hwirq;
    u32 virq;
};

static struct fwd_entry *fwd_vec;  // Dynamic vector
static u32 fwd_count;
```

Implementation: Linear search through vector (sufficient for small interrupt counts).

### Reverse Mapping: VIRQ → (chip_uid, hwirq)

```c
struct virq_entry {
    u32 chip_uid;
    u32 hwirq;
};

struct virq_chunk {
    struct virq_entry e[VIRQ_CHUNK_SIZE];  // 64 entries per chunk
};

static struct virq_chunk **rev_chunks;  // Chunked table
```

Implementation: O(1) lookup via `rev_chunks[virq >> 6]->e[virq & 63]`

### VIRQ Allocator

```c
static unsigned long *virq_bitmap;  // Bit = 1 means allocated
static u32 virq_bitmap_bits;
```

## Public APIs

| Function | Description |
|----------|-------------|
| `sbi_virq_map_init(cap)` | Initialize allocator with capacity |
| `sbi_virq_map_one(chip_uid, hwirq, ...)` | Create or get mapping |
| `sbi_virq_hwirq2virq(chip_uid, hwirq, out)` | Forward lookup |
| `sbi_virq_virq2hwirq(virq, out_chip, out_hwirq)` | Reverse lookup |
| `sbi_virq_unmap_one(virq)` | Free a mapping |
| `sbi_virq_map_uninit()` | Cleanup all mappings |

## Memory Usage

- Grows dynamically as mappings are added
- Chunks allocated on demand (64 entries each)
- Forward vector grows by 16 entries at a time

## Related

- [[VIRQ Layer Overview]]
- [[VIRQ Courier Handler]]

