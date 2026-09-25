# 1. 准备 ROM

获取 ROM 网址：
`https://hyperos.fans/zh/devices/zorn`

K80 EEA 线刷 ROM：
`zorn_eea_global_images_OS3.0.302.0.WOKEUXM_20260715.0000.00_16.0_eea_666bcd4cc8.tgz`

K80 国行线刷 ROM：
`zorn_images_OS3.0.306.0.WOKCNXM_20260708.0000.00_16.0_cn_4173919e32.tgz`

手机刷 EEA 固件，从国行固件提取文件。

Windows 解压国行固件后，把里面的：
`images\super.img`

复制到 Ubuntu，例如：
`~/k80_nfc/super.img`

安装 sparse image 工具：
```Bash
mkdir -p ~/k80_nfc

sudo apt install android-sdk-libsparse-utils \
    pkg-config libblkid-dev libdevmapper-dev libmd-dev build-essential -y

simg2img ~/k80_nfc/super.img ~/k80_nfc/super_raw.img

cd ~/k80_nfc
git clone https://gitlab.com/flamingradian/make-dynpart-mappings dynpart-src

cd dynpart-src

make clean && make USERSPACE=1

sudo losetup --find --show --read-only ~/k80_nfc/super_raw.img
```

根据上一步返回的结果决定下一步的参数，我这里得到的是：
`/dev/loop12`

所以：
```
cd ~/k80_nfc/dynpart-src
sudo ./make-dynpart-mappings /dev/loop12 0
```

然后挂载需要的国行分区，例如：
```
sudo mkdir -p /mnt/k80_product
sudo mount -o ro /dev/mapper/product_a /mnt/k80_product
```

# 2. 从国行 ROM 提取四套组件
## 2.1 先确认四个组件确实存在
```Bash
sudo find /mnt/k80_product -type f \( \
-name 'MITSMClient.apk' -o \
-name 'UPTsmService.apk' -o \
-name 'MINextpay.apk' -o \
-name 'MIpay.apk' \
\)
```

找到：
```Bash
/mnt/k80_product/app/MITSMClient/MITSMClient.apk
/mnt/k80_product/app/UPTsmService/UPTsmService.apk
/mnt/k80_product/app/MINextpay/MINextpay.apk
/mnt/k80_product/data-app/MIpay/MIpay.apk
```

UPTsmService 还要检查 native libraries：
```Bash
sudo find /mnt/k80_product/app/UPTsmService -type f
```

除了 APK，应当还能看到 lib/arm64/ 下的 9 个 .so。

## 2.2 建立 Magisk 模块目录
```Bash
mkdir -p ~/k80_nfc/K80-CN-NFC/system/product/app/MITSMClient
mkdir -p ~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService
mkdir -p ~/k80_nfc/K80-CN-NFC/system/product/app/MINextpay
mkdir -p ~/k80_nfc/K80-CN-NFC/system/product/data-app/MIpay
```

## 2.3 复制 MITSMClient
```Bash
cp -a \
/mnt/k80_product/app/MITSMClient/MITSMClient.apk \
~/k80_nfc/K80-CN-NFC/system/product/app/MITSMClient/
```

## 2.4 复制 UPTsmService，包括 9 个 .so
整个目录内容一起复制：
```Bash
cp -a \
/mnt/k80_product/app/UPTsmService/. \
~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService/
```

这样会同时得到：
UPTsmService.apk
lib/
└── arm64/
    ├── *.so
    └── ...

而不是只复制 APK。

## 2.5 复制 MINextpay
```Bash
cp -a \
/mnt/k80_product/app/MINextpay/MINextpay.apk \
~/k80_nfc/K80-CN-NFC/system/product/app/MINextpay/
```

## 2.6 复制 MIpay
```Bash
cp -a \
/mnt/k80_product/data-app/MIpay/MIpay.apk \
~/k80_nfc/K80-CN-NFC/system/product/data-app/MIpay/
```

## 2.7 检查最终提取结果
执行：
```Bash
find ~/k80_nfc/K80-CN-NFC/system/product -type f | sort
```

应该看到：
```Bash
~/k80_nfc/K80-CN-NFC/system/product/app/MINextpay/MINextpay.apk
~/k80_nfc/K80-CN-NFC/system/product/app/MITSMClient/MITSMClient.apk
~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService/lib/arm64/libcwlive.so
~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService/lib/arm64/libDeepNetV2.so
~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService/lib/arm64/libenlic.so
~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService/lib/arm64/libentryexpro.so
~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService/lib/arm64/libupbio_net.so
~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService/lib/arm64/libupliveness.so
~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService/lib/arm64/libuptsmaddonmi.so
~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService/lib/arm64/libuptsmservice.so
~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService/lib/arm64/libxdjacrypto.so
~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService/UPTsmService.apk
~/k80_nfc/K80-CN-NFC/system/product/data-app/MIpay/MIpay.apk
```

# 3. 建 Magisk 模块目录
我们的工作目录最后是：
`~/k80_nfc/K80-CN-NFC/`

结构为：
```Bash
~/k80_nfc/K80-CN-NFC$ tree ./
./
├── customize.sh
├── module.prop
├── post-fs-data.sh
├── system
│   └── product
│       ├── app
│       │   ├── MINextpay
│       │   │   └── MINextpay.apk
│       │   ├── MITSMClient
│       │   │   └── MITSMClient.apk
│       │   └── UPTsmService
│       │       ├── lib
│       │       │   └── arm64
│       │       │       ├── libcwlive.so
│       │       │       ├── libDeepNetV2.so
│       │       │       ├── libenlic.so
│       │       │       ├── libentryexpro.so
│       │       │       ├── libupbio_net.so
│       │       │       ├── libupliveness.so
│       │       │       ├── libuptsmaddonmi.so
│       │       │       ├── libuptsmservice.so
│       │       │       └── libxdjacrypto.so
│       │       └── UPTsmService.apk
│       └── data-app
│           └── MIpay
│               └── MIpay.apk
└── system.prop

11 directories, 17 files

```

这里的：
`system/product/...`

由 Magisk systemless 映射成：
`/product/...`

因此我们特意保持了国行 ROM 原始 APK 路径。

## 3.1. module.prop 文件内容
```Text
id=k80_cn_nfc
name=K80 CN NFC TSM
version=0.1
versionCode=1
author=local
description=CN MiPay module: restore Xiaomi CN MITSMClient, UPTsmService, MINextpay and MIpay on K80 EEA
```

## 3.2. system.prop 文件内容
```Text
ro.se.type=HCE,UICC,eSE
ro.vendor.se.type=eSE,HCE,UICC
```

不过后来我们发现，真正关键的是 ro.vendor.se.type，而且单靠 system.prop 并没有解决启动时序问题。
所以又在 post-fs-data.sh 提前执行 resetprop。

## 3.3. post-fs-data.sh 文件内容
```Text
#!/system/bin/sh

MODDIR=${0%/*}

resetprop ro.vendor.se.type HCE,UICC,eSE

```

这里：
`resetprop ro.vendor.se.type HCE,UICC,eSE`
是非常关键的修改。


## 3.4. customize.sh 文件内容
它主要负责给模块里的目录、APK 和 UPTsmService native libraries 设置正常权限。
```Text
set_perm_recursive "$MODPATH/system/product/app/MITSMClient" 0 0 0755 0644
set_perm_recursive "$MODPATH/system/product/app/UPTsmService" 0 0 0755 0644
set_perm_recursive "$MODPATH/system/product/app/MINextpay" 0 0 0755 0644
set_perm_recursive "$MODPATH/system/product/data-app/MIpay" 0 0 0755 0644
```

# 4. 生成 ZIP 包
进入模块根目录以后再 `zip .`，不能把外层 K80-CN-NFC 目录一起套进去。
执行：
```Bash
cd ~/k80_nfc/K80-CN-NFC
zip -r ../K80-CN-NFC-v0.1.zip .
```

生成：
~/k80_nfc/K80-CN-NFC-v0.1.zip

# 5. 手机上安装
手机已经 Magisk Root 后：
```Cmd
adb push E:\K80-CN-NFC-v0.1.zip /sdcard/Download/
```
放进手机。

手机操作：
```Android
Magisk → 模块 → 从本地安装 → 选择 ZIP(/sdcard/Download/K80-CN-NFC-v0.1.zip) → 重启。
```

重启后我们验证：
NFC 设置出现“内置安全模块”，小米钱包已经恢复正常。
