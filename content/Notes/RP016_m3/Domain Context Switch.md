---
title: Domain Context Switch
tags:
  - virq
  - domain
  - context-switch
---

# Domain Context Switch

When an interrupt needs to be delivered to a different security domain than the currently running one, OpenSBI performs a domain context switch.

## When Context Switch Occurs

| Current Domain | Interrupt For | Action |
|----------------|---------------|--------|
| Linux | OP-TEE (secure) | Switch to OP-TEE |
| OP-TEE | Linux (normal) | Switch to OP-TEE (managed exit), then to Linux |
| Same domain | Same domain | No switch needed |

## API

```c
int sbi_domain_context_enter(struct sbi_domain *dom);
```

Defined in `lib/sbi/sbi_domain_context.c`.

## In VIRQ Courier Handler

```c
int sbi_virq_courier_handler(u32 hwirq, void *opaque)
{
    // ...
    target_dom = sbi_virq_route_lookup_domain(hwirq);
    current_dom = sbi_domain_thishart_ptr();

    if (target_dom != current_dom) {
        sbi_printf("[VIRQ] Domain switch: '%s' -> '%s'\n",
                   current_dom->name, target_dom->name);
        
        rc = sbi_domain_context_enter(target_dom);
        if (rc) {
            sbi_printf("[VIRQ] Context switch failed: %d\n", rc);
        }
    }
    // ...
}
```

## Context Save/Restore

The domain context includes:

- General purpose registers (x1-x31)
- CSRs: `sstatus`, `sepc`, `scause`, `stval`, etc.
- Floating point state (if enabled)
- Vector state (if enabled)

## Per-Hart Context Storage

```c
struct hart_context {
    // Saved register state
    unsigned long ra, sp, gp, tp;
    unsigned long s0-s11;
    // ... more registers ...
    
    // Saved CSRs
    unsigned long sstatus;
    unsigned long sepc;
    // ...
};

// Each domain has per-hart context storage
struct hart_context *contexts[MAX_HARTS];
```

## Managed Exit (Foreign IRQ)

When OP-TEE receives an interrupt meant for Linux:

1. M-mode routes to OP-TEE (trusted domain)
2. OP-TEE's SSE handler detects "foreign IRQ"
3. OP-TEE voluntarily exits to Linux
4. Linux handles the interrupt
5. Linux may call back into OP-TEE later

```
OP-TEE running
     │
     ▼ Normal IRQ arrives
M-mode VIRQ handler
     │
     ▼ Route to OP-TEE (always route to trusted first)
OP-TEE SSE handler
     │
     ▼ pop() returns VIRQ for Linux device
     │
     ▼ "Foreign IRQ" - not for me
OP-TEE does smc(OPTEE_SMC_RETURN_RPC)
     │
     ▼ Domain switch to Linux
Linux handles interrupt
```

## Related

- [[VIRQ Courier Handler]] - Triggers context switch
- [[VIRQ Layer Overview]] - Architecture
- [[Per-Domain Queue]] - Queue per domain

