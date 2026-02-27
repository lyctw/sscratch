---
title: APLIC M-mode Interrupt Handling
tags:
  - aplic
  - m-mode
  - interrupts
---

# APLIC M-mode Interrupt Handling

This documents how external interrupts (e.g., UART hwirq=10) are caught by M-mode when using APLIC in direct delivery mode.

## Hardware Flow

```
UART RX data
     │
     ▼
┌─────────────────────────────────────┐
│ APLIC M-mode Domain                 │
│                                     │
│ SOURCECFG[9] = LEVEL_HIGH           │
│ TARGET[9] → hart 0, priority 1      │
│ IE bit enabled                      │
│ IDC[0].IDELIVERY = 1                │
└──────────────┬──────────────────────┘
               │
               ▼
        MIP.MEIP = 1 (hardware sets)
               │
               ▼
┌─────────────────────────────────────┐
│ CPU checks:                         │
│   MIE.MEIE = 1? ✓                   │
│   MSTATUS.MIE = 1? ✓                │
│        ↓                            │
│ Take M-mode trap                    │
│ MCAUSE = 0x8000_0000_0000_000B      │
│        (IRQ_M_EXT = 11)             │
└──────────────┬──────────────────────┘
               │
               ▼
         _trap_handler (MTVEC)
```

## CSR Configuration

| CSR | Bits | Value | Set By |
|-----|------|-------|--------|
| MTVEC | all | `_trap_handler` | `fw_base.S` |
| MIE | bit 11 (MEIE) | 1 | `sbi_irqchip_init()` |
| MSTATUS | bit 3 (MIE) | 1 | Runtime |
| MIDELEG | bits 1,5,9 | S-mode IRQs | `delegate_traps()` |
| MIP | bit 11 (MEIP) | Hardware | Read-only |

**Note**: M-mode external interrupts (MEIP) are **not delegatable** - they always trap to M-mode.

## APLIC Register Configuration

For hwirq=10 (UART), hart 0:

| Register | Offset | Value | Meaning |
|----------|--------|-------|---------|
| DOMAINCFG | 0x0000 | 0x101 | IE=1, DM=0 (direct), BE=1 |
| SOURCECFG[9] | 0x0028 | 0x6 | Level-high trigger |
| TARGET[9] | 0x3028 | 0x00000001 | Hart 0, priority 1 |
| SETIENUM | 0x1EDC | 10 | Enable interrupt 10 |
| IDC[0].IDELIVERY | 0x4000 | 1 | Delivery enabled |
| IDC[0].ITHRESHOLD | 0x4008 | 0 | All priorities |

## Trap Handler Chain

```
_trap_handler (asm)
     │
     ▼ save registers
sbi_trap_handler()
     │
     ▼ check mcause
sbi_trap_nonaia_irq(IRQ_M_EXT)
     │
     ▼
sbi_irqchip_process()
     │
     ▼ iterate irqchips
aplic_process_hwirqs()
     │
     ▼ read IDC.CLAIMI
aplic_hwirq_claim() → hwirq=10
     │
     ▼
sbi_irqchip_process_hwirq(chip, 10)
     │
     ▼
handler(10, opaque)  ← [[VIRQ Courier Handler]]
```

## Claiming Interrupts

```c
static u32 aplic_hwirq_claim(struct aplic_data *aplic)
{
    u32 hartindex = current_hartindex();
    unsigned long idc = aplic->addr + APLIC_IDC_BASE + 
                        hartindex * APLIC_IDC_SIZE;
    
    // Read CLAIMI - atomically claims highest priority pending IRQ
    u32 claimi = readl((void *)(idc + APLIC_IDC_CLAIMI));
    
    return (claimi >> APLIC_IDC_CLAIMI_INFO_SHIFT);  // Extract hwirq
}
```

## Related

- [[APLIC Integration]] - VIRQ registration
- [[VIRQ Layer Overview]] - Overall architecture

