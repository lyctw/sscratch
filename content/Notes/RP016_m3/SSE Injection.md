---
title: SSE Injection
tags:
  - virq
  - sse
  - s-mode
---

# SSE Injection

SSE (Supervisor Software Event) is used to notify S-mode that there are pending VIRQs to process.

## What is SSE?

SSE is an SBI extension that allows M-mode to inject software events into S-mode. S-mode registers handlers for specific event types.

## Event Type Used

```c
SBI_SSE_EVENT_LOCAL_SOFTWARE
```

This is a per-hart event suitable for interrupt notification.

## Injection in VIRQ Layer

```c
int sbi_virq_courier_handler(u32 hwirq, void *opaque)
{
    // ... map, route, enqueue ...

    // Inject SSE event to notify S-mode
    rc = sbi_sse_inject_event(SBI_SSE_EVENT_LOCAL_SOFTWARE);
    if (rc) {
        sbi_printf("[VIRQ] SSE inject failed: %d\n", rc);
        // Not fatal - S-mode may not have registered handler yet
    }

    return SBI_OK;
}
```

## S-mode Handler (Expected Implementation)

```c
// In S-mode payload (OP-TEE or Linux)

void virq_sse_handler(void)
{
    u32 virq;

    // Pop all pending VIRQs
    while ((virq = sbi_ecall_virq_pop()) != 0) {
        // Find device handler
        struct irq_handler *h = find_handler(virq);
        if (h)
            h->callback(virq, h->priv);

        // Complete the VIRQ (unmasks hwirq)
        sbi_ecall_virq_complete(virq);
    }
}

// Register during init
sbi_sse_register(SBI_SSE_EVENT_LOCAL_SOFTWARE, virq_sse_handler);
```

## Alternative: Polling

If SSE is not available or not registered, S-mode can poll:

```c
// Polling approach (less efficient)
void timer_tick_handler(void)
{
    u32 virq;
    while ((virq = sbi_ecall_virq_pop()) != 0) {
        handle_virq(virq);
        sbi_ecall_virq_complete(virq);
    }
}
```

## SSE vs MIP.SEIP

| Mechanism | How it works | Pros | Cons |
|-----------|--------------|------|------|
| SSE | Inject software event | Structured, typed events | Requires SSE extension |
| MIP.SEIP | Set pending bit | Simple, hardware-like | No event data, just "something pending" |

Current implementation uses SSE for richer semantics.

## Related

- [[VIRQ Courier Handler]] - Injects SSE
- [[S-mode ECALL Interface]] - Pop/complete VIRQs
- [[Per-Domain Queue]] - Where VIRQs are queued

