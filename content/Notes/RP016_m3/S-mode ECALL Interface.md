---
title: S-mode ECALL Interface
tags:
  - virq
  - ecall
  - s-mode
---

# S-mode ECALL Interface

S-mode payloads (OP-TEE, Linux) interact with the VIRQ layer via SBI ecalls.

## ECALL Functions

### VIRQ Pop

Pop the next pending VIRQ for the current domain/hart.

```c
// S-mode
u32 sbi_ecall_virq_pop(void)
{
    struct sbiret ret = sbi_ecall(SBI_EXT_VIRQ, 
                                   SBI_EXT_VIRQ_POP,
                                   0, 0, 0, 0, 0, 0);
    return ret.value;
}
```

**Returns:** VIRQ number, or 0 if queue empty

### VIRQ Complete

Signal completion and unmask the underlying hwirq.

```c
// S-mode
void sbi_ecall_virq_complete(u32 virq)
{
    sbi_ecall(SBI_EXT_VIRQ,
              SBI_EXT_VIRQ_COMPLETE,
              virq, 0, 0, 0, 0, 0);
}
```

## ECALL Extension ID

```c
// In sbi_ecall_interface.h
#define SBI_EXT_VIRQ                0x56495251  // "VIRQ"
#define SBI_EXT_VIRQ_POP            0
#define SBI_EXT_VIRQ_COMPLETE       1
```

## M-mode Handler

```c
// In lib/sbi/sbi_ecall_virq.c

static int sbi_ecall_virq_handler(unsigned long extid,
                                   unsigned long funcid,
                                   struct sbi_trap_regs *regs)
{
    switch (funcid) {
    case SBI_EXT_VIRQ_POP:
        regs->a0 = sbi_virq_pop_thishart();
        return 0;

    case SBI_EXT_VIRQ_COMPLETE:
        sbi_virq_complete_thishart(regs->a0);
        return 0;

    default:
        return SBI_ENOTSUPP;
    }
}
```

## Usage Pattern

```c
// S-mode SSE handler
void virq_sse_handler(void)
{
    u32 virq;

    // Drain all pending VIRQs
    while ((virq = sbi_ecall_virq_pop()) != 0) {
        // Dispatch to device handler
        struct irq_desc *desc = irq_to_desc(virq);
        if (desc && desc->handler)
            desc->handler(virq, desc->data);

        // Unmask hwirq
        sbi_ecall_virq_complete(virq);
    }
}
```

## Security Considerations

- Each domain can only pop VIRQs from its own queue
- `sbi_domain_thishart_ptr()` ensures domain isolation
- Cannot complete VIRQs belonging to other domains

## File Location

- Header: `include/sbi/sbi_ecall_interface.h`
- Implementation: `lib/sbi/sbi_ecall_virq.c`

## Related

- [[SSE Injection]] - How S-mode is notified
- [[Per-Domain Queue]] - Queue implementation
- [[VIRQ Layer Overview]]

