---
title: ELF Header Introduction 0x01
date: 2019-7-20 08:00:00 +0800
categories: [ Reverse ]
tags: [ Linux ]     # TAG names should always be lowercase
---
## Motivate
會想寫這篇是因為目前在做惡意程式分析時，我們發現很多惡意程式會為了隱蔽自己而更改 ELF 的header 或是其他地方，增加研究人員分析的困難度。


## Foreword
在 Linux 作業系統中，可執行的程式檔或是編譯出得目的檔都會是 ELF (Executable and Linkable Format) 格式的，當然也有其他的格式，像是 windows 所使用的 PE 和 Mac os 的 Mach-O，每種作業系統不能兼容其他格式，只能透過使用者另外安裝程式來達成跨平台，像是如果要在linux上執行 PE，可以透過 wine 來解決這個問題。再來我就來好好的介紹這些可執行檔的格式。

## Useful tools

這些工具可以幫忙我們觀察檔案
* file
* xxd
* objdump
* lld



## 0x00 | ELF Header

程式的 Header 就像是一個人的外觀，記載了一個執行檔所有的特徵，我們可以先用 `file` 這個指令觀察一下這隻程式的型態

```
in0de@ubuntu:~/Desktop/test/test2$ file main
main: ELF 32-bit MSB executable, MIPS, MIPS32 rel6 version 1 (SYSV), dynamically linked, interpreter /lib/ld-, for GNU/Linux 4.16.0, not stripped
```
從這個指令的結果可以詳細的了解到一個可執行檔的資訊，我放在下面，但這些是 `File` 程式告訴我們的，那接下來就來驗證看看是否有錯誤吧

File Format :   ELF<br>
Endianness  :   MSB ( Big-endian )<br>
Architecture:   MIPS32 Release 6<br>
Library link:   dynamically linked<br>
kernel version: GNU/Linux 4.16.0<br>
symble table:   not stripped

### 0x00 ~ 0x10 bytes
```
in0de@ubuntu:~/Desktop/test/test2$ xxd ./main | grep 00000000
00000000: 7f45 4c46 0102 0100 0000 0000 0000 0000  .ELF............
```

#### File Format
利用 xxd 來檢視可執行檔的 binary，開頭四個位元是 ELF 格式的 Magic Number [7f 45 4c 46]

#### Endianness & Architecture Bit & ELF Version
在 Magic Number 後面接著是 [ 01 02 01 ]，第一個 byte 代表 Architecture Bit; 01 指的是 32 bit; 02 指的是 64 bit
第二個 byte 代表 Endianness; 01 指的是 LSB; 02指的是 MSB。第三個 byte 代表 程式所使用的 ELF Version，接下來是是8個 bits 的 buffer，
可以忽略

### 0x10~0x20
in0de@ubuntu:~/Desktop/test/test2$ xxd ./main | grep 00000010
00000010: 0002 0008 0000 0001 0040 0540 0000 0034  .........@.@...4

####  Flie Type
0x21,0x22 是一個 2 bytes 的資訊。因此會跟前面提到的 Endianness 有關係，假如跟例子一樣是
MSB 的話就是正常看 -> 0x0002，但如果是 LSB就必須倒過來看 -> 0x0200。
確定數值後就可以找出是哪種檔案了，而 0x0002 代表的是可執行檔

#### Machine Type
接下來的 0x23,0x24 為 Ｍachine type，例如 arm, x86_64, ppc 等等的，
這些代號相當多，我就不列了，下面的reference 有附上相關連結，
這邊的例子 0x0008 是 MIPS，之吼的參數比較不會用到，先跳過


## 0x01 | Try To Modify Metadata
許多 malware 會透過更改一些程式的 hex，來混淆或是增加資安人員在做行為分析的困難，因此遇到這種
 malware 時，記得檢查看看他的 hex 是否故意更改

### Install hexeditor
sudo apt install ncurses-hexedit

### Modify hex

把 endianness byte 改掉會變成下面這樣，無法解析出後面的東西，但是程式還是可以順利執行，其他的就讓各位去探索吧
```
in0de@ubuntu:~/Desktop/test/test2$ file mainI386el
mainI386el: ELF 64-bit MSB *unknown arch 0x3e00* (SYSV)

in0de@ubuntu:~/Desktop/test/test2$ ./mainI386el
GOGOGOG
```


## reference
http://llvm.org/doxygen/Support_2ELF_8h_source.html

https://paper.seebug.org/papers/Archive/refs/elf/Understanding_ELF.pdf
