---
title: VIRQ Layer Overview
tags:
  - virq
  - architecture
---

# VIRQ Layer Overview

The VIRQ (Virtual IRQ) layer is an abstraction that sits between the physical irqchip (APLIC) and domain-specific interrupt handlers. It enables interrupt routing between security domains like OP-TEE and Linux.

## Design Goals

1. **Domain Isolation**: Route interrupts to the correct security domain
2. **Transparent Virtualization**: S-mode sees VIRQs, not raw hwirqs
3. **Level-Trigger Safety**: Mask hwirqs during handling to prevent storms
4. **Flexible Routing**: Configuration via DeviceTree (future: `opensbi,host-irqs`)

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Hardware                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                   │
│  │  UART    │    │  GPIO    │    │  Timer   │                   │
│  │ hwirq=10 │    │ hwirq=15 │    │ hwirq=7  │                   │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘                   │
│       │               │               │                          │
│       └───────────────┼───────────────┘                          │
│                       ▼                                          │
│              ┌────────────────┐                                  │
│              │     APLIC      │  (M-mode domain)                 │
│              │  M-mode IDC    │                                  │
│              └───────┬────────┘                                  │
└──────────────────────┼──────────────────────────────────────────┘
                       │ MIP.MEIP
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                     M-mode (OpenSBI)                             │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                  VIRQ Courier Layer                         │ │
│  │                                                             │ │
│  │  1. Claim hwirq from APLIC                                  │ │
│  │  2. Map: (chip_uid, hwirq) → VIRQ                           │ │
│  │  3. Lookup destination domain                               │ │
│  │  4. Mask hwirq                                              │ │
│  │  5. Enqueue VIRQ to domain's queue                          │ │
│  │  6. Domain context switch (if needed)                       │ │
│  │  7. Inject SSE event                                        │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
                       │
         ┌─────────────┴─────────────┐
         ▼                           ▼
┌─────────────────┐         ┌─────────────────┐
│   OP-TEE (S)    │         │   Linux (S)     │
│  Trusted Domain │         │ Untrusted Domain│
│                 │         │                 │
│  SSE handler    │         │  SSE handler    │
│       │         │         │       │         │
│       ▼         │         │       ▼         │
│  ecall: pop()   │         │  ecall: pop()   │
│  handle VIRQ    │         │  handle VIRQ    │
│  ecall:complete │         │  ecall:complete │
└─────────────────┘         └─────────────────┘
```

## Key Components

| Component | Location | Purpose |
|-----------|----------|---------|
| [[VIRQ Mapping]] | `sbi_virq.c` | hwirq ↔ VIRQ translation |
| [[Interrupt Routing Table]] | `sbi_virq.c` | hwirq → domain rules |
| [[Per-Domain Queue]] | `sbi_virq.c` | Pending VIRQ queues |
| [[VIRQ Courier Handler]] | `sbi_virq.c` | Main routing logic |

## Related Documentation

- [[APLIC M-mode Interrupt Handling]] - How interrupts reach M-mode
- [[Domain Context Switch]] - Switching between domains
- [[SSE Injection]] - Notifying S-mode of pending VIRQs

