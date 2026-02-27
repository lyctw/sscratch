---
title: VIRQ Courier Handler
tags:
  - virq
  - implementation
---

# VIRQ Courier Handler

The courier handler is the core routing function that processes incoming hardware interrupts and routes them to the appropriate security domain.

## Function Signature

```c
int sbi_virq_courier_handler(u32 hwirq, void *opaque);
```

**Parameters:**
- `hwirq`: Hardware interrupt number from APLIC claim
- `opaque`: Pointer to `struct sbi_virq_courier_ctx` containing irqchip device

**Returns:** `SBI_OK` on success, error code on failure

## Implementation Flow

```c
int sbi_virq_courier_handler(u32 hwirq, void *opaque)
{
    // Step 1: Map (chip_uid, hwirq) -> VIRQ
    rc = sbi_virq_map_one(chip->id, hwirq, false, 0, &virq);

    // Step 2: Lookup destination domain
    target_dom = sbi_virq_route_lookup_domain(hwirq);
    current_dom = sbi_domain_thishart_ptr();

    // Step 3: Mask hwirq (prevent level-trigger storm)
    chip->hwirq_mask(chip, hwirq);

    // Step 4: Enqueue VIRQ to target domain's queue
    binding.dom = target_dom;
    binding.chip = chip;
    binding.virq = virq;
    sbi_virq_enqueue(&binding);

    // Step 5: Domain context switch if needed
    if (target_dom != current_dom) {
        sbi_domain_context_enter(target_dom);
    }

    // Step 6: Inject SSE event
    sbi_sse_inject_event(SBI_SSE_EVENT_LOCAL_SOFTWARE);

    return SBI_OK;
}
```

## Routing Decision Table

| Interrupt Type | Current Domain | Action |
|----------------|----------------|--------|
| Secure (hwirq=10) | Linux | Context switch to OP-TEE, inject SSE |
| Secure (hwirq=10) | OP-TEE | Inject SSE (native IRQ) |
| Normal | Linux | Inject SSE (handled by Linux) |
| Normal | OP-TEE | Context switch to OP-TEE first (managed exit), then to Linux |

## Level-Trigger Handling

For level-triggered interrupts (like UART), the hwirq must be **masked** before returning from the handler. Otherwise:

1. Handler returns
2. Interrupt still asserted (UART RX FIFO not drained)
3. Immediate re-trap → infinite loop

The mask/unmask flow:

```
M-mode courier:     mask(hwirq)  ──────────────────────────┐
                         │                                  │
S-mode handler:          ▼                                  │
                    pop() → VIRQ                            │
                    handle (drain FIFO)                     │
                    complete(VIRQ) ─────────────────────────┤
                                                            ▼
M-mode complete:                                    unmask(hwirq)
```

## Context Structure

```c
struct sbi_virq_courier_ctx {
    struct sbi_irqchip_device *chip;  // For mask/unmask operations
};
```

Passed as `opaque` when registering with irqchip:

```c
// In aplic.c
static struct sbi_virq_courier_ctx aplic_virq_ctx;
aplic_virq_ctx.chip = &aplic->irqchip;

sbi_irqchip_register_handler(&aplic->irqchip,
                             1, aplic->num_source,
                             sbi_virq_courier_handler,
                             &aplic_virq_ctx);
```

## Related

- [[VIRQ Layer Overview]] - Architecture overview
- [[Per-Domain Queue]] - Queue management
- [[Domain Context Switch]] - How domain switching works
- [[APLIC Integration]] - Registration with APLIC

