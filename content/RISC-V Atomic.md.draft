### RISC-V ATOMIC

---

### Motivation for Atomicity

* Parallelism
	* Facilitates efficient execution by leveraging multi-core architectures.
* Multi-threading
	* Improves responsiveness, scalability, CPU utilization and achieves resource sharing.
* Challenge: Race Conditions
	 * Occur when multiple threads/CPUs access shared variables concurrently without appropriate synchronization mechanisms.

---

Even in uni-processor, systems can suffer race conditions due to
* scheduling
* interrupts

---

### Key concurrency terms

---

What?

* **Race Condition**: A situation where multiple threads enter the critical section at roughly the same time, leading to a surprising and undesirable outcome.

---

Where?

* **Critical Section**: A piece of code that accesses a shared resource, usually a variable or data structure.

---

How?

* **Mutual Exclusion**: A technique that guarantees only a single thread enters a critical section, avoiding races and ensuring deterministic program outputs.

---

### Examples

---

#### Software State Machine

Hart state management (HSM): hart state is per-hart (stored in `struct sbi_scratch`) data which could be updated by current/remote harts.

![hsm|600](https://i.imgur.com/Oha5XYN.png)

---

#### Concurrent counter

Alex's [patch](https://patchwork.kernel.org/project/linux-riscv/patch/20231207150348.82096-5-alexghiti@rivosinc.com): sfence counter w/ debugfs

```c
u64 nr_sfence_vma; /* concurrent counter */

static inline void local_flush_tlb_all(void)
{
/* Using GCC built-in to count specific
 * function calls in a multi-processing context */
	__sync_fetch_and_add(&nr_sfence_vma_all, 1UL);
	__asm__ __volatile__ ("sfence.vma" : : : "memory");
}

/* Expose the counter to user space with debugfs. */
static int debugfs_nr_sfence_vma(void)
{
	debugfs_create_u64("nr_sfence_vma", 0444,
					 NULL, &nr_sfence_vma);
	return 0;
}
device_initcall(debugfs_nr_sfence_vma);
```

---

#### Lottery

![](https://i.imgur.com/2Fg1Mm5.png)

---

#### Lottery

```
_try_lottery:
        lla     a6, _boot_status
        li      a7, BOOT_STATUS_LOTTERY_DONE
        amoswap.w a6, a7, (a6)
        bnez    a6, _wait_for_boot_hart
```


---

#### Lottery

![alt|700](https://i.imgur.com/EnSDUWJ.png)

---

#### Memory-mapped I/O

OpenSBI uses a spin lock to prevent outputs from interleaving.

```c
void sbi_puts(const char *str)
{
        unsigned long len = sbi_strlen(str);

        spin_lock(&console_out_lock);
        nputs_all(str, len);
        spin_unlock(&console_out_lock);
}
```

---

## Definition of Atomicity

Atomicity refers to the property of a operation that ensures they are completed as a ==single, indivisible unit==.

---

### Providing Hardware Support for Synchronization

---

Design decisions

* LR/SC pair:
	* Commonly used on  2-operand ISAs
	* Two registers for each instruction
* Single CAS instruction:
	* CISC machine
	* 3 registers
		1. address
		2. old value
		3. new value
		
---

### Providing Hardware Support for Synchronization

* RISC-V adopts LR/SC (a.k.a LL/SC, Load-link/store-conditional)
	* The LR/SC pair is more robust compared to Compare-And-Swap (CAS) operation.
		* CAS may fail if the original value has been restored (ABA problem)


---

### RISC-V Atomic Instruction (RVA)

![a](https://i.imgur.com/vifftBT.png)

---

### RISC-V Atomic Instruction (RVA)

![b](https://i.imgur.com/JXVOqgk.png)

---

### RISC-V Atomic Compare-and-Swap instructions (`Zacas`)

November 2023 ratified

![](https://i.imgur.com/jmW3Ja4.png)

---

To avoid the ABA problem, the algorithms associate a ==reference counter== with the pointer variable and perform updates using a ==quadword compare and swap== of both the pointer and the counter.
* RV32: double-word CAS
* RV64: quad-word CAS

---

Problem: A => B => A
* We can't tell whether A is original one.
* Tagging the value: 1A => 2B => 3A

---

![](https://i.imgur.com/xppZ2fv.png)

---

### RISC-V Byte and Halfword Atomic Memory Operations (`Zabha`)

April 2024 ratified

![](https://i.imgur.com/zuQvTGP.png)

---

### Spinlock Implementations

Each layers, framework usually implements their own synchronization primitives.


OpenSBI and Linux kernel are using ==ticket lock==.

---

### Machine mode: OpenSBI v0.9 Simple Lock

* acquire/release memory ordering
	* ACUIRE_BARRIER yield `fence r, rw`
	* `__smp_store_release()` implies `fence rw, w`
* alternativly, use amo instruction with aq/lr annotation
	* `amoswap.w.aq` => critical section => `amoswap.w.rl`
---

![as|600](https://i.imgur.com/KK8IWjU.png)

---

Why do we need `spin_lock_check()`?

![](https://i.imgur.com/Q2UxMjg.png)

---

AMO is **EXPENSIVE**

---

Consider that the lock variable is cached in the ==local cache line==. 
Before performing an atomic swap (amoswap), we can:

1. Reduce the overhead of atomic instructions (bus traffic) with `spin_lock_check()`
	* MESI shared request
2. Attempt to acquire lock with `spin_trylock()`
	* MESI exclusive request (more expensive)

---

![](https://i.imgur.com/1vaYnor.png)

---

### Machine mode: OpenSBI v1.0 Ticket Lock

Implementing a ticket lock can mitigate the advantage a CPU gains from cache upon winning the initial round, ensuring fairness in re-acquiring the lock.

![](https://i.imgur.com/4NKvtSS.png)

---

![](https://i.imgur.com/PZ7ASLP.png)

---

### Supervisor mode: Linux Spinlock Variants

<!---
|                       | disable preemption | disable interrupts | save IRQ status | sleep |
| --------------------- | ------------------ | ------------------ | --------------- | --- |
| `spin_lock()`         | O                  | X                  | X               | X |
| `spin_lock_irq()`     | O                  | O                  | X               | X |
| `spin_lock_irqsave()` | O                  | O                  | O               | X|
| `mutex_lock()`         | O                  | X                  | X               | O|
-->

![](https://i.imgur.com/XjLyfcD.png)

---

## User mode: GCC built-ln functions

C11 [`__atomic`](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html) builtins = legacy [`__sync`](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fsync-Builtins.html) builtins w/ explicit memory order

* 6 different memory orders that can be specified.
* The valid memory order is different for each API, and the effect varies depending on the architecture.

---

| Memory consistency model | Arch |
| --- | --- |
| SC (Sequential Consistency) | MIPS R10K |
| TSO (Total Store Order) | IBM-370, x86, sparc, RISC-V RVTSO |
| Weak, multi-copy-atomic | ==RISC-V RVWMO==, Revised ARMv8|
| Weak, non-multi-copy-atomic  | ARMv7, original ARMv8, IBM POWER|



---

## Lock-free Programming

---

### How to achieve lock-free?

(Reference: OSTEP - COMMON CONCURRENCY PROBLEMS)

```c
int CompareAndSwap(int *address, int expected, int new) {
	if (*address == expected) {
		*address = new;
		return 1; // success
	}
	return 0; // failure
}
```

---

```
/* 
 * int CompareAndSwap(int *address, int expected, int new);
 * Returns:  
 *  - 1 if value matched the expected and was swapped
 *  - 0 if fails
 */
CompareAndSwap:
	lr.w a3, (a0)
	bne  a3, a1, fail
	sc.w a3, a2, (a0) # if success, a3 <= 0
	                  # otherwise , a3 <= non-zero
	bnez a3, fail     # store-conditional failed
	li   a0, 1
	ret
fail:
	li   a0, 0
	ret
```

---

```c
void AtomicIncrement(int *value, int amount) {
	do {
		int old = *value;
	} while (CompareAndSwap(value, old, old + amount) == 0);
}
```

---

### Lock-free v.s Lock

---

Insert a node in the front of list in single threaded program.

```mermaid
graph LR  
    A[new node] -.-> B[head node]  
    B --> C[node]  
    C --> D[null]
```

```c
void insert(int value)
{
	node_t *new_node = malloc(sizeof(node_t));
	new_node->value = value;
	/* Update head */
	new_node->next = head;
	head = new_node;
}
```

---

Concurrent linked-list (w/ spinlock): insert a node

```c
void insert(int value)
{
	node_t *new_node = malloc(sizeof(node_t));
	new_node->value = value;
	pthread_mutex_lock(listlock); // begin critical section
	/* Update head */
	new_node->next = head;
	head = new_node;
	pthread_mutex_unlock(listlock); // end critical section
}
```

---

Concurrent linked-list (w/o spinlock): insert a node

```c
void insert(int value)
{
	node_t *new_node = malloc(sizeof(node_t));
	new_node->value = value;
	/* Update head */
	do {
		new_node->next = head;
	/* 
	 * 1. Expecting head == new_node->next
	 * 2. Atomically assign new_node to head
	 */
	} while (CompareAndSwap(&head, new_node->next, new_node) == 0);
}
```

---

## Wait... is it still a busy waiting?

Spinlocks maintain mutual exclusion by holding locks, preventing other threads from accessing critical sections, but can cause busy-waiting and degrade performance under high contention. CAS, used for atomic updates, offers a more **granular** and scalable lock-free approach, minimizing overhead.

---

## Experimant: concurrent-ll

sysprog21/concurrent-ll

[![sysprog21/concurrent-ll - GitHub](https://gh-card.dev/repos/sysprog21/concurrent-ll.svg)](https://github.com/sysprog21/concurrent-ll)

---

CPU: AMD Ryzen Threadripper PRO 5965WX s (48) @ 3.800GHz

![](https://i.imgur.com/nMz9SWE.png)

---

CPU: Spacemit X60 (8) @ 1.600GHz

![](https://i.imgur.com/dFt8SQg.png)

---

## Conclusion

* Atomicity is a critical concept in operating systems.
* Ensures data integrity, concurrency control, performance optimization.

