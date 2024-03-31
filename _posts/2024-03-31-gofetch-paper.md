---
title: Introduce GoFetch Vulnerability in Apple M-Series Chips
date: 2024-3-31 1:00:00 +0800
categories: [ Paper ]
tags: [ macos, apple ]     # TAG names should always be lowercase
---

<style>

.text-center{
    text-align: center; //文字置中
}
.text-left{
    text-align: left; //文字靠左
}
.text-right{
    text-align: right; //文字靠右
}

</style>


## 📌介紹

了解這個手法前先來聊聊背景，在蘋果的 macOS 嚴格限制行程間是不能互相存取記憶體內容，除非有特殊的 entitlement。而本篇想探討的就是嘗試用硬體內的行為繞過軟體防護機制透過 cache 來竊取其程式的記憶體內容。

針對 cache 的攻擊曾出過像是 Meltdown 和 Spectre，而這次的漏洞處在 Apple 晶片的 DMP(Data Memory-Dependent Prefetchers)，既然他的名稱有個 Prefetcher 你應該可以猜到他就是一種預先載入的機制。

與一般 prefetcher 不同處在於當他遇到內容是指標的時候會將指標的記憶體 dereference(解參考)後放進 cache，減少下一次遇到要解參考的時間。

![](/image/2024-03-31/1.png)

graph made by  `[3]`

例如當 arr[n+1] 指向一個 address 時，DMP 會儲存 *arr[n+1]。

![](/image/2024-03-31/2.png)

graph made by `[3]`

## 📌攻擊情境

機制懂了那就來用作者給的簡單範例來講解一下吧，以下是受害行程執行的演算法，在這邊攻擊者的目的是試圖找到 secret 的內容。

```
void ct-swap(uint64_t secret, uint64_t *a, uint64_t *b, size_t len) {
    uint64_t delta;
    uint64_t mask = ~(secret-1);
    for (size_t i = 0; i < len; i++) {
            delta = (a[i] ^ b[i]) & mask;
            a[i] = a[i] ^ delta;
            b[i] = b[i] ^ delta;
    }
}
```

首先攻擊情境中總共有三隻行程，攻擊者可以要求加密行程進行加密請求，並且指定要加密的內容(b or a)
➢ Victim - 負責執行 `ct-swap` 的行程其中包含 secret 
➢ AttackerA - 負責請求 Victim 執行 `ct-swap` 以及指定 a 或是 b 的內容
➢ AttackerB - 負責執行經典的 Prime+Probe [1] 攻擊，算出另一個陣列的內容

⚔ 攻擊流程：
1. 攻擊者隨機挑選 a 或 b 陣列輸入（這邊假設 b），接著將 b 陣列中的其中一個位置填入 ptr (ptr 指的是攻擊者決定的記憶體地址)
2. 開始 ct-swap 時，攻擊者觀察是否因為 a 和 b 在做交換時讓 a 陣列出現 ptr，當 a 陣列出現 ptr 時
3. 好玩的就在這，DMP 的機制會因為 a 陣列出現 ptr 而觸發 (因為陣列中出現 address，所以 DMP 將其內容放到 cache)
4. 然後攻擊者使用傳統的快取旁路攻擊（Prime+Probe）來觀察 ptr 是否由於 ct-swap 對 a 的計算而被 DMP 解參考
5. 這樣多次重複執行後(3200 次)即可透過檢查分布圖推斷 secret 的內容是 1 或是 0

![](/image/2024-03-31/3.png)

graph made by `[2]`


說到這邊大家可能會有個疑問，阿攻擊者控制的 b 陣列原本的 ptr 也會進到 DMP cache 呀，那怎麼分辨哪個是來自 a 陣列的 cache，實際上做者還有透過其他方式將 a 陣列不斷從 cache 移除（evicting），但為了篇幅容我省略這部分。

## 📌總結

簡單的解釋大概是這樣，聽完有興趣的人可以嘗試閱讀補充的兩篇論文，就會看到 golang 加密方式被攻擊的手法，事實上這不是第一次針對 DMP 的攻擊，但可能是相對完整，我會覺得這種方式離穩定在實際惡意程式上執行還有一段距離。

已使用者的角度來說，從每次蘋果安全更新的數量來看或許真的該擔心的是軟體所導致的漏洞吧😆，但這或許也是個警示在這個硬體不斷推層出新且搶進硬體的市場或許在硬體變革的同時也需要了解加速背後的安全考量。

## 📌Ref
- [1] Prime+Probe: 是一種已經行之有年的攻擊手法，透過不斷的將 cache 寫滿來檢查每次哪邊有出現 cache miss 來推測內容
- [2] GoFetch: Breaking Constant-Time Cryptographic Implementations Using Data Memory-Dependent Prefetchers https://gofetch.fail/
- [3] Augury: Using Data Memory-Dependent Prefetchers to Leak Data at Rest https://ieeexplore.ieee.org/document/9833570/