| Platforms | M-level MSIs | S-level MSIs | VS-level MSIs | M-level Wired Interrupts | S-level Wired Interrupts | VS-level Wired Interrupts | M-level IPIs | S-level IPIs | VS-level IPIs | M-level Timer | S-level Timer | VS-level Timer |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Legacy Wired IRQs** | NA | NA | NA | PLIC | PLIC | PLIC (Emulate) | <font color="green">MSWI (CLINT)</font> | <font color="red">SBI IPI</font> | <font color="red">SBI IPI</font> | <font color="green">MTIMER (CLINT)</font> | <font color="red">SBI Timer</font> | <font color="red">SBI Timer</font> |
| **Only Wired IRQs** | NA | NA | NA | <font color="blue">APLIC M-level</font> | <font color="blue">APLIC S-level</font> | <font color="blue">APLIC S-level (Emulate)</font> | <font color="green">MSWI</font> | <font color="green">SSWI</font> | <font color="red">SBI IPI</font> | <font color="green">MTIMER</font> | <font color="brown">Priv Sstc</font> | <font color="brown">Priv Sstc</font> |
| **MSIs and Wired IRQs** | <font color="blue">IMSIC M-file</font> | <font color="blue">IMSIC S-file</font> | <font color="blue">IMSIC S-file (Emulate)</font> | <font color="blue">APLIC M-level</font> | <font color="blue">APLIC S-level</font> | <font color="blue">APLIC S-level (Emulate)</font> | <font color="blue">IMSIC M-file</font> | <font color="blue">IMSIC S-file</font> | <font color="red">SBI IPI</font> | <font color="green">MTIMER</font> | <font color="brown">Priv Sstc</font> | <font color="brown">Priv Sstc</font> |
| **MSIs, Virtual MSIs and Wired IRQs** | <font color="blue">IMSIC M-file</font> | <font color="blue">IMSIC S-file</font> | <font color="blue">IMSIC VS-file</font> | <font color="blue">APLIC M-level</font> | <font color="blue">APLIC S-level</font> | <font color="blue">APLIC S-level (Emulate)</font> | <font color="blue">IMSIC M-file</font> | <font color="blue">IMSIC S-file</font> | <font color="blue">IMSIC VS-file</font> | <font color="green">MTIMER</font> | <font color="brown">Priv Sstc</font> | <font color="brown">Priv Sstc</font> |

## QEMU Hardware Behavior

// TBD·

```
MIP.MEIP = pending[src]
            && enabled[src]
            && DOMAINCFG.IE
            && IDC.idelivery
            && (priority > IDC.ithreshold)
            && (target_hart == current_hart)
```
