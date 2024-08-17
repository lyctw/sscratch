---
title: RISC-V 性能監控單元與 Sscofpmf 擴展
tags:
  - RISC-V
  - perf-tool
  - Linux
  - OpenSBI
date: 2023-08-09
---

perf 自 [Linux 5.18](https://lwn.net/Articles/879905/) 後標準規格與軟硬體實作 ([Sscofpmf](https://github.com/riscv/riscv-count-overflow), [SBI PMU](https://github.com/riscv-non-isa/riscv-sbi-doc/blob/v1.0.0/riscv-sbi.adoc#performance-monitoring-unit-extension-eid-0x504d55-pmu), [Linux perf driver](https://github.com/torvalds/linux/blob/v5.18/drivers/perf/riscv_pmu_sbi.c)) 已經成熟，取代 RISC-V legacy perf，並且 [AndesCore™ AX65](https://www.andestech.com/en/products-solutions/andescore-processors/riscv-ax65/) 是第一個支援 Sscofpmf 的 Andes A系列 CPU, 取代了 45 系列以前的 Andes HPM。本文先簡介 perf 指令基本用法，接著是 ISA 層級的 Sscofpmf 規格、SBI 的 PMU extension 提供的界面，嘗試整理 RISC-V 架構上的現況。

### Glossary

| 名詞                     | 描述                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Andes HPM                | 新增 `$mi{e,p}.PMOVI` 與一組 custom CSRs， 在 Sscofpmf 尚未 ratified 前作為先行 (pre-spec) 方案支援 perf sampling 與 filtering 功能，因此當硬體有支援 Sscofpmf 時，Andes HPM 不會存在，反之亦然，可由 `$mmsc_cfg.PMNDS` 來確認 bitmap 是否有實作。由於已被 Sscofpmf 取代，本文將不介紹這個 Andes-specific 設計                                                                                       |
| Sscofpmf                 | Count Overflow and Mode-Based Filtering Extension, Ratified in November 2021, RISC-V 的標準 extension，但尚未被整合到 ISA specifications volume I 或 volume II, 文件在 Github [riscv-counter-overflow](https://github.com/riscv/riscv-count-overflow)                                                                                                                                          |
| SBI PMU                  | Supervisor Binary Interface 所定義的 [Performance Monitor Unit Extension](https://github.com/riscv-non-isa/riscv-sbi-doc/blob/v1.0.0/riscv-sbi.adoc#performance-monitoring-unit-extension-eid-0x504d55-pmu), Linux 原始碼中 RISC-V perf driver 會透過這個 SBI extension 來得知 hardware/firmware counter 數量，並配置可用的 counter。這個 extension 也支援  microarchitecture 的各種 raw event |
| firmware counter         | RISC-V 特有的 event, 用於計數 SBI firmware 特定函數，無需硬體支援，完全由軟體實作，不支援 counter overflow interrupt                                                                                                                                                                                                                                                                          |
| raw event                | SBI 所定義的 hardware event 有三種，若不屬於 hardware general 或 hardware cache event 即歸類在 raw event，一般為 microarchitecture 層級的硬體事件，種類五花八門 (e.g. branch mispredictions, prefetch bus accesses, etc，詳細可參閱 AndesCore DataSheet Event Selectors 表格)，`perf` 指令能用 raw event descriptor 來計數                                                                     |
| programmable HPM counter | HPM3-31, 可以由 mhpmevent 來決定 mhpmcounter 正在計數哪種硬體事件。而 cycle 與 instret 這兩個就是 fixed counter, 只能用來計數 CPU cycle 與 instruction retired 次數，且不具備 counter overflow 能力（沒有對應的 mhpmeventx.OF）                                                                                                                                                                |
| perf                     | Linux 的效能量測工具，為 user space 的命令列工具，但原始碼為 Linux kernel source code (`tools/perf`) 的一部分                                                                                                                                                                                                                                                                                  |
| perf driver              | Linux kernel 中的 PMU 框架，實作初始化、啟動、讀取、停止 counter 等功能，RISC-V 需要編譯位於 `drivers/perf` 下的 `riscv_pmu.c` 與 `riscv_pmu_sbi.c` 的兩支檔案                                                                                                                                                                                                                                 |

### 相關 CSR

Linux kernel 與 OpenSBI 的 PMU 相關程式蠻多的，但只要記得他們最終都是要操作以下幾個 CSRs 就能理解其中的邏輯，因此，我們需要清楚以下各個 CSRs 的用途。

| CSR             | Encoding    | Mode | Description                                                                                                              |
| --------------- | ----------- | ---- | ------------------------------------------------------------------------------------------------------------------------ |
| `mcycle`          | 0xB00       | MRW  | Machine cycle counter                                                                                                    |
| `minstret`        | 0xB02       | MRW  | Machine instructions-retired counter                                                                                     |
| `mhpmcounter3-31` | 0xB03-0xB1F | MRW  | Machine performance-monitoring counter                                                                                   |
| `mhpmevent3-31`   | 0x324-0x33F | MRW  | Machine performance-monitoring event selector                                                                            |
| `mcounteren`      | 0x306       | MRW  | Machine counter enable                                                                                                   |
| `mcountinhibit`   | 0x320       | MRW  | Machine counter-inhibit register                                                                                         |
| `cycle`           | 0xC00       | URO  | Cycle counter for RDCYCLE instruction                                                                                    |
| `time`            | 0xC01       | URO  | Timer for RDTIME instruction                                                                                             |
| `instret`         | 0xC02       | URO  | Instructions-retired counter for RDINSTRET instruction                                                                   |
| `hpmcounter3-31`  | 0xC03-0xC1F | URO  | Performance-monitoring counter                                                                                           |
| `scountovf`       | 0xDA0       | SRO  | 32-bit read-only register that contains shadow copies of the OF bits in the 29 mhpmevent CSRs (mhpmevent3 - mhpmevent31) |

## Perf 基礎指令

perf 可說是 Linux 標準的 profiler tool, 原始碼本身存在於 Linux kernel source 內，支援豐富軟硬體、tracepoint 事件，本文僅對測試硬體事件計數器會用到的相關指令做介紹

```
$ perf list | grep Hardware
  branch-instructions OR branches                    [Hardware event]
  branch-misses                                      [Hardware event]
  bus-cycles                                         [Hardware event]
  cache-misses                                       [Hardware event]
  cache-references                                   [Hardware event]
  cpu-cycles OR cycles                               [Hardware event]
  instructions                                       [Hardware event]
  ref-cycles                                         [Hardware event]
  stalled-cycles-backend OR idle-cycles-backend      [Hardware event]
  stalled-cycles-frontend OR idle-cycles-frontend    [Hardware event]
  L1-dcache-load-misses                              [Hardware cache event]
  L1-dcache-loads                                    [Hardware cache event]
  L1-dcache-prefetch-misses                          [Hardware cache event]
  L1-dcache-prefetches                               [Hardware cache event]
  L1-dcache-store-misses                             [Hardware cache event]
  L1-dcache-stores                                   [Hardware cache event]
  L1-icache-load-misses                              [Hardware cache event]
  L1-icache-loads                                    [Hardware cache event]
  L1-icache-prefetch-misses                          [Hardware cache event]
  L1-icache-prefetches                               [Hardware cache event]
  LLC-load-misses                                    [Hardware cache event]
  LLC-loads                                          [Hardware cache event]
  LLC-prefetch-misses                                [Hardware cache event]
  LLC-prefetches                                     [Hardware cache event]
  LLC-store-misses                                   [Hardware cache event]
  LLC-stores                                         [Hardware cache event]
  branch-load-misses                                 [Hardware cache event]
  branch-loads                                       [Hardware cache event]
  dTLB-load-misses                                   [Hardware cache event]
  dTLB-loads                                         [Hardware cache event]
  dTLB-prefetch-misses                               [Hardware cache event]
  dTLB-prefetches                                    [Hardware cache event]
  dTLB-store-misses                                  [Hardware cache event]
  dTLB-stores                                        [Hardware cache event]
  iTLB-load-misses                                   [Hardware cache event]
  iTLB-loads                                         [Hardware cache event]
  node-load-misses                                   [Hardware cache event]
  node-loads                                         [Hardware cache event]
  node-prefetch-misses                               [Hardware cache event]
  node-prefetches                                    [Hardware cache event]
  node-store-misses                                  [Hardware cache event]
  node-stores                                        [Hardware cache event]
  rNNN                                               [Raw hardware event descriptor]
```

### Perf Statistics

`stat` subcommand 紀錄一個行程的開始至結束所發生的事件發生次數，如下為 clock cycle, cache miss 與 cache reference （即 cache hit） 所發生的次數

```
$ perf stat -e cycles,cache-misses,cache-references ./program

 Performance counter stats for './program':

           1474542      cycles                                                             
              8871      cache-misses        # 1.147 % of all cache refs    
            773542      cache-references                                                   

       0.031257950 seconds time elapsed

       0.010388000 seconds user
       0.020776000 seconds sys

```

### Perf Sampling

Sampling (取樣) 功能會在開始調節適當頻率觸發中斷，通知 Linux kernel 紀錄當下執行的 user/kernel space 函式，可以作為最佳化的參考，如下蒐集到 142 個發生 cache miss 的取樣點，
，其中 21% 位在 user space 的 `_dl_relocate_object()`， kernel space 最頻繁發生 cache miss 是位在 `memcpy()` 

```
$ perf record -e cache-misses ./program 
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.007 MB perf.data (142 samples) ]
$ perf report
# To display the perf.data header info, please use --header/--header-only options.
#                                                                                                          
#                                                                                                          
# Total Lost Samples: 0                                                                                    
#                                                                                                          
# Samples: 142  of event 'cache-misses'                                                                    
# Event count (approx.): 9438                                                                              
#                                                                                                          
# Overhead  Command  Shared Object                Symbol                                  
# ........  .......  ...........................  ........................................
#                                                                                                          
    21.08%  program  ld-linux-riscv64-lp64d.so.1  [.] _dl_relocate_object
    12.28%  program  ld-linux-riscv64-lp64d.so.1  [.] check_match           
     4.83%  program  ld-linux-riscv64-lp64d.so.1  [.] strcmp 
     4.49%  program  ld-linux-riscv64-lp64d.so.1  [.] do_lookup_x    
     3.16%  program  [kernel.kallsyms]            [k] memcpy          
     2.00%  program  [kernel.kallsyms]            [k] memset         
     1.94%  program  [kernel.kallsyms]            [k] page_remove_rmap
     1.92%  program  [kernel.kallsyms]            [k] unmap_page_range
     1.43%  program  [kernel.kallsyms]            [k] vma_interval_tree_remove
     1.41%  program  [kernel.kallsyms]            [k] filemap_map_pages
     1.29%  program  [kernel.kallsyms]            [k] xas_load
     1.26%  program  [kernel.kallsyms]            [k] PageHuge
```

### Perf Filtering

當我們想限制事件觸發的權限模式，可以在事件後加上 `:{k|u}` 來指定僅在 kernel or user space 進行計數

```
$ perf stat -e cache-misses:u,cache-references:k ./program

 Performance counter stats for './program':

              4694      cache-misses:u                                                     
            458822      cache-references:k                                                 

       0.027455533 seconds time elapsed

       0.000000000 seconds user
       0.028443000 seconds sys
```

`perf record` 亦可使用，可見 symbol 一行前面有 `[.]` 表示其為 userspace, 而 kernel space 將會顯示 `[k]`

```
# perf record -e cache-misses:u ./program                                               
Lowering default frequency rate from 4000 to 3600.                                                         
Please consider tweaking /proc/sys/kernel/perf_event_max_sample_rate.                                      
[ perf record: Woken up 221 times to write data ]                                                          
[ perf record: Captured and wrote 0.016 MB perf.data (315 samples) ]                                       
# perf report | tee /tmp/asdkgj                                                                            
# To display the perf.data header info, please use --header/--header-only options.                         
#                                                                                                          
#                                                                                                          
# Total Lost Samples: 0                                                                                    
#                                                                                                          
# Samples: 97  of event 'cache-misses:u'                                                                   
# Event count (approx.): 4974                                                                              
#                                                                                                          
# Overhead  Command  Shared Object                Symbol                                                   
# ........  .......  ...........................  ............................                             
#                                                                                                          
    34.38%  program  ld-linux-riscv64-lp64d.so.1  [.] _dl_relocate_object                                  
    19.32%  program  ld-linux-riscv64-lp64d.so.1  [.] check_match                                          
    16.28%  program  ld-linux-riscv64-lp64d.so.1  [.] do_lookup_x                                          
     4.46%  program  ld-linux-riscv64-lp64d.so.1  [.] strcmp                                               
     3.58%  program  ld-linux-riscv64-lp64d.so.1  [.] _dl_lookup_symbol_x       
     2.49%  program  libc.so.6                    [.] malloc                                               
     1.87%  program  ld-linux-riscv64-lp64d.so.1  [.] _dl_find_object_from_map                             
     1.87%  program  libc.so.6                    [.] __libc_early_init                                    
     1.71%  program  libc.so.6                    [.] __libc_start_call_main                               
     1.53%  program  libc.so.6                    [.] __sbrk                                         
     1.47%  program  ld-linux-riscv64-lp64d.so.1  [.] dl_main                                        
     1.33%  program  ld-linux-riscv64-lp64d.so.1  [.] _dl_map_object_from_fd                         
     1.27%  program  busybox                      [.] endofname
```


### Event Multiplexing

由於硬體事件有很多種 （AX45MP datasheet 所紀錄的就有 50 種以上），目前 Andes 的 CPU 可用的 HPM counter 只有 4 個，因此當 perf 同時計算超過 4 個硬體事件 （`perf stat -e evt1,evt2,evt3,evt4,evt5 <cmd>`） 就會用多工 (multiplexing) 的方式，切分程式運作的時間段，讓每個事件都有有機會佔用 HPM counter 來做計數，如下圖最右邊一行所顯示百分比即表示該事件僅在特定時間段有進行計數。firmware counter 完全是軟體定義、位在記憶體上的變數，但也是有數量限制，OpenSBI 有一個 counter array，其中有 64bit counter 共 16 個用於計數 firmware events, 包含 standard 與 custom firmware event （見後面介紹）, 因此當事件超過 16 個便會開始多工。

```
$ perf stat -e L1-dcache-loads,L1-dcache-load-misses,L1-icache-loads,L1-icache-l
oad-misses,ra0,r02 sleep 3

 Performance counter stats for 'sleep 3':

            364045      L1-dcache-loads          (19.97%)
              7838      L1-dcache-load-misses    (54.87%)
           1013941      L1-icache-loads          (94.56%)
             13313      L1-icache-load-misses    
             35662      ra0                      (85.47%)
              8268      r02                      (45.13%)

       3.030985043 seconds time elapsed

       0.010592000 seconds user
       0.021185000 seconds sys
```

由於 counter 數量總是有限的，因此若希望能精確計算某個程式的某個事件發生的次數，就不要同時計算超過 counter 數量的事件，如此一來該事件就能總是佔用某個 counter 進行計數。

## RISC-V ISA extension - Sscofpmf

> 'Ss' 代表 Privileged arch and Supervisor-level extensions, 'cofpmf' 代表 Count OverFlow and Privilege Mode Filtering

Sscofpmf 的制定完成之前，T-head 與 Andes 皆有各自功能相似的實作，為避免各家軟硬體實作的分歧，以及完善 RISC-V privilege spec 的 Hardware Performance Monitor 章節，因此有了有統一規格的需求, 以 RV64 為例，文中定義了。

1. mhpmevent CSRs 的欄位
    * 低位如同 Privilege spec 所定義，作為 event selection 用
    * 最高 63 bit 代表所對應之 mhpmcounter 是否發生 overflow，軟體的應予以清除 (TODO: OpenSBI pmu_ctr_enable_irq_hw)，否則視同 interrupt disabled, 下次對應的 mhpmcounter 若 overflow 將無法再次觸發中斷 （mip 或 sip 的 LCOFIP 不會舉起）
    * 62-58 bits 用於排除該 event 在某個 privilege mode 的計數，即 privilege mode filtering 功能 
1. 對 Privlege spec 的 mip/mie/sip/sie 的 13 bit （local interrupt）定義為 LCOFI（Local Count Overflow Interrupt）
2. 定義 `scountovf`, 由於 S-mode 無法直接存取 `mhpmevent`，因此新增一個 32bit read-only CSR -- `scountovf` 讓 Linux kernel 能夠快速得知 `mhpmevent[3:31].OF` 的狀態  ![[../assets/pmu/scountovf.png]]

> 規格定義 mhpmcounter.OF 這個 bit 為 1 時視同中斷關閉，是個簡潔的設計，不需要額外 CSR 來開關，功能等同 Andes HPM 的 `mcounterinten` 

## RISC-V SBI PMU extension

Linux perf 有一系列硬體事件, perf driver 為了與 OpenSBI 溝通我們想要計算的事件類別，SBI 規格書定義了 event_idx 來對 hardware general event, hardware cache event, hardware raw event 以及 firmware event 給予編號, 後兩者透過 raw hardware descriptor 來描述。

由於 S-mode 的 Linux kernel 不能直接存取 `mhpmcounter`, 所以需要[初始化 `mcounteren`](https://github.com/riscv-software-src/opensbi/blob/v1.2/lib/sbi/sbi_hart.c#L57-L63) 讓 S/U-mode 可以有權限讀寫 shadow CSR `cycle`, `instret`, `hpmcounter3-31`, 透過以下界面 (OpenSBI v1.2) 操作相關 CSR 

1. `sbi_pmu_num_counters()`
2. `sbi_pmu_counter_get_info()`
3. `sbi_pmu_counter_config_matching()`
4. `sbi_pmu_counter_start()`
5. `sbi_pmu_counter_stop()`
6. `sbi_pmu_counter_fw_read()`

### `sbi_pmu_num_counters()`, `sbi_pmu_counter_get_info()`

這兩個 SBI call 會在 Linux boot-time 由 RISC-V perf driver 呼叫，以探知 hardware & firmware counter 數量，取得 [counter 資訊](https://github.com/riscv-non-isa/riscv-sbi-doc/blob/v1.0.0/riscv-sbi.adoc#function-get-details-of-a-counter-fid-1)，前者根據硬體實作數量會有差異, 後者固定 16 個，都是 per-hart counters。

> HPM counter 的數量
> 
> * Andes CPUs 有 4 個, 即 HPM3-6
> * QEMU virt 有 16 個, HPM3-18
> * 最多可以有 29 個, HPM3-31
>
> 以 AX45MP 為例，OpenSBI 會探知（[嘗試讀寫](https://github.com/riscv-software-src/opensbi/blob/master/lib/sbi/sbi_hart.c#L632)，如果有 illegal inst. 表示 CSR 不存在）並顯示 HPM counter 數量:
> ```
> Boot HART MHPM Count      : 4
> ```
> 然而 Linux perf driver 在初始化時，會加上 CY, IR 兩個 non-programmable HPM counter 而顯示 6
> ```
> [    0.439419] riscv-pmu-sbi: 16 firmware and 6 hardware counters
> ```
> 注意 OpenSBI 中的 active_events 陣列為了讓 cidx (counter idx) 可以直接對應，[num_hw_ctrs 會加上 CY, TM, IR](https://github.com/riscv-software-src/opensbi/blob/v1.2/lib/sbi/sbi_pmu.c#L840), 再加上 16 個 firmware event 得到 total_cts, 作為 active_events 的實際長度，見下圖
> ![[../assets/pmu/pmu_active_events.png]]

###  `sbi_pmu_counter_config_matching()`,  `sbi_pmu_counter_start()`, `sbi_pmu_counter_stop()`

這三個 SBI call 用來 runtime 時配置 programmable HPM counter，並且因為 Linux 有 scheduling，當 perf 針對某程式計數時，如果發生 context switch 或是 migrate 到其他 CPU 上，就需要用 `sbi_pmu_ctr_cfg_match()` 重新配置該 hart 可用的 HPM counter。同理 event multiplexing 時也要重新配置。

OpenSBI 會維護一個  active_events 的 per-hart table, 當值為 INVALID 時代表可被使用，如果正被佔用則會填入 event_idx, 如下圖 HPM6 正被用來計算 0x10000 (Level1 data cache event)

active_events:
![[../assets/pmu/pmu_active_events_invalid.png]]



runtime 時透過 SBI call 來配置 programmable HPM counter，並且因為 Linux 有 scheduling，當 perf 針對某 user 程式計數時，如果發生 context switch 或是 migrate 到其他 CPU 上，就需要用 `sbi_pmu_ctr_cfg_match()` 重新配置該 hart 可用的 HPM counter
![[../assets/pmu/sbi_pmu_interface.png]]



### Selector value and device tree bindings


當 hardware general/cache events 計數時傳入一個 event_idx 時, mhpmevent 要填入什麼 selector value 屬於平台規格, 因此會需要將兩者對應 (mapping) 起來, 我們的 AndesCore datasheet 有一個 event selectors 表格, 其中某些是 SBI 定義的 hardware general/cache events, 除此之外就是 hardware raw event，我們要做的就是把該表格的資訊填到 device tree 的 pmu node, 讓 OpenSBI 在開機時初始化 table (兩個 array 分別是 `hw_event_map` 與 `fdt_pmu_evt_select`)，之後使用 perf 就根據這兩個 tables 配置可用的 HPM counter。目前 Andes 的平台的任何 HPM counter (HPM3,4,5,6) 可以用來計數任何事件 

> 值得注意的是 SBI spec 有提到平台可以為了硬體實作上的方便，[把 event_idx 當 selector value 使用](https://github.com/riscv-non-isa/riscv-sbi-doc/blob/v1.0.0/riscv-sbi.adoc?plain=1#L1378-L1380)


* PMU node 有以下三個 property，前兩者為 hardware general/cache event 的資訊，當 `riscv,event-to-mhpmevent` 有給定，則 `riscv,event-to-mhpmcounters` 必須提供; 反之則不然，此種情況就是把 event_idx 當作 selector value 使用
    * `riscv,event-to-mhpmevent`: event_idx 對應到 selector value (一對一對映), 下圖為三個範例
        * ![[../assets/pmu/riscv,event-to-mhpmevent.png]]
    * `riscv,event-to-mhpmcounters`: 一組 event_idx (start_idx ~ end_idx) 可以用的一組 HPM counter (bitmap 格式), 意即使用者可以決定某些 event 只能由特定 HPM counter 計數, [boot-time 會紀錄在 hw_event_map](https://github.com/riscv-software-src/opensbi/blob/v1.2/lib/utils/fdt/fdt_pmu.c#L86-L87), 下圖範例表示 event_idx=3,4 可用 HPM3 or HPM4 計數, event_idx=0x10000,0x10009 可用 HPM5 or HPM6 計數
        * ![[../assets/pmu/riscv,event-to-mhpmcounters.png]]
    * `riscv,raw-event-to-mhpmcounters`: 紀錄 selector value(s) 可使用的 HPM counters, 其 event_idx 都是 0x20000, 在 boot-time 會[紀錄在 hw_event_map](https://github.com/riscv-software-src/opensbi/blob/v1.2/lib/utils/fdt/fdt_pmu.c#L120-L121)，將來 perf 使用 raw event descriptor 並透過 `sbi_pmu_ctr_cfg_match()` 的 `event_data` 傳下來就能[比對](https://github.com/riscv-software-src/opensbi/blob/v1.2/lib/sbi/sbi_pmu.c#L592-L598)是否有支援該 raw event
        * `perf stat -e r21 <cmd>` 的 `r21`, 稱為 raw event descriptor, `sbi_pmu_ctr_cfg_match()` 的參數 `event_data` 就會是 `0x21`

下圖為簡化過的邏輯, runtime 時，也就是 userspace 用 perf 來計數某支程式，OpenSBI 就能從 boot-time 填好的 `hw_event_map[]` 與目前的 `active_events[]` 得知這個 `event_idx` 可以用哪個 HPM counter (得到 `ctr_idx`), 再透過 `fdt_pmu_evt_select[]` [將 `event_idx` 對應到 `select`](https://github.com/riscv-software-src/opensbi/blob/v1.2/platform/generic/platform.c#L247) 填入 `mhpmevent`  (raw event 不須對應，[data 即為 mhpmevent 的 selector value](https://github.com/riscv-software-src/opensbi/blob/v1.2/platform/generic/platform.c#L239-L240))，最後將這個 `event_idx` 填入 `active_events[hartid][ctr_idx]` 表示該 HPM counter 已被佔用 

![[../assets/pmu/hpm_counter_used.png]]


前文已介紹，pmu node 中三個 property 的用途，如果需要詳細資訊可以參考 device tree binding ，在 [OpenSBI](https://github.com/riscv-software-src/opensbi/blob/v1.2/docs/pmu_support.md) 與 [Linux kernel](https://github.com/torvalds/linux/blob/v6.4-rc5/Documentation/devicetree/bindings/perf/riscv%2Cpmu.yaml) 各有一份文件，但是 PMU node 僅在 OpenSBI boot-time 用來填 `hw_event_map` 與 `fdt_pmu_evt_select` 這兩個 table， U-Boot 或 Linux 不會用到, 所以 OpenSBI 在跳到 S-mode 前會[抹除 pmu node](https://github.com/riscv-software-src/opensbi/blob/v1.2/lib/utils/fdt/fdt_fixup.c#L312) 的資訊, 以下為 AE350 AX45MP 的 pmu node 範例：

 ```
 pmu {
         compatible = "riscv,pmu";
         device_type = "pmu";
         riscv,event-to-mhpmevent = <    0x3 0x0000 0x41>, /* D-Cache access */
                                    <    0x4 0x0000 0x51>, /* D-Cache miss */
                                    <0x10000 0x0000 0x61>, /* D-Cache load access */
                                    <0x10001 0x0000 0x71>, /* D-Cache load miss */
                                    <0x10002 0x0000 0x81>, /* D-Cache store access */
                                    <0x10003 0x0000 0x91>, /* D-Cache store miss */
                                    <0x10008 0x0000 0x21>, /* I-Cache access */
                                    <0x10009 0x0000 0x31>; /* I-Cache miss */
         riscv,event-to-mhpmcounters = <    0x3     0x4 0x78>,
                                       <0x10000 0x10003 0x78>,
                                       <0x10008 0x10009 0x78>;
         riscv,raw-event-to-mhpmcounters = <0x0  0x30 0xffffffff 0xffffffff 0x78>, /* Integer load instruction count */
                                           <0x0  0x40 0xffffffff 0xffffffff 0x78>, /* Integer store instruction count */
                                           <0x0  0x50 0xffffffff 0xffffffff 0x78>, /* Atomic instruction count */
                                           <0x0  0x60 0xffffffff 0xffffffff 0x78>, /* System instruction count */
                                           <0x0  0x70 0xffffffff 0xffffffff 0x78>, /* Integer computational instruction count */
                                           <0x0  0x80 0xffffffff 0xffffffff 0x78>, /* Conditional branch instruction count */
                                           <0x0  0x90 0xffffffff 0xffffffff 0x78>, /* Taken conditional branch instruction count */
                                           <0x0  0xA0 0xffffffff 0xffffffff 0x78>, /* JAL instruction count */
                                           <0x0  0xB0 0xffffffff 0xffffffff 0x78>, /* JALR instruction count */
                                           <0x0  0xC0 0xffffffff 0xffffffff 0x78>, /* Return instruction count */
                                           <0x0  0xD0 0xffffffff 0xffffffff 0x78>, /* Control transfer instruction count */
                                           <0x0  0xE0 0xffffffff 0xffffffff 0x78>, /* EXEC.IT instruction count */
                                           <0x0  0xF0 0xffffffff 0xffffffff 0x78>, /* Integer multiplication instruction count */
                                           <0x0 0x100 0xffffffff 0xffffffff 0x78>, /* Integer division instruction count */
                                           <0x0 0x110 0xffffffff 0xffffffff 0x78>, /* Floating-point load instruction count */
                                           <0x0 0x120 0xffffffff 0xffffffff 0x78>, /* Floating-point store instruction count */
                                           <0x0 0x130 0xffffffff 0xffffffff 0x78>, /* Floating-point addition instruction count */
                                           <0x0 0x140 0xffffffff 0xffffffff 0x78>, /* Floating-point multiplication instruction count */
                                           <0x0 0x150 0xffffffff 0xffffffff 0x78>, /* Floating-point fused multiply-add instruction count */
                                           <0x0 0x160 0xffffffff 0xffffffff 0x78>, /* Floating-point division or square-root instruction count */
                                           <0x0 0x170 0xffffffff 0xffffffff 0x78>, /* Other floating-point instruction count */
                                           <0x0 0x180 0xffffffff 0xffffffff 0x78>, /* Integer multiplication and add/sub instruction count */
                                           <0x0 0x190 0xffffffff 0xffffffff 0x78>, /* Retired operation count */
                                           <0x0  0x01 0xffffffff 0xffffffff 0x78>, /* ILM access */
                                           <0x0  0x11 0xffffffff 0xffffffff 0x78>, /* DLM access */
                                           <0x0  0x21 0xffffffff 0xffffffff 0x78>, /* I-Cache access */
                                           <0x0  0x31 0xffffffff 0xffffffff 0x78>, /* I-Cache miss */
                                           <0x0  0x41 0xffffffff 0xffffffff 0x78>, /* D-Cache access */
                                           <0x0  0x51 0xffffffff 0xffffffff 0x78>, /* D-Cache miss */
                                           <0x0  0x61 0xffffffff 0xffffffff 0x78>, /* D-Cache load access */
                                           <0x0  0x71 0xffffffff 0xffffffff 0x78>, /* D-Cache load miss */
                                           <0x0  0x81 0xffffffff 0xffffffff 0x78>, /* D-Cache store access */
                                           <0x0  0x91 0xffffffff 0xffffffff 0x78>, /* D-Cache store miss */
                                           <0x0  0xA1 0xffffffff 0xffffffff 0x78>, /* D-Cache writeback */
                                           <0x0  0xB1 0xffffffff 0xffffffff 0x78>, /* Cycles waiting for I-Cache fill data */
                                           <0x0  0xC1 0xffffffff 0xffffffff 0x78>, /* Cycles waiting for D-Cache fill data */
                                           <0x0  0xD1 0xffffffff 0xffffffff 0x78>, /* Uncached fetch data access from bus */
                                           <0x0  0xE1 0xffffffff 0xffffffff 0x78>, /* Uncached load data access from bus */
                                           <0x0  0xF1 0xffffffff 0xffffffff 0x78>, /* Cycles waiting for uncached fetch data from bus */
                                           <0x0 0x101 0xffffffff 0xffffffff 0x78>, /* Cycles waiting for uncached load data from bus */
                                           <0x0 0x111 0xffffffff 0xffffffff 0x78>, /* Main ITLB access */
                                           <0x0 0x121 0xffffffff 0xffffffff 0x78>, /* Main ITLB miss */
                                           <0x0 0x131 0xffffffff 0xffffffff 0x78>, /* Main DTLB access */
                                           <0x0 0x141 0xffffffff 0xffffffff 0x78>, /* Main DTLB miss */
                                           <0x0 0x151 0xffffffff 0xffffffff 0x78>, /* Cycles waiting for Main ITLB fill data */
                                           <0x0 0x161 0xffffffff 0xffffffff 0x78>, /* Pipeline stall cycles caused by Main DTLB miss */
                                           <0x0 0x171 0xffffffff 0xffffffff 0x78>, /* Hardware prefetch bus access */
                                           <0x0 0x181 0xffffffff 0xffffffff 0x78>, /* Cycles waiting for source operand ready in the integer register file scoreboard */
                                           <0x0  0x02 0xffffffff 0xffffffff 0x78>, /* Misprediction of conditional branches */
                                           <0x0  0x12 0xffffffff 0xffffffff 0x78>, /* Misprediction of taken conditional branches */
                                           <0x0  0x22 0xffffffff 0xffffffff 0x78>; /* Misprediction of targets of Return instructions */
 };
 ```

### RISC-V firmware event

一般 perf 有分 software event 與 hardware event 兩類，然而 RISC-V 與其他架構不同，有 SBI firmware 作為 M-mode runtime 服務 S-mode software 的運作，因此特別支援了 firmware event，本質上還是 software event, 不支援 sampling, 一般 OpenSBI 就有支援，如前文所述，每個 hart 各有 16 個 counter 

> OpenSBI v1.2 為全域且[長度 16 的 64bit 陣列](https://github.com/riscv-software-src/opensbi/blob/v1.2/lib/sbi/sbi_pmu.c#L69), 其實這樣挺浪費，因為它直接宣告 OpenSBI 可支援的 hart 數量上限 128，128 x 16 x 8bytes 為 16KiB，約佔 firmware 總體的 4%，因此，OpenSBI v.1.3 已引進 memory allocator, 改成放在[每個 hart 的 scratch 空間](https://github.com/riscv-software-src/opensbi/blob/v1.3/lib/sbi/sbi_pmu.c#L70)而不是全域陣列   

### Standard firmware event

> SBI spec 沒有提到 standard firmware event, 這裡僅為了跟 custom firmware event 區別所以稱之 standard

目前 SBI 定義以下 21 種 firmware event, 主要就是各種 exception 或 software/timer interrupt 相關的操作

| Firmware Event Name                  | Code | Description |
| --- | --- | --- |
| SBI_PMU_FW_MISALIGNED_LOAD           |    0 | Misaligned load trap event |
| SBI_PMU_FW_MISALIGNED_STORE          |    1 | Misaligned store trap event |
| SBI_PMU_FW_ACCESS_LOAD               |    2 | Load access trap event |
| SBI_PMU_FW_ACCESS_STORE              |    3 | Store access trap event |
| SBI_PMU_FW_ILLEGAL_INSN              |    4 | Illegal instruction trap event |
| SBI_PMU_FW_SET_TIMER                 |    5 | Set timer event |
| SBI_PMU_FW_IPI_SENT                  |    6 | Sent IPI to other HART event |
| SBI_PMU_FW_IPI_RECEIVED              |    7 | Received IPI from other HART event |
| SBI_PMU_FW_FENCE_I_SENT              |    8 | Sent FENCE.I request to other HART event |
| SBI_PMU_FW_FENCE_I_RECEIVED          |    9 | Received FENCE.I request from other HART event |
| SBI_PMU_FW_SFENCE_VMA_SENT           |   10 | Sent SFENCE.VMA request to other HART event |
| SBI_PMU_FW_SFENCE_VMA_RECEIVED       |   11 | Received SFENCE.VMA request from other HART event |
| SBI_PMU_FW_SFENCE_VMA_ASID_SENT      |   12 | Sent SFENCE.VMA with ASID request to other HART event |
| SBI_PMU_FW_SFENCE_VMA_ASID_RECEIVED  |   13 | Received SFENCE.VMA with ASID request from other HART event |
| SBI_PMU_FW_HFENCE_GVMA_SENT          |   14 | Sent HFENCE.GVMA request to other HART event |
| SBI_PMU_FW_HFENCE_GVMA_RECEIVED      |   15 | Received HFENCE.GVMA request from other HART event |
| SBI_PMU_FW_HFENCE_GVMA_VMID_SENT     |   16 | Sent HFENCE.GVMA with VMID request to other HART event |
| SBI_PMU_FW_HFENCE_GVMA_VMID_RECEIVED |   17 | Received HFENCE.GVMA with VMID request from other HART event |
| SBI_PMU_FW_HFENCE_VVMA_SENT          |   18 | Sent HFENCE.VVMA request to other HART event |
| SBI_PMU_FW_HFENCE_VVMA_RECEIVED      |   19 | Received HFENCE.VVMA request from other HART event |
| SBI_PMU_FW_HFENCE_VVMA_ASID_SENT     |   20 | Sent HFENCE.VVMA with ASID request to other HART event |
| SBI_PMU_FW_HFENCE_VVMA_ASID_RECEIVED |   21 | Received HFENCE.VVMA with ASID request from other HART event |

## How to enable perf sampling/filtering

要得知硬體是否支援 sampling/filtering，可由 boot log 中得知，首先 OpenSBI 會嘗試讀取 `scountovf` 這個 Sscofpmf 唯一新增的 CSR, 如果沒出 illegal instruction 代表硬體支援 Sscofpmf 

```
Boot HART ISA Extensions  : sscofpmf,time
...
Boot HART MHPM Count      : 4
Boot HART MIDELEG         : 0x0000000000003666
```

在 Linux 的 RISC-V PMU driver 初始化時，只會印出以下：

```
[    0.678004] riscv-pmu-sbi: SBI PMU extension is available
[    0.678553] riscv-pmu-sbi: 16 firmware and 18 hardware counters
```

進到 shell 後，可由 `/proc/cpuinfo` 確認 device tree 有給對

```
$ cat /proc/cpuinfo | grep isa
isa             : rv64imafdch_zicbom_zihintpause_zbb_sscofpmf_sstc
```

使用 `perf record` 之後會看到 irq-13 有在增加，會跟 sample 到的次數差不多

```
$ cat /proc/interrupts | grep riscv-pmu
 13:        190          0  RISC-V INTC  13 Edge      riscv-pmu
```

如果 Linux 開機過程出現以下訊息，代表 device tree 的 isa string 沒有 "sscofpmf", 這時即使硬體有支援也會無法使用 `perf record`

```
[    0.678773] riscv-pmu-sbi: Perf sampling/filtering is not supported as sscof extension is not available
```

以上，紀錄我研究 RISC-V PMU 標準流程，希望對於想要在 RISC-V 上使用 perf 工具的人有所幫助。:)
