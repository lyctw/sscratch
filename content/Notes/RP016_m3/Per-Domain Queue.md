---
title: Per-Domain Queue
tags:
  - virq
  - implementation
---

# Per-Domain Queue

Each (domain, hart) pair has a dedicated VIRQ pending queue. M-mode pushes VIRQs, S-mode pops and completes them.

## Data Structures

### Per-Hart State

```c
struct sbi_domain_virq_state {
    spinlock_t lock;
    u32 head;                        // Pop index
    u32 tail;                        // Push index
    u32 q[VIRQ_QSIZE];              // Ring buffer (32 entries)
    struct sbi_irqchip_device *chip; // For unmask on complete
};
```

### Per-Domain Private Data

```c
struct sbi_domain_virq_priv {
    u32 nharts;                      // Number of harts in domain
    struct sbi_domain_virq_state st[]; // Flexible array
};
```

## Queue Operations

### Enqueue (M-mode)

```c
int sbi_virq_enqueue(struct sbi_virq_courier_binding *c)
{
    st = virq_get_state(c->dom, current_hartindex());
    
    spin_lock(&st->lock);
    next_tail = (st->tail + 1) % VIRQ_QSIZE;
    if (next_tail == st->head) {
        // Queue full - drop interrupt
        return SBI_ENOMEM;
    }
    st->q[st->tail] = c->virq;
    st->tail = next_tail;
    st->chip = c->chip;
    spin_unlock(&st->lock);
    
    return SBI_OK;
}
```

### Pop (S-mode via ecall)

```c
u32 sbi_virq_pop_thishart(void)
{
    dom = sbi_domain_thishart_ptr();
    st = virq_get_state(dom, current_hartindex());
    
    spin_lock(&st->lock);
    if (st->head == st->tail) {
        // Queue empty
        return 0;
    }
    virq = st->q[st->head];
    st->head = (st->head + 1) % VIRQ_QSIZE;
    spin_unlock(&st->lock);
    
    return virq;
}
```

### Complete (S-mode via ecall)

```c
void sbi_virq_complete_thishart(u32 virq)
{
    // Reverse lookup to get hwirq
    sbi_virq_virq2hwirq(virq, &chip_uid, &hwirq);
    
    // Unmask the hwirq
    chip->hwirq_unmask(chip, hwirq);
}
```

## Queue Size

```c
#define VIRQ_QSIZE  32
```

- Current policy: **Drop** on overflow (return `SBI_ENOMEM`)
- Future: Could implement priority or expand dynamically

## Domain Data Integration

Uses OpenSBI's `sbi_domain_data` framework:

```c
static struct sbi_domain_data virq_domain_data = {
    .data_size = sizeof(struct sbi_domain_virq_priv) + 
                 sizeof(struct sbi_domain_virq_state) * MAX_HARTS,
    .data_setup = virq_domain_data_setup,
    .data_cleanup = virq_domain_data_cleanup,
};

// Register during init
sbi_domain_register_data(&virq_domain_data);
```

## Related

- [[VIRQ Courier Handler]] - Pushes to queue
- [[S-mode ECALL Interface]] - Pop/complete APIs
- [[VIRQ Layer Overview]]

