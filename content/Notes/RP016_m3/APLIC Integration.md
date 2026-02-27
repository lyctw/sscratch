---
title: APLIC Integration
tags:
  - virq
  - aplic
  - integration
---

# APLIC Integration

The VIRQ layer integrates with the APLIC irqchip driver to intercept hardware interrupts in M-mode.

## Registration

In `lib/utils/irqchip/aplic.c`:

```c
#include <sbi/sbi_virq.h>

int aplic_cold_irqchip_init(...)
{
    // ... APLIC initialization ...

#ifndef APLIC_QEMU_WIRED_TEST
    if (aplic->targets_mmode) {
        static struct sbi_virq_courier_ctx aplic_virq_ctx;
        aplic_virq_ctx.chip = &aplic->irqchip;

        rc = sbi_irqchip_register_handler(
            &aplic->irqchip,
            1, aplic->num_source,        // hwirq range
            sbi_virq_courier_handler,    // handler
            &aplic_virq_ctx              // opaque context
        );
    }
#endif
}
```

## Conditional Compilation

| Mode | Behavior |
|------|----------|
| `APLIC_QEMU_WIRED_TEST` defined | Use test handler (M-mode UART echo) |
| Normal | Use VIRQ courier handler |

## Interrupt Flow with APLIC

```
APLIC hardware
     │
     ▼ MIP.MEIP set
┌────────────────────────────────────────────────┐
│ sbi_trap_handler()                             │
│     ↓                                          │
│ sbi_irqchip_process()                          │
│     ↓                                          │
│ aplic_process_hwirqs()                         │
│     ↓                                          │
│ aplic_hwirq_claim() → hwirq                    │
│     ↓                                          │
│ sbi_irqchip_process_hwirq(chip, hwirq)         │
│     ↓                                          │
│ sbi_virq_courier_handler(hwirq, ctx)  ◄────────│── VIRQ layer entry
│     ↓                                          │
│ [map, route, enqueue, switch, inject]          │
└────────────────────────────────────────────────┘
```

## APLIC Configuration for M-mode

The APLIC M-mode domain must be configured to deliver interrupts to M-mode:

| Register | Value | Purpose |
|----------|-------|---------|
| DOMAINCFG | `IE=1, DM=0` | Enable, direct mode |
| SOURCECFG[n] | `LEVEL_HIGH` | Trigger mode |
| TARGET[n] | `hart_idx << 18 | prio` | Route to hart |
| IDC.IDELIVERY | `1` | Enable delivery |
| IDC.ITHRESHOLD | `0` | No threshold |

See [[APLIC M-mode Interrupt Handling]] for details.

## Related

- [[VIRQ Courier Handler]] - The registered handler
- [[APLIC M-mode Interrupt Handling]] - APLIC configuration
- [[VIRQ Layer Overview]]

