---
title: UPX Dynamic Unpacking for ELF
date: 2020-11-21 08:00:00 +0800
categories: [ Reverse ]
tags: [ Linux ]     # TAG names should always be lowercase
---

## 前言
一直以來都覺得被 UPX 壓縮的執行檔可以利用官方的 UPX utility 來脫殼，但今天碰到了一隻有趣的惡意程式，其作者對傳統 UPX 殼做了一些手腳，導致無法直接使用 UPX Utility 來做，因此這次就將自己動態脫殼的步驟給記錄一下

## Tools & Env.
不像 Windows 有 x64dbg 這類超方便脫殼的工具，在 Linux 上我比較習慣：
- `IDA Pro`
- `gdb server`
- `readelf` or `elf parser`

測試環境：
虛擬機(ubuntu 16.04 x86_64)
主機(Mac OS 10.15.16)

## UPX Introduce
UPX 是一種在 Windows 和 Linux 都相當常見的壓縮套件，而這些壓縮套件常被用於惡意程式，透過將執行檔壓縮可以一定程度的規避掉防毒軟體的檢查，像是以 rule-based 的 yara 就幾乎無法在加殼後的程式找到特徵。

這次對象是一個 ELF32 的惡意程式，初步檢查這隻惡意程式有用UPX 加殼，但是在 UPX 特徵字串做了更改，讓其無法使用正規的 UPX 來解開。

在 LINUX 的環境下UPX 只能在執行檔是 static-link 的狀態下使用，這可能的原因是 UPX 不想幫 ELF 修動態library binding 的動作，為了理解 UPX 脫殼順序我們可以先寫一個簡單的c 語言，並且把他編譯成 static-link，並把它用 upx 加殼。
```
gcc -static ./a.c
upx ./a.out
readelf ./a.out
```
結果如下圖，大部分的 header 都會被抹掉，只留下 ELF header 和 Program header 來讓程式可以正常運行

![readelf](/image/2020-11-21/readelf.png)

經過一些測試我發現了一個蠻快可以跳到 oep (original entrypoint) 的方式，這邊大家可以照我的步驟走，基本上就可以成功脫殼了


## Dynamic Unpacking
打開靜態分析工具 ida 會發現 function 蠻少的，接著找到編排像下圖的function，這個 function 會做像是 sysmmap 和 sysmprotect 的動作來初始化環境，因此直接下 breakpoint 在 function 最後的 `jmp r13`，接著進入動態 debugging

![step1](/image/2020-11-21/step1.png)

設定好環境後利用 remote debug
```
gdbserver 0.0.0.0:1234 ./a.out
```

一開始停在 start 的頭，直接按下 F9 跳到 breakpoint，接著就 F7 跳進 新開的空間，這時候會停在一個 call 上
![step2](/image/2020-11-21/step2.png)

接著跳進這個 call 的 function，會看到三個往前的短跳躍，這個標記非常好認，如果有看到代表有跟到這個步驟，最後在這個function的最後面會看到一個長跳躍


![step3](/image/2020-11-21/step3.png)

```
MEMORY:00007FFFF7FF986C pop     rax
MEMORY:00007FFFF7FF986D jmp     qword ptr [r14-8]
```

長跳躍後就會看到短短的三行，繼續F7就會看到 IDA 提示 RIP 被更改了，這就代表要跳回真正的 _start 了

![step4](/image/2020-11-21/step4.png)

接著就可以如下圖的樣子，身為一個逆向人員應該很常見到的 _start 程式碼佈局
![step5](/image/2020-11-21/step5.png)

成功脫殼後，接下來要把真正的執行檔dump出來，這部分已經有一些教學文有寫好code了，這邊直接貼兩篇，都可以完美 dump ELF 的文章

[[原創]ELF64手脫UPX殼實戰](https://bbs.pediy.com/thread-255519.htm)<br>
[linux 下 upx 脱壳笔记](https://www.pediy.com/kssd/pediy10/79061.html)

這兩篇其實提供的程式碼邏輯都相同，只是差在 64 和 32 bits 的差別，大致邏輯是因為 UPX 在最後會修改 program header ，跳到 OEP 後 program header 所記錄的會是真正的 test segment 和 data segment，利用這兩個來撈出 ELF 檔。

到這邊就可告一個段落，但把dump 丟出來得檔案和用 upx -d 解出來的比較一下還是可以發現 upx 的比較齊全，主要原因是dump 出來的只有 `PT_LOAD` 的 segment，也就是會被load 進 memory 的區段，其他section 像是 section Header 和存放 symol 的 section 都不會被 dump 下來，因此這部分如何修復dump 下來的section header 又是一個蠻大的問題，並且會與 UPX 如何實作有關，等有空我再好好研究。


## FLIRT: function symbol 修復

function symbol 很常被惡意程式作者給 strip 掉，因此在做分析時很常需要去解這個問題，因此就搭配上面的 UPX 做一個簡單的組合技。

FLIRT 是 ida 的工具可以用它提供的程式來建立library 的 特徵，利用這個特徵來識別執行檔是否有相應的 library。

當要辨識一隻被 strip 過並且又是static link 的執行檔函式庫時，可以先利用 strings 來查看是否有殘留編譯器或是 libc 資訊藉此來確定要編譯哪個library

```
> strings imsd_unpack | grep "GCC"

GCC: (Alpine 5.3.0) 5.3.0
```

發現是由 Alpine 編譯出來的，Alpine 是一款小型的linux 系統，在library 上使用的是 musl 而非傳統的 Glibc，所以網路上一些可用的特徵檔 
[sig-database](https://github.com/push0ebp/sig-database)、[ALLirt](https://github.com/push0ebp/ALLirt) 無用武之地，只好自己編了。

由於我們只有 gcc 的版本，所以接下來要找出 library 的版本，一開始在網路上找到[官網](https://pkgs.alpinelinux.org/packages?name=gcc&branch=v3.3)有 tool 的版本和對應的 Alpine 版本，從這可以知道要使用 Alpine 3.3版

有了版本就可以建立一個 Alpine 3.3 docker，進去查看是用哪版的 musl 的 library

![FLIRT_step1](/image/2020-11-21/FLIRT_step1.png)

接著[下載](https://musl.libc.org/releases.html
)編譯對應版本的library，得到 libc.a 後利用工具將它變成 sig 的形式，指令如下


```sh
./pelf libc.a libc.pat

./sigmake -n"libc-x86_64-linux-musl.sig" libc.a.pat libc.a.sig
```
假如執行 sigmake 有噴錯代表有一些 signature 有衝突，當前目錄應該會多 test.exc，這個就是紀錄錯誤檔，打開後依照自己喜好更改衝突的 function，也可以直接把下面三行刪除，排除錯誤

```
;--------- (delete these lines to allow sigmake to read this file)
; add '+' at the start of a line to select a module
; add '-' if you are not sure about the selection
; do nothing if you want to exclude all modules
```
最後將這個 sig 檔放入 ida 的 /sig/pc 底下，開啟 ida 後 control + F5，開啟sig 視窗，將剛剛做好的sig檔匯入進來

結果如下，成功辨識出惡意程式中的 183 個 函式庫 function，這樣逆向就少了很多不確定因素了~~~~
![FLIRT_step2](/image/2020-11-21/FLIRT_step2.png)

## 總結
會寫這篇主要是ELF的 UPX 文章偏少，我上面提到的那兩篇BLOG 在 ida trace 那塊真的是蠻不清楚的，所以我記錄下我的步驟，讓對於 UPX 比較不熟的讀者可以跟著步驟做並解出 ELF，我自己一開始在動態解 UPX 的時候是撞到蠻多坑的，至於後續做 function symbol 主要是給我自己當備忘錄，因為太常用到了，又懶得重找一遍步驟。

## 其他
變種 UPX 修復
https://zhuanlan.kanxue.com/article-4671.htm
https://thinkycx.me/2019-07-15-how-to-use-signature-file-in-IDA.html
https://xz.aliyun.com/t/4484
