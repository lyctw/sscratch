---
title: OpenSBI VIRQ (Virtual IRQ) Layer 架構解析
tags:
  - RISC-V
  - OpenSBI
date: 2026-02-16
---

RISE RP016 (OpenSBI Extensions for TEEs) 由 RISCStar 正在進行的工作項目是 APLIC direct mode 系統上實現 Supervisor domain 特定的 interrupt (注意這邊的 domain 不是 APLIC domain, 而是一個 hart 的兩個 S-mode context e.g. Linux/OP-TEE OS), 目前最新的 patch: https://patchwork.ozlabs.org/project/opensbi/cover/20260213190459.2540597-1-raymondmaoca@gmail.com/

Use case:
- 能夠處理某個 trusted device 的 secure interrupt in OP-TEE
- 異質多核心 e.g. hart0 RTOS + hart1,2,3,4 Linux 各自專屬的 interrupt handling

實作這樣的架構除了不把 interrupt delegate 到 S-mode domain APLIC 還有幾個問題需要解決：

1. M-mode 攔到 interrupt 後，如何轉送給 S-mode? 目前計畫使用 SSE Inject
2. Arm GIC [^1] 能區分 FIQ/IRQ 並將中斷導向 secure/normal world, 對 RISC-V CPU 來說只有一種中斷訊號 mip.MEIP, 解決方法：DT 註明哪個 S-mode domain 負責 handle:

```
opensbi-domains {
    domain_a {
        opensbi,host-irqs = <10 5>;  /* IRQ 10-14 -> domain_a */
    };
    domain_b {
        opensbi,host-irqs = <20 10>; /* IRQ 20-29 -> domain_b */
    };
};
```
OpenSBI 已經提供 `sbi_domain_context_enter()` API 可指定下一次 mret 回到哪個 S-mode context

3. S-mode 收到 M-mode 轉送來的 IRQ 如何 claim? S-mode 讀 IDC.claimi 不會有效因為 IRQ 沒有 delegate 到這個 S-mode APLIC. 目前計畫使用 RPMI SystemIRQ service group[^2] 透過半虛擬方式取得 IRQ [^3]
4. 即使 S-mode 成功 claim 並開始服務這個中斷，完全處理完之前隨時可能被 M-mode interrupt 打斷，所以 M-mode 需要實作 irq queue 來紀錄 incoming pending interrupts

[^1]: https://optee.readthedocs.io/en/latest/architecture/core.html#native-and-foreign-interrupts
[^2]: https://github.com/riscv-non-isa/riscv-rpmi/pull/150/changes
[^3]: https://lore.kernel.org/all/CAK9=C2XzBFTouGyJ-PCgJmfiq+8Sji+FW=uiuwT9R+Fwg7KtWg@mail.gmail.com/

## 整體架構

```
┌────────────────────────────────────────────────────────────────────┐
│                     VIRQ Layer Architecture                        │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    S-mode Domains                            │  │
│  │                                                              │  │
│  │  ┌─────────────┐        ┌─────────────┐        ┌───────────┐ │  │
│  │  │  Domain A   │        │  Domain B   │        │   Root    │ │  │
│  │  │  (Linux)    │        │  (RTOS)     │        │  Domain   │ │  │
│  │  │             │        │             │        │           │ │  │
│  │  │ per-hart    │        │ per-hart    │        │ per-hart  │ │  │
│  │  │ VIRQ queue  │        │ VIRQ queue  │        │ VIRQ queue│ │  │
│  │  └──────┬──────┘        └──────┬──────┘        └─────┬─────┘ │  │
│  └─────────┼──────────────────────┼─────────────────────┼───────┘  │
│            │ ecall pop/complete   │                     │          │
│            ▼                      ▼                     ▼          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    M-mode OpenSBI                            │  │
│  │                                                              │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │                   VIRQ Layer                            │ │  │
│  │  │                                                         │ │  │
│  │  │  1. VIRQ Mapping:   (chip_uid, hwirq) <──> VIRQ         │ │  │
│  │  │  2. Route Rules:    HWIRQ ──> Domain                    │ │  │
│  │  │  3. Per-hart Queue: Push/Pop/Complete                   │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  │                          ▲                                   │  │
│  │                          │ courier handler                   │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │              Host IRQchip Drivers                       │ │  │
│  │  │           (APLIC / PLIC / IMSIC)                        │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          ▲                                         │
│                          │                                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                   Physical Hardware                          │  │
│  │        UART IRQ 10      │      Timer IRQ 7      │   ...      │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

## 三大核心功能

### 1. VIRQ Mapping（虛擬 IRQ 映射）

```
┌───────────────────────────────────────────────────────────────────────┐
│  HWIRQ vs VIRQ                                                        │
│                                                                       │
│  HWIRQ = Hardware IRQ (真實硬體中斷號)                                │
│  VIRQ  = Virtual IRQ  (虛擬中斷號，給 S-mode 看的)                    │
│                                                                       │
│  為什麼需要 mapping？                                                 │
│  - 多個 irqchip 可能有相同的 hwirq 號                                 │
│  - VIRQ 提供統一的命名空間                                            │
│                                                                       │
│  ┌─────────────────┐              ┌─────────────────┐                 │
│  │ M-mode APLIC    │              │    VIRQ Space   │                 │
│  │ chip_uid = 0x100│              │                 │                 │
│  │                 │              │                 │                 │
│  │ hwirq 10  ──────┼──────────────▶  VIRQ 1 ────────┼─▶ APLIC hwirq 15│
│  │ hwirq 15  ──────┼──────────────▶  VIRQ 2 ────────┼─▶ APLIC hwirq 10│
│  └─────────────────┘              │                 │                 │
│                                   │                 │                 │
│  ┌─────────────────┐              │                 │                 │
│  │ PLIC            │              │                 │                 │
│  │ chip_uid = 0x200│              │                 │                 │
│  │                 │              │                 │                 │
│  │ hwirq 10  ──────┼──────────────▶  VIRQ 3 ────────┼─▶ PLIC hwirq 10 │
│  └─────────────────┘              │                 │                 │
│                                   └─────────────────┘                 │
└───────────────────────────────────────────────────────────────────────┘
```

```c
/* Forward mapping: (chip_uid, hwirq) -> VIRQ */
int sbi_virq_hwirq2virq(u32 chip_uid, u32 hwirq, u32 *out_virq);

/* Reverse mapping: VIRQ -> (chip_uid, hwirq) */
int sbi_virq_virq2hwirq(u32 virq, u32 *out_chip_uid, u32 *out_hwirq);
```

### 2. Route Rules（路由規則）

```
┌────────────────────────────────────────────────────────────────────┐
│  DTS 定義 routing rules                                            │
│                                                                    │
│  opensbi-domains {                                                 │
│      domain_a {                                                    │
│          opensbi,host-irqs = <10 5>;  /* IRQ 10-14 -> domain_a */  │
│      };                                                            │
│      domain_b {                                                    │
│          opensbi,host-irqs = <20 10>; /* IRQ 20-29 -> domain_b */  │
│      };                                                            │
│  };                                                                │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Route Rules Table                                          │   │
│  │                                                             │   │
│  │  Rule 0: [10..14] ──▶ Domain A                              │   │
│  │  Rule 1: [20..29] ──▶ Domain B                              │   │
│  │  Default: other   ──▶ Root Domain                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  當 HWIRQ 15 發生時：                                              │
│    sbi_virq_route_lookup_domain(15) ──▶ 沒有 match ──▶ Root Domain │
│                                                                    │
│  當 HWIRQ 22 發生時：                                              │
│    sbi_virq_route_lookup_domain(22) ──▶ match rule 1 ──▶ Domain B  │
└────────────────────────────────────────────────────────────────────┘
```

### 3. Per-(Domain, Hart) Pending Queue

```
┌────────────────────────────────────────────────────────────────────┐
│  Per-(Domain, Hart) VIRQ Queue                                     │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Domain A                                                   │   │
│  │  ┌───────────────────┐  ┌───────────────────┐               │   │
│  │  │ Hart 0 Queue      │  │ Hart 1 Queue      │               │   │
│  │  │ [VIRQ3][VIRQ7][ ] │  │ [VIRQ5][ ][ ][ ]  │               │   │
│  │  │  head──▲    tail  │  │ head──▲     tail  │               │   │
│  │  └───────────────────┘  └───────────────────┘               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Domain B                                                   │   │
│  │  ┌───────────────────┐  ┌───────────────────┐               │   │
│  │  │ Hart 0 Queue      │  │ Hart 1 Queue      │               │   │
│  │  │ [ ][ ][ ][ ]      │  │ [VIRQ2][ ][ ][ ]  │               │   │
│  │  │ (empty)           │  │ head──▲     tail  │               │   │
│  │  └───────────────────┘  └───────────────────┘               │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

```c
struct sbi_domain_virq_state {
    spinlock_t lock;
    u32 head;
    u32 tail;
    u32 q[VIRQ_QSIZE];              /* Ring buffer */
    struct sbi_irqchip_device *chip; /* For unmask on complete */
};
```

## 完整中斷處理流程

```
┌────────────────────────────────────────────────────────────────────┐
│  Step 1: UART IRQ 10 觸發                                          │
│         │                                                          │
│         ▼                                                          │
│  Step 2: M-mode APLIC ──▶ OpenSBI (MEIP trap)                      │
│         │                                                          │
│         ▼                                                          │
│  Step 3: irqchip driver claim HWIRQ 10                             │
│         │                                                          │
│         ▼                                                          │
│  Step 4: 呼叫 sbi_virq_courier_handler(hwirq=10, ctx)              │
│         │                                                          │
│         │  4a. map (chip_uid, 10) ──▶ VIRQ 3                       │
│         │  4b. route_lookup(10) ──▶ Domain A                       │
│         │  4c. mask HWIRQ 10 (防止 level-trigger storm)            │
│         │  4d. enqueue VIRQ 3 into Domain A / Hart 0 queue         │
│         │  4e. inject SSE event to Domain A                        │
│         │                                                          │
│         ▼                                                          │
│  Step 5: mret ──▶ Domain A S-mode (SSE handler 觸發)               │
│         │                                                          │
│         ▼                                                          │
│  Step 6: S-mode SSE handler:                                       │
│         │                                                          │
│         │  6a. ecall sbi_virq_pop_thishart()                       │
│         │      ──▶ 返回 VIRQ 3                                     │
│         │                                                          │
│         │  6b. 執行 device ISR (處理 UART)                         │
│         │                                                          │
│         │  6c. ecall sbi_virq_complete_thishart(VIRQ 3)            │
│         │      ──▶ OpenSBI unmask HWIRQ 10                         │
│         │                                                          │
│         ▼                                                          │
│  Step 7: 中斷處理完成                                              │
└────────────────────────────────────────────────────────────────────┘
```

## API 總覽

```c
/*=== Mapping APIs ===*/
int sbi_virq_map_init(u32 init_virq_cap);
int sbi_virq_map_one(u32 chip_uid, u32 hwirq, ...);
int sbi_virq_hwirq2virq(u32 chip_uid, u32 hwirq, u32 *out_virq);
int sbi_virq_virq2hwirq(u32 virq, u32 *out_chip_uid, u32 *out_hwirq);
int sbi_virq_unmap_one(u32 virq);
void sbi_virq_map_uninit(void);

/*=== Routing APIs ===*/
void sbi_virq_route_reset(void);
int sbi_virq_route_add_range(struct sbi_domain *dom, u32 first, u32 count);
struct sbi_domain *sbi_virq_route_lookup_domain(u32 hwirq);

/*=== Queue APIs ===*/
int sbi_virq_enqueue(struct sbi_virq_courier_binding *c);
u32 sbi_virq_pop_thishart(void);
void sbi_virq_complete_thishart(u32 virq);

/*=== Courier Handler (for irqchip driver) ===*/
int sbi_virq_courier_handler(u32 hwirq, void *opaque);

/*=== Init/Exit ===*/
int sbi_virq_domain_init(struct sbi_domain *dom);
void sbi_virq_domain_exit(struct sbi_domain *dom);
int sbi_virq_init(u32 init_virq_cap);
```

## S-mode 使用（透過 ecall）

```c
/* S-mode payload (Linux 或 bare-metal) */

void virq_sse_handler(void)
{
    u32 virq;
    
    /* Pop 所有 pending VIRQs */
    while ((virq = sbi_ecall_virq_pop()) != 0) {
        /* 找到對應的 device handler */
        struct irq_handler *h = find_handler(virq);
        if (h)
            h->callback(virq, h->priv);
        
        /* Complete 這個 VIRQ */
        sbi_ecall_virq_complete(virq);
    }
}
```

## 與其他方案的比較

| 方案 | PMP 需求 | 效率 | S-mode 修改 | 標準化 |
|------|---------|------|------------|--------|
| **Trap-and-Emulate (PMP)** | 需要 | 差 | 不需要 | - |
| **Custom 指令** | 不需要 | 中 | 需要 | 非標準 |
| **Ecall Paravirt (自定義)** | 不需要 | 好 | 需要 | 非標準 |
| **RPMI System IRQ over MPXY** | 不需要 | 好 | 需要 | SBI 標準 |
| **VIRQ Layer (此 patch)** | 不需要 | 好 | 需要 | 待提議 |

## 關鍵特點

| 特點 | 說明 |
|------|------|
| **Per-domain routing** | 不同 domain 可以接收不同的 IRQs |
| **Per-hart queue** | 每個 hart 有自己的 pending queue |
| **Level-trigger safe** | 在 enqueue 前 mask，complete 時 unmask |
| **SSE integration** | 使用 SSE 通知 S-mode |
| **Scalable** | Chunked allocation，隨需成長 |

這個 VIRQ layer 提供了一個完整的 paravirt interrupt 解決方案，讓 OpenSBI 可以在 M-mode 處理中斷並安全地轉發給不同的 S-mode domains！

