---
title: OP-TEE ARM vs RISC-V ABI Comparison
tags:
  - RISC-V
  - ARM
  - OP-TEE
  - OpenSBI
  - TF-A
date: 2026-03-08
---

## Overview

This note documents the observation that **ARM and RISC-V OP-TEE use identical internal ABI conventions** for returning from secure world to normal world. Both architectures place an internal "CALL_DONE" code in the first register, which must be stripped before returning results to Linux.

## Key Finding

| Aspect | ARM | RISC-V |
|--------|-----|--------|
| First register | x0/r0 | a0 |
| Internal code | `TEESMC_OPTEED_RETURN_CALL_DONE` | `TEEABI_OPTEED_RETURN_CALL_DONE` |
| Value | `0xBE000005` | `0xBE000005` |
| Actual results | x1-x4 | a1-a4 |
| Who strips? | **TF-A** (EL3) | **OpenSBI reqfwd** |

## Register Layout When OP-TEE Returns

```
OP-TEE sends (both ARM and RISC-V):
┌──────────────────────────────────────────────────────────┐
│ reg0 = 0xBE000005 (CALL_DONE)  ← Internal signal         │
│ reg1 = SMC return value        ← Actual result for Linux │
│ reg2 = return param 1                                    │
│ reg3 = return param 2                                    │
│ reg4 = return param 3                                    │
└──────────────────────────────────────────────────────────┘

Linux expects:
┌──────────────────────────────────────────────────────────┐
│ reg0 = SMC return value        ← Clean result            │
│ reg1 = return param 1                                    │
│ reg2 = return param 2                                    │
│ reg3 = return param 3                                    │
└──────────────────────────────────────────────────────────┘
```

## ARM: TF-A Handles the Stripping

### Source: `opteed_main.c`

```c
case TEESMC_OPTEED_RETURN_CALL_DONE:
    // x0 is dropped, x1-x4 become return values
    SMC_RET4(ns_cpu_context, x1, x2, x3, x4);
```

### Definition: `teesmc_opteed.h`

```c
/*
 * Issued when returning from "std_smc" or "fast_smc" vector
 * Register usage:
 *   r0/x0   SMC Function ID, TEESMC_OPTEED_RETURN_CALL_DONE
 *   r1-4/x1-4  Return value 0-3 passed to normal world in r0-3/x0-3
 */
#define TEESMC_OPTEED_FUNCID_RETURN_CALL_DONE  5
```

## RISC-V: OpenSBI reqfwd Handles the Stripping

### Source: `fdt_mpxy_reqfwd.c`

```c
// Transform callback skips a0, copies a1-a4
optee_transform_response(src, src_len, dst, dst_max, &dst_len);
```

### Definition: `teeabi_opteed.h`

```c
/*
 * Issued when returning from "std_abi" or "fast_abi" vector
 * Register usage:
 *   a0   ABI Function ID, TEEABI_OPTEED_RETURN_CALL_DONE
 *   a1-4 Return value 0-3 passed to non-secure domain in a0-3
 */
#define TEEABI_OPTEED_FUNCID_RETURN_CALL_DONE  5
```

## Flow Comparison

### ARM Flow

```
OP-TEE ──── SMC ────► TF-A (EL3)
                        │
x0 = CALL_DONE         │ case TEESMC_OPTEED_RETURN_CALL_DONE:
x1 = SMC result        │     SMC_RET4(ctx, x1, x2, x3, x4);
x2 = ret1              │              │
x3 = ret2              │              │ shift: x1→x0, x2→x1, ...
x4 = ret3              │              ▼
                        │     Linux receives x0-x3 (clean)
```

### RISC-V Flow

```
OP-TEE ── MPXY COMPLETE ──► OpenSBI reqfwd
                              │
a0 = CALL_DONE               │ transform callback:
a1 = SMC result              │     skip a0, copy a1-a4
a2 = ret1                    │              │
a3 = ret2                    │              ▼
a4 = ret3                    │     sender_rx[0]=a1, [1]=a2, ...
                              │              │
                              │              ▼
                              │     Linux receives a0-a3 (clean)
```

## Source Files

| Architecture | File | Purpose |
|--------------|------|---------|
| ARM | [`teesmc_opteed.h`](https://github.com/OP-TEE/optee_os/blob/master/core/arch/arm/include/sm/teesmc_opteed.h) | Define TEESMC_OPTEED_RETURN_* |
| ARM | [`opteed_main.c`](https://github.com/ARM-software/arm-trusted-firmware/blob/master/services/spd/opteed/opteed_main.c) | TF-A strips x0 |
| RISC-V | [`teeabi_opteed.h`](https://github.com/OP-TEE/optee_os/blob/master/core/arch/riscv/include/tee/teeabi_opteed.h) | Define TEEABI_OPTEED_RETURN_* |
| RISC-V | [`thread_optee_abi_rv.S`](https://github.com/OP-TEE/optee_os/blob/master/core/arch/riscv/kernel/thread_optee_abi_rv.S) | Set a0 = CALL_DONE |
| RISC-V | [`sbi_mpxy_rpmi.c`](https://github.com/OP-TEE/optee_os/blob/master/core/arch/riscv/kernel/sbi_mpxy_rpmi.c) | MPXY return path |
| RISC-V | [`fdt_mpxy_reqfwd.c`](https://github.com/riscv-software-src/opensbi/blob/master/lib/utils/mpxy/fdt_mpxy_reqfwd.c) | reqfwd strips a0 |
| RISC-V | [`optee_dispatcher.c`](https://github.com/riscv-software-src/opensbi/blob/master/lib/utils/tee/optee_dispatcher.c) | Transform callback |

## Why RISC-V Doesn't Handle All OPTEED Return Codes

### ARM TF-A Handles Many Return Codes

ARM TF-A's `opteed_main.c` handles **eight different return codes**:

| Return Code | Purpose | When Used |
|-------------|---------|-----------|
| `TEESMC_OPTEED_RETURN_ENTRY_DONE` | Cold boot initialization complete | Once at boot |
| `TEESMC_OPTEED_RETURN_ON_DONE` | CPU on (PSCI) complete | Secondary CPU boot |
| `TEESMC_OPTEED_RETURN_OFF_DONE` | CPU off (PSCI) complete | CPU hotplug off |
| `TEESMC_OPTEED_RETURN_SUSPEND_DONE` | CPU suspend (PSCI) complete | Sleep entry |
| `TEESMC_OPTEED_RETURN_RESUME_DONE` | CPU resume (PSCI) complete | Wake from sleep |
| `TEESMC_OPTEED_RETURN_SYSTEM_OFF_DONE` | System off complete | Shutdown |
| `TEESMC_OPTEED_RETURN_SYSTEM_RESET_DONE` | System reset complete | Reboot |
| `TEESMC_OPTEED_RETURN_CALL_DONE` | Normal SMC call complete | Every TEE call |

### ARM Vector Table (9 entries)

```asm
// From thread_optee_smc_a64.S
FUNC thread_vector_table
    b   vector_std_smc_entry        // slot 0: Std SMC
    b   vector_fast_smc_entry       // slot 1: Fast SMC
    b   vector_cpu_on_entry         // slot 2: CPU On
    b   vector_cpu_off_entry        // slot 3: CPU Off
    b   vector_cpu_resume_entry     // slot 4: Resume
    b   vector_cpu_suspend_entry    // slot 5: Suspend
    b   vector_fiq_entry            // slot 6: FIQ
    b   vector_system_off_entry     // slot 7: System Off
    b   vector_system_reset_entry   // slot 8: System Reset
END_FUNC thread_vector_table
```

### RISC-V Vector Table (Only 3 entries used)

```asm
// From thread_optee_abi_rv.S
FUNC thread_vector_table
    j   vector_std_abi_entry        // slot 0: Std ABI
    j   vector_fast_abi_entry       // slot 1: Fast ABI
    j   .                           // slot 2: (unused)
    j   .                           // slot 3: (unused)
    j   .                           // slot 4: (unused)
    j   .                           // slot 5: (unused)
    j   vector_fiq_entry            // slot 6: FIQ
    j   .                           // slot 7: (unused)
    j   .                           // slot 8: (unused)
END_FUNC thread_vector_table
```

### Why the Difference?

**The fundamental reason is architectural:**

#### ARM: SMC Trap-Based Model

On ARM, OP-TEE runs at S-EL1 (Secure EL1) and TF-A runs at EL3. **Every mode switch** between secure and non-secure world requires an SMC trap to EL3:

```
Linux (EL1) ──SMC──► TF-A (EL3) ──ERET──► OP-TEE (S-EL1)
                          ▲                    │
                          │      SMC trap      │
                          └────────────────────┘
```

When PSCI operations occur (cpu_on, cpu_off, suspend, resume), **TF-A orchestrates them** and calls into OP-TEE's vector table to notify it. OP-TEE then signals completion via these return codes.

#### RISC-V: MPXY Message-Based Model

On RISC-V, OP-TEE runs in a separate **security domain** and communication happens via RPMI/MPXY **messages**, not SMC traps:

```
Linux ──MPXY msg──► OpenSBI ──domain switch──► OP-TEE
                        ▲                          │
                        │     MPXY COMPLETE        │
                        └──────────────────────────┘
```

**Key differences:**

| Aspect | ARM | RISC-V |
|--------|-----|--------|
| Communication | SMC traps | MPXY messages |
| Power management | TF-A calls OP-TEE vectors | SBI HSM handles directly |
| CPU hotplug | TF-A notifies OP-TEE via vector | SBI HSM, OP-TEE not involved |
| System reset | TF-A notifies OP-TEE | SBI SRST, OP-TEE not involved |

### RISC-V Power Management is Separate

On RISC-V:

1. **SBI HSM extension** handles CPU on/off/suspend/resume directly
2. **SBI SRST extension** handles system reset/shutdown
3. **OP-TEE is not in the critical path** for power management

OP-TEE only needs to handle:
- `TEEABI_OPTEED_RETURN_ENTRY_DONE` - Boot complete (once)
- `TEEABI_OPTEED_RETURN_ON_DONE` - Secondary hart cold boot (once per hart)
- `TEEABI_OPTEED_RETURN_CALL_DONE` - Normal call complete (runtime)
- `TEEABI_OPTEED_RETURN_FIQ_DONE` - Interrupt handled (runtime)

The power management codes (`OFF_DONE`, `SUSPEND_DONE`, `RESUME_DONE`, etc.) are **defined but never used** in current RISC-V OP-TEE because those operations are handled by SBI, not by message proxying through OP-TEE.

### OpenSBI reqfwd Only Handles CALL_DONE

The OpenSBI reqfwd channel only processes `TEEABI_OPTEED_RETURN_CALL_DONE` because:

1. **ENTRY_DONE** and **ON_DONE** are handled at boot time via `sbi_domain_context_exit()`, not through reqfwd
2. **Power management** codes are not implemented on RISC-V OP-TEE
3. **FIQ_DONE** is handled separately in the interrupt path

The transform callback in `optee_dispatcher.c` simply strips a0 (the return code) regardless of its value - it doesn't need to dispatch based on the code like TF-A does.

## Conclusion

The internal ABI is **identical** between ARM and RISC-V OP-TEE. The only difference is **which component performs the register stripping**:

- **ARM**: TF-A (Trusted Firmware-A) at EL3
- **RISC-V**: OpenSBI reqfwd transform callback

This design allows the secure monitor to distinguish different return events (CALL_DONE, FIQ_DONE, ENTRY_DONE, etc.) while presenting clean SMC results to the normal world.

**However**, RISC-V's message-based architecture means OpenSBI doesn't need to handle the full set of PSCI-related return codes because RISC-V power management flows through SBI HSM/SRST extensions rather than through the TEE dispatcher.

## Related Notes

- [[rpmi-tee-design]]

