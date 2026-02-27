---
title: Interrupt Routing Table
tags:
  - virq
  - routing
---

# Interrupt Routing Table

The routing table determines which security domain should handle each hardware interrupt.

## Rule Structure

```c
struct sbi_virq_route_rule {
    u32 first;              // First hwirq in range (inclusive)
    u32 last;               // Last hwirq in range (inclusive)
    struct sbi_domain *dom; // Destination domain
};
```

## Configuration

### Current: Hardcoded

```c
// In sbi_virq_init()
/* TODO: Parse opensbi,host-irqs from DT instead of hardcoding */
sbi_virq_route_add_range(trusted_dom, 10, 1);  // hwirq=10 → OP-TEE
```

### Future: DeviceTree Parsing

```dts
opensbi-domain-optee {
    compatible = "opensbi,domain";
    opensbi,host-irqs = <10 1>;  // first=10, count=1
    // ...
};
```

## Lookup Algorithm

```c
struct sbi_domain *sbi_virq_route_lookup_domain(u32 hwirq)
{
    for (i = 0; i < route_rule_count; i++) {
        if (hwirq >= route_rules[i].first &&
            hwirq <= route_rules[i].last) {
            return route_rules[i].dom;
        }
    }
    return &root;  // Default: root domain (Linux)
}
```

## Public APIs

| Function | Description |
|----------|-------------|
| `sbi_virq_route_reset()` | Clear all rules |
| `sbi_virq_route_add_range(dom, first, count)` | Add a routing rule |
| `sbi_virq_route_lookup_domain(hwirq)` | Find destination domain |

## Example Routing

| hwirq | Device | Destination | Reason |
|-------|--------|-------------|--------|
| 10 | UART | OP-TEE | Secure console |
| 11-15 | GPIO | Linux | Normal peripherals |
| 16 | Crypto | OP-TEE | Secure accelerator |

## Overlap Detection

```c
// Overlap is rejected
if (!(last < rules[i].first || first > rules[i].last)) {
    return SBI_EALREADY;
}
```

## Related

- [[VIRQ Layer Overview]]
- [[VIRQ Courier Handler]]

