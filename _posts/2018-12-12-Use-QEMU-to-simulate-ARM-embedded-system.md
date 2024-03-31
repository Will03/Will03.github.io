---
title: Use QEMU to simulate ARM embedded system
date: 2018-12-12 08:00:00 +0800
categories: [IoT, Emulation]
tags: [iot]     # TAG names should always be lowercase
---


最近要學著找 IOT 設備上的漏洞
因此需要在自己的電腦中模擬 IOT 設備
於是就來寫個教學吧

## 前言
這篇教學會利用 qemu 模擬 IOT 開發板 Versatile Express a9
並且在上面放上基本的根目錄系統

## 操作環境
ubuntu 16.04
gcc version 5.4.0
qemu-arm version 2.5.0

## (一) 生成 linux kernel Image 檔

可選擇想要使用的kernel 版本
注意：下載3.x版以前的可能會因為本身ubuntu版本過新，導致有些套件需要降版，像是make和gcc 等等的

### 下載
``` shell
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.9.99.tar.xz
```
### 編譯

安装arm交叉編譯套件
```
sudo apt-get install gcc-arm-linux-gnueabi
```

解壓縮剛剛下載的壓縮檔並進入資料夾
```
tar Jxvf linux-4.9.99.tar.xz
cd linux-4.9.99
```

生成 Versatile Express 開發板的 config 文件
```
make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm vexpress_defconfig
```
管理相關設定
```
make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm menuconfig
```

執行指令後會進入選單，請設定以下項目
```
System Type -->
    [ ] Enable the L2x0 outer cache controller
    取消此選項
Kernel Features -->
    [*] Use the ARM EABI to compile the kernel
    確保是開啟狀態
``` 
開始進行交叉編譯
```
make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm
```

編譯完成後，我們會找到以下兩個我們要的檔案

- arch/arm/boot/zImage 
    - 作業系統映像檔

-  arch/arm/boot/dts/vexpress-v2p-ca9.dtb
    - 存放實體設備訊息

將這兩個檔案拉出來放到另外的資料夾之後會比較方便
```
cp arch/arm/boot/zImage ../
cp arch/arm/boot/dts/vexpress-v2p-ca9.dtb ../
```


### (二) QEMU安装

qemu會因為版本的問題出現很多問題，下載方式有兩種
1. 使用 os 本身package包
2. 下載官方程式碼自行編譯

第1種方式所下載到的qemu不會是最新版的，可能會有問題
但是比第2種快很多

**我這邊使用的是第1種**

安裝方式
```
sudo apt-get install qemu
```

完成後可以拿我們剛剛編譯好的linux kernel 測試
進入剛剛複製出 zImage 和 vexpress-v2p-ca9.dtb的資料夾中
新增一個 .sh檔
將下面程式碼複製到該sh檔並執行
``` 
#!/bin/bash

qemu-system-arm \
-M vexpress-a9 \
-m 512M \
-kernel zImage \
-dtb vexpress-v2p-ca9.dtb \
-nographic \
-append "console=ttyAMA0" 
```

如果成功會出現以下類似畫面

![](https://i.imgur.com/YnkUl2S.png)

最後一行報錯是因為我們未放入根目錄檔因此停在這邊

### (三) 製作根文件系统

一般嵌入式設備的根目錄都相當簡單，大部份是使用 busybox 搭配自己要使用的服務而已
因此我這邊就使用busybox當作根目錄

#### 下載與交叉編譯 busybox
```
wget https://busybox.net/downloads/busybox-1.27.2.tar.bz2
tar xjvf busybox-1.27.2.tar.bz2
cd busybox-1.27.2
make defconfig
make menuconfig
- Support /etc/networks
- Support external DHCP clients 

make CROSS_COMPILE=arm-linux-gnueabi-
make install CROSS_COMPILE=arm-linux-gnueabi-
```

在資料夾下創建rootfs
``` 
mkdir rootf
```

將busybox中編譯得到的檔案放進rootfs中
```
cp -r busybox-1.27.2/_install/* rootfs/
```

建立4個tty終端設備 (c 代表字符設備，4是主設備號，1~4為次設備號)
```
mkdir -p rootfs/dev
mknod rootfs/dev/tty1 c 4 1
mknod rootfs/dev/tty2 c 4 2
mknod rootfs/dev/tty3 c 4 3
mknod rootfs/dev/tty4 c 4 4
```

放入lib
```
sudo cp -P /usr/arm-linux-gnueabi/lib/* rootfs/lib/
```

生成一個空镜像
```
dd if=/dev/zero of=a9rootfs.ext3 bs=1M count=32
```


將他格式化生成ext3文件系统
```
mkfs.ext3 a9rootfs.ext3
```


將剛剛建立的rootf資料夾複製進鏡像檔中
```
mkdir tmpfs
sudo mount -t ext3 a9rootfs.ext3 tmpfs/ -o loop
sudo cp -r rootfs/* tmpfs/
sudo umount tmpfs
```

這樣就完成了鏡像檔設置

更改之前測試qemu的 shell檔
```
#!/bin/bash

qemu-system-arm \
-M vexpress-a9 \
-m 512M \
-kernel linux-4.14.7/arch/arm/boot/zImage \
-dtb linux-4.14.7/arch/arm/boot/dts/vexpress-v2p-ca9.dtb \
-nographic \
-append "root=/dev/mmcblk0 console=ttyAMA0 rw init=/linuxrc" \
-sd a9rootfs.ext3

```

執行shell 這樣就完成了基礎模擬嵌入式設備
![](https://i.imgur.com/wMGqYYr.png)


## telnet  功能
https://zhuanlan.zhihu.com/p/28467328
參考此網站配置

進入後更改passwd 
輸入 telnetd 指令
即可打開port


## 使用buildroot

https://dzone.com/articles/an-arm-image-with-buildroot
