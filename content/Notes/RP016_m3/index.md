---
title: OpenSBI VIRQ Layer Documentation
---

# OpenSBI VIRQ Layer

This documentation covers the Virtual IRQ (VIRQ) layer implementation for OpenSBI, designed for OP-TEE/Linux dual-domain interrupt routing.

## Overview

The VIRQ layer provides interrupt virtualization and routing between security domains (e.g., OP-TEE and Linux) in a RISC-V system running OpenSBI.

## Documentation

### Core Concepts
- [[VIRQ Layer Overview]] - High-level architecture and design goals
- [[Interrupt Routing Table]] - How hwirqs are mapped to domains

### Implementation Details
- [[VIRQ Mapping]] - Bidirectional (chip_uid, hwirq) ↔ VIRQ mapping
- [[VIRQ Courier Handler]] - M-mode interrupt routing logic
- [[Per-Domain Queue]] - Per-(domain, hart) pending VIRQ queues
- [[Domain Context Switch]] - Switching between security domains

### Integration
- [[APLIC Integration]] - How VIRQ integrates with APLIC irqchip
- [[SSE Injection]] - S-mode event notification mechanism
- [[S-mode ECALL Interface]] - Pop/complete VIRQs from S-mode

### Background
- [[APLIC M-mode Interrupt Handling]] - How M-mode catches external interrupts

## Quick Links

| Component | File | Description |
|-----------|------|-------------|
| Header | `include/sbi/sbi_virq.h` | Public API declarations |
| Implementation | `lib/sbi/sbi_virq.c` | Core VIRQ layer |
| ECALL Handler | `lib/sbi/sbi_ecall_virq.c` | S-mode interface |
| APLIC Integration | `lib/utils/irqchip/aplic.c` | Courier registration |

## Current Status

- ✅ VIRQ mapping (hwirq ↔ VIRQ)
- ✅ Routing rules infrastructure
- ✅ Per-domain queue management
- ✅ Courier handler with domain context switch
- ⏳ DT parsing for `opensbi,host-irqs` (hardcoded hwirq=10 for now)

