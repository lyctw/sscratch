---
title: NAS 配置紀錄
tags:
  - NAS
  - ARM
date: 2023-12-22
---

最近終於完成了第一組 NAS（Network Attached Storage）系統的設置。過去，我一直使用 Google Drive 作為主要資料儲存方式，後來移到 Seagate One Touch HDD 外接硬碟，但經常因為操作不慎而覆寫資料，外加對其穩定性不夠信任，一直讓我很想轉移到  NAS，然而預算總是很吃緊（窮），最近終於下定決心，就當作一次性消費給他買下去，畢竟這種東西肯定是越早買越好，不然對雲端服務的依賴性只會越來越強。

## 選擇與規劃

在規劃階段考慮了幾個方案。群暉那樣專用 NAS 主機雖然方便，開箱即可使用，但因為更喜歡自己動手組裝並安裝作業系統[^f3]的過程。因此，最後選擇了 CM3588 NAS SDK 的解決方案 [^f1]，是 ARM based 的 RK3588 SoC（四核 Cortex-A76 + 四核 Cortex-A55 架構），並配備 4 條 M.2 插槽，足夠以 RAID5 的配置充分利用硬碟容量同時確保一定的容錯能力。

![[../assets/cm3588_nas/Pasted image 20241222195026.png]]

[^f3]: 作業系統是 OpenMediaVault, 是 Debian-based 的系統，可以支援 Software RAID
[^f1]: 其他候選的平台有同一個 SoC 的 [Radxa ROCK 5 ITX](https://radxa.com/products/rock5/5itx/), x86 的 N100 的[NVMe 擴展板](https://cwwk.net/collections/frontpage/products/x86-p5-development-version-special-machine-4-m-2-nvme-adapter-board-only-applicable-to-cwwk-x86-p5-n100-i3-n305-model-%E7%9A%84%E5%89%AF%E6%9C%AC) 或著是 RISC-V based 的 [Milk-V Jupiter](https://cwwk.net/collections/frontpage/products/x86-p5-development-version-special-machine-4-m-2-nvme-adapter-board-only-applicable-to-cwwk-x86-p5-n100-i3-n305-model-%E7%9A%84%E5%89%AF%E6%9C%AC) 加上這種 [SATA 擴展板](https://www.aliexpress.com/item/1005004249579141.html?spm=a2g0o.productlist.main.1.21b14815jW6cal&algo_pvid=03c5357e-1633-4231-b69c-0e0f4f7728ce&algo_exp_id=03c5357e-1633-4231-b69c-0e0f4f7728ce-0&pdp_npi=4%40dis%21TWD%211105.56%211105.56%21%21%2133.90%2133.90%21%402140c1e917348673947394774e68b4%2112000028519592199%21sea%21TW%210%21ABX&curPageLogUid=kxE30Zuvnfli&utparam-url=scene%3Asearch%7Cquery_from%3A)，但附近的二手電腦買不到足夠這麼多 SATA 電供的機殼，只好放棄...

## 散熱與效能表現

使用過程中，散熱是需要注意的問題， 可能影響硬碟壽命。雖然待機時 CPU 的使用率保持在 10% 以下，但偶爾散熱片會達到 48.1°C，M.2 SSD 平時保持室溫，工作時溫度會升至 38°C。因此主動冷卻可能是必要的，但機殼只可以在 CPU 散熱片加的風扇，但是插上了卻不會轉動:
![[../assets/cm3588_nas/Pasted image 20241222195135.png]]
還特地跑附近電料行請他們幫忙測試，目前懷疑是 PWM 控制少了，看這個 [kernel patch](https://lore.kernel.org/linux-arm-kernel/20240616215354.40999-2-seb-dev@mail.de/T/) 應該有對應的驅動要打開？ ... 先留個坑，好險 FriendlyElec 的軟體支援做得相當不錯，要自己編譯 kernel 有提供 v6.1 的[原始碼及指令](https://wiki.friendlyelec.com/wiki/index.php/CM3588#Build_the_code_using_scripts)。

[^f2]: 

## 成本與效益的權衡

整套系統的採購清單約 13,000 新台幣:

| 購買平台      | 物件                                                 | 價格            |
| ------------ | --------------------------------------------------- | ---------------- |
| FriendlyElec | [CM3588 Plus](https://www.friendlyelec.com/index.php?route=product/product&product_id=299&search=CM3588&description=true&category_id=0&sub_category=true)                                         | US$179           |
|              | RK3588 SoM (16GB RAM - 64GB eMMC)                   | US$125           |
|              | NAS Kit                                             | US$35            |
|              | 12V 4A Universal Power Adapter                      | US$10            |
|              | Shipping fee                                        | US$9             |
| PChome       | 4條 MSI SPATIUM M371 1TB NVMe M.2 SSD (NT$1590/條)  | NT$6,360         |
| Aliexpress   | Metal Aluminum Case                                 | NT$860           |
|              | Case + Fan                                          | NT$583           |
|              | Shipping fee                                        | NT$277           |
|              |                                                     | Total: NT$13,120 |

雖然體積小是優勢，但相比商用 NAS，其實在 C/P 值上並不算高。乙太網口的傳輸速度 2.5Gbps 是瓶頸，但目前對我的需求已經足夠。以 4 條 SSD 組成 RAID5 配置後，實際容量約 3TB。與雲端儲存相比，這套系統需要 2.5 年才能回收成本。不過，如果未來再添購一台 NAS，屆時可以將這組系統轉為備份空間。


## 結語

過程還是比較有成就感，雖然成本不低，但對於熱愛動手組裝與探索技術細節的人來說或許值得。

