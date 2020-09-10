---
title: Android Security Introduce Part01  Architecture Introduce
date: 2019-5-27 08:00:00 +0800
categories: [ Reverse ]
tags: [ android ]     # TAG names should always be lowercase
---
## 動機
這一系列的文章會探討 Android Security，這個主題非常龐大，歷史也相當悠久，目前Ａndroid 的穩定版本為 Ａndroid Pie (Version 9)，而 Ａndroid Q 已經釋放了beta版，每代更新就象徵著 Android Source code 越來越複雜，所以我決定來推出這系列的文章帶大家慢慢了解 Ａnroid 生態圈。


## Android Architecture

首先，先來介紹android的整體架構，那我們就使用 buttom-up 的方式，由下往上介紹

<img src="/image/2019-5-27-Arch.png" alt="drawing" width="600"/>

### Linux kernel
就跟所有電子設備一樣，android 系統也需要去控制週邊設備，像是鏡頭、螢幕、喇叭、電池等等的，因此Ａndroid 使用了經過一些特別加工的 Linux kernel 來當作系統的最底層，這些加工內包括像是 Low Memory Killer（是一種記憶體管理系統，可以更有效的去保存記憶體）、wake locks （協同上層的 Power Manager 對電池做管理的一個系統），這些新增的功能都不會對原本的 Driver 造成影響。


### Hardware abstraction layer (HAL).
HAL 是很重要的一層，他的住要目的是建立一個上層系統與底層 Driver 溝通的管道，HAL 定義了一個協定，只要是使用 android 系統的每個設備都必須符合這個規範，讓底層在實作function的時候，不會干擾到上層的系統，上層系統在開發 service 的時候也不用管底層是怎麼寫的，兩邊互不干擾，

### System services
Android 主要分成兩群的 service，一群是用來管理系統的服務像是Window Manager、 Notification Manager 等等的，另一種是處理多媒體的服務像是錄音，遊戲音樂等等的。並且必須建立起連接 API 的韓式，讓 Application Framework 可以與 System Service 溝通來取得底層的服務，這層也包含 Core Library 和 ART的部分後去會再作介紹。

### Binder IPC
全名是（ Binder Inter-Process Communication (IPC) mechanism ），這機制允許 application 跨越 process boundaries 去呼叫到 System Service 的函式，開啟高等級的 framework APIs，與System Service 互動。

### Application framework
這層就是開發者熟悉的 Application framework，包含 Ａctivity、Content Provider、。


### HAL interface definition language (HIDL)
在Android 8.0 後的系統架構中，為了讓 vender 可以更快速並花更低的成本更新硬體，因此在 8.0 版中詳細指明了 HAL 和 users 的介面，讓android 架構能夠替換而不用重建 HALs


## 小結
這邊淺淺淺介紹完了 Android 的架構，其實蠻多東西沒講到的，但大致上的整體架構都有摸到，再來就是每個部分去深入研究，感謝大家觀看我的文章

## Reference

https://source.android.com/setup


如果有任何問題歡迎私訊我的粉專
https://www.facebook.com/In0de-Security-Memo-2649518185123453/
