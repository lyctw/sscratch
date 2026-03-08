---
title: RPMI TEE Service Group Design Analysis
tags:
  - RISC-V
  - OpenSBI
date: 2026-02-16
---

## Overview

This document describes the design of RPMI TEE Service Group and its integration
with REQUEST_FORWARD Service Group for OP-TEE on RISC-V.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   Service Group Relationship                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   TEE Service Group (0x000E)                                    │
│   └── For REE (Linux)                                           │
│       ├── TEE_GET_ATTRIBUTES: Query TEE attributes              │
│       └── TEE_COMMUNICATE: Send request to TEE                  │
│                                                                  │
│   REQUEST_FORWARD Service Group (0x000D)                        │
│   └── For TEE (OP-TEE)                                          │
│       ├── REQFWD_RETRIEVE: Get forwarded message                │
│       └── REQFWD_COMPLETE: Complete and respond                 │
│                                                                  │
│   ┌─────────────┐                    ┌─────────────┐            │
│   │   Linux     │    TEE SrvGrp      │   OP-TEE    │            │
│   │   (REE)     │ ───────────────►   │   (TEE)     │            │
│   │             │                    │             │            │
│   │             │   ReqFwd SrvGrp    │             │            │
│   │             │ ◄───────────────   │             │            │
│   └─────────────┘                    └─────────────┘            │
│         │                                   │                    │
│         │         ┌─────────────┐           │                    │
│         └────────►│   OpenSBI   │◄──────────┘                    │
│                   │  Dispatcher │                                │
│                   └─────────────┘                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Current Problem

### Linux Driver (conduit_riscv.c)

```c
// Request: 8 XLEN values (RV64 = 64 bytes)
struct mpxy_opteed_msg {
    unsigned long a0, a1, a2, a3, a4, a5, a6, a7;  // 8 * 8 = 64 bytes
};

// Response: 4 XLEN values (RV64 = 32 bytes)
struct mpxy_opteed_resp {
    unsigned long a0, a1, a2, a3;  // 4 * 8 = 32 bytes
};
```

**Issues:**
1. `unsigned long` is 4 bytes on RV32, 8 bytes on RV64
2. RPMI spec uses `uint32` (word) as unit
3. Current spec only says "implementation-specific" without clear definition

## Design Goals

1. **Generality**: Support OP-TEE and other TEE implementations
2. **Extensibility**: Support different request/response sizes
3. **RPMI Alignment**: Use word (uint32) as unit
4. **Explicit Sizing**: Return max request/response size in `TEE_GET_ATTRIBUTES`

## Proposed RPMI TEE Spec Changes

### Modified TEE_GET_ATTRIBUTES Response

```
.Response Data
| Word | Name           | Type   | Description                          |
|------|----------------|--------|--------------------------------------|
| 0    | STATUS         | int32  | Return error code                    |
| 1    | FLAGS          | uint32 | Bit[0]: XLEN (0=32-bit, 1=64-bit)   |
| 2    | TEE_IMPL_ID    | uint32 | TEE implementation identifier        |
| 3    | COMM_REQ_REGS  | uint32 | Number of XLEN-sized regs in request |
| 4    | COMM_RESP_REGS | uint32 | Number of XLEN-sized regs in response|
```

### TEE_COMMUNICATE Data Format

#### Request Data (OP-TEE Example)

For OP-TEE with COMM_REQ_REGS=8:

**XLEN=64 (RV64):**
- Words 0-1: a0 (little-endian 64-bit)
- Words 2-3: a1
- Words 4-5: a2
- Words 6-7: a3
- Words 8-9: a4
- Words 10-11: a5
- Words 12-13: a6
- Words 14-15: a7
- **Total: 16 words (64 bytes)**

**XLEN=32 (RV32):**
- Word 0: a0
- Word 1: a1
- ...
- Word 7: a7
- **Total: 8 words (32 bytes)**

#### Response Data (OP-TEE Example)

For OP-TEE with COMM_RESP_REGS=4:

**XLEN=64 (RV64):**
- Word 0: STATUS
- Words 1-2: a0
- Words 3-4: a1
- Words 5-6: a2
- Words 7-8: a3
- **Total: 9 words (36 bytes)**

**XLEN=32 (RV32):**
- Word 0: STATUS
- Word 1-4: a0-a3
- **Total: 5 words (20 bytes)**

## Size Summary

| Platform    | Request Size      | Response Size     |
|-------------|:-----------------:|:-----------------:|
| RV64 OP-TEE | 8×8 = **64 bytes**| 4×8 = **32 bytes**|
| RV32 OP-TEE | 8×4 = **32 bytes**| 4×4 = **16 bytes**|
| Other TEE   | `req_regs × XLEN` | `resp_regs × XLEN`|

## Complete Call Flow

```
═══════════════════════════════════════════════════════════════════
Boot Time
═══════════════════════════════════════════════════════════════════

(1) OpenSBI boots, initializes domains
    - trusted domain (OP-TEE)
    - untrusted domain (Linux)
    - Registers TEE MPXY channel (per-hart)
    - Registers ReqFwd MPXY channel (per-hart)

(2) OpenSBI jumps to OP-TEE (trusted domain, coldboot)

(3) OP-TEE finishes boot initialization
    ├── Sets up internal state
    └── Ready to receive requests

(4) OP-TEE calls REQFWD_RETRIEVE_CURRENT_MESSAGE
    │
    │   ECALL → OpenSBI
    │   ├── FIFO is empty
    │   ├── Mark: is_waiting_message = true
    │   ├── Save OP-TEE context
    │   └── sbi_domain_context_exit()
    │       → Switch to untrusted domain
    ▼
(5) Linux boots (untrusted domain)
    ├── Probes OP-TEE driver
    └── Gets MPXY TEE channel

═══════════════════════════════════════════════════════════════════
Runtime (repeating)
═══════════════════════════════════════════════════════════════════

(6) Linux: TEE_COMMUNICATE (a0-a7)
    │
    │   ECALL → OpenSBI OP-TEE Dispatcher
    │   ├── Find target ReqFwd channel
    │   ├── Wrap message: TEE srvgrp → ReqFwd FIFO
    │   │   ┌────────────────────────────────┐
    │   │   │ RPMI Header:                   │
    │   │   │   servicegroup_id = TEE (0xE)  │
    │   │   │   service_id = COMMUNICATE     │
    │   │   │   datalen = 64 (RV64)          │
    │   │   ├────────────────────────────────┤
    │   │   │ Data: a0-a7 (64 bytes)         │
    │   │   └────────────────────────────────┘
    │   ├── sbi_fifo_enqueue()
    │   ├── if (is_waiting_message):
    │   │     retrieve_message() → fill OP-TEE RX buffer
    │   ├── Save Linux context
    │   ├── Restore OP-TEE context
    │   └── MRET → OP-TEE domain
    ▼
(7) OP-TEE: Returns from REQFWD_RETRIEVE
    │
    │   RX buffer contains:
    │   ├── STATUS = SUCCESS
    │   ├── REMAINING = 0
    │   ├── RETURNED = 64
    │   └── REQUEST_MESSAGE = [TEE header + a0-a7]
    │
    ├── Parse RPMI header
    ├── Extract a0-a7
    └── Handle SMC request

(8) OP-TEE: Process request
    ├── Execute Trusted Application
    └── Prepare response (a0-a3)

(9) OP-TEE: REQFWD_COMPLETE_CURRENT_MESSAGE
    │
    │   TX contains: a0-a3 (response)
    │
    │   ECALL → OpenSBI
    │   ├── Copy response to sender's RX buffer
    │   ├── Clear current_msg
    │   └── Return SUCCESS
    ▼
(10) OP-TEE: REQFWD_RETRIEVE_CURRENT_MESSAGE (loop)
     │
     │   ECALL → OpenSBI
     │   ├── Check FIFO
     │   ├── If empty:
     │   │   ├── is_waiting_message = true
     │   │   ├── Save OP-TEE context
     │   │   └── sbi_domain_context_exit()
     │   │       → Switch back to Linux
     ▼
(11) Linux: Returns from TEE_COMMUNICATE
     │
     │   RX contains: a0-a3
     │
     └── Process response, continue...

→ Go to step (6)
```

## How ReqFwd Handles Variable Sizes

**Key: ReqFwd doesn't care about content format, only passes bytes!**

```c
struct rpmi_message_slot {
    struct rpmi_message_header header;  // 8 bytes
    //                                     ^^^^^^^^
    //                                     datalen records actual size
    u8 data[RPMI_MSG_DATA_SIZE(RPMI_SLOT_SIZE_MIN)];
    void *sender_rx;  // Address to write response back
};
```

### Flow:

1. **TEE Dispatcher receives TEE_COMMUNICATE:**
   ```c
   // tx_len = 64 (RV64, 8 registers) or 32 (RV32)
   header.datalen = cpu_to_le16(tx_len);
   header.servicegroup_id = cpu_to_le16(RPMI_SRVGRP_TEE);
   header.service_id = RPMI_TEE_SRV_COMMUNICATE;

   mpxy_reqfwd_forward_message(channel, &header,
                               tx, tx_len,
                               rx, rx_max_len,
                               &ack_len);
   ```

2. **ReqFwd stores message:**
   ```c
   // Save complete header (including datalen)
   sbi_memcpy(&msg.header, header, RPMI_MSG_HDR_SIZE);
   // Copy only tx_len bytes of data
   sbi_memcpy(msg.data, tx, tx_len);
   sbi_fifo_enqueue(&reqfwd->msg_fifo, &msg, true);
   ```

3. **OP-TEE retrieves message:**
   ```c
   // Get actual size from header.datalen
   datalen = le16_to_cpu(current_msg->header.datalen);
   // Copy to OP-TEE's RX buffer
   sbi_memcpy(&((u32 *)rx)[3], current_msg->data, datalen);
   *ack_len = 3 * sizeof(u32) + datalen;
   ```

4. **OP-TEE completes and responds:**
   ```c
   // tx_len determined by OP-TEE (response size)
   // e.g., RV64 OP-TEE: 4 * 8 = 32 bytes
   sbi_memcpy(current_msg->sender_rx, ..., tx_len);
   reqfwd->ack_len = tx_len;
   ```

## Proposed Code Changes

### 1. rpmi_msgprot.h Changes

```c
/** RPMI TEE ServiceGroup Service IDs */
enum rpmi_tee_service_id {
    RPMI_TEE_SRV_ENABLE_NOTIFICATION = 0x01,
    RPMI_TEE_SRV_GET_ATTRIBUTES = 0x02,
    RPMI_TEE_SRV_COMMUNICATE = 0x03,
    RPMI_TEE_SRV_MAX_COUNT,
};

/** TEE Implementation IDs */
enum rpmi_tee_impl_id {
    RPMI_TEE_IMPL_ID_OPTEE = 0x00000000,
    /* 0x00000001 - 0x7FFFFFFF: Reserved */
    /* 0x80000000 - 0xFFFFFFFF: Implementation Specific */
};

/** TEE Attributes Flags */
#define RPMI_TEE_FLAGS_XLEN_64    (1U << 0)

struct rpmi_tee_get_attributes_resp {
    s32 status;
    u32 flags;
    u32 tee_impl_id;
    u32 comm_req_regs;   /* Number of XLEN-sized regs in request */
    u32 comm_resp_regs;  /* Number of XLEN-sized regs in response */
};

/*
 * TEE_COMMUNICATE request/response are variable-size:
 *
 * Request size  = comm_req_regs * XLEN_BYTES
 * Response size = sizeof(status) + comm_resp_regs * XLEN_BYTES
 *
 * Where XLEN_BYTES = 4 (RV32) or 8 (RV64) based on FLAGS.XLEN_SIZE
 */
```

### 2. Linux Driver Changes (conduit_riscv.c)

```c
static void optee_riscv_sbi_mpxy(unsigned long a0, unsigned long a1,
                                 unsigned long a2, unsigned long a3,
                                 unsigned long a4, unsigned long a5,
                                 unsigned long a6, unsigned long a7,
                                 struct optee_conduit_res *res)
{
    unsigned long tx_regs[8] = {a0, a1, a2, a3, a4, a5, a6, a7};
    unsigned long rx_regs[4];
    struct rpmi_mbox_message msg = {0};
    int ret;

    /*
     * tx_size = 8 * sizeof(unsigned long)
     *         = 64 bytes (RV64) or 32 bytes (RV32)
     * rx_size = 4 * sizeof(unsigned long)
     *         = 32 bytes (RV64) or 16 bytes (RV32)
     */
    rpmi_mbox_init_send_with_response(&msg,
                                       RPMI_TEE_SRV_COMMUNICATE,
                                       tx_regs, sizeof(tx_regs),
                                       rx_regs, sizeof(rx_regs));

    ret = __mpxy_mbox_send_message(&msg);
    if (ret) {
        pr_err_ratelimited("TEE MPXY failed: %d\n", ret);
        return;
    }

    /* Copy response registers */
    res->a0 = rx_regs[0];
    res->a1 = rx_regs[1];
    res->a2 = rx_regs[2];
    res->a3 = rx_regs[3];
}
```

## Design Principles

1. **TEE_GET_ATTRIBUTES** returns `XLEN` size and register counts
2. **TEE_COMMUNICATE** request/response sizes are computed from above attributes
3. **ReqFwd** doesn't parse content, only uses `header.datalen` to pass data
4. All sizes are stored as word (uint32) units in RPMI messages

## Key Design Decisions

1. **TEE Service Group** is the generic REE→TEE interface
2. **ReqFwd** is OP-TEE specific implementation detail
3. Other TEEs may use different mechanisms (interrupts, polling, etc.)
4. ReqFwd API calls should be in `optee_dispatcher.c` since ReqFwd is mandatory
   only for OP-TEE dispatcher

## Comparison: ARM TrustZone vs RISC-V RPMI

| Feature           | ARM TrustZone        | RISC-V RPMI          |
|-------------------|----------------------|----------------------|
| Hardware Isolation| NS bit (bus level)   | PMP (M-mode control) |
| Secure Monitor    | EL3 (ATF/BL31)       | M-mode (OpenSBI)     |
| World Switch      | SMC instruction      | ECALL + Domain Switch|
| Protocol          | SMC Calling Conv.    | **RPMI** (standard)  |
| Memory Isolation  | TZASC, TZPC (HW)     | PMP (SW configured)  |
| Flexibility       | Fixed 2 worlds       | Multiple domains     |
| Standardization   | ARM proprietary      | **Open standard**    |

