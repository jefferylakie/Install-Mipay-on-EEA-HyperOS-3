# 1. Prepare the ROMs

ROM download page:
`https://hyperos.fans/zh/devices/zorn`

K80 EEA fastboot ROM:
`zorn_eea_global_images_OS3.0.302.0.WOKEUXM_20260715.0000.00_16.0_eea_666bcd4cc8.tgz`

K80 China fastboot ROM:
`zorn_images_OS3.0.306.0.WOKCNXM_20260708.0000.00_16.0_cn_4173919e32.tgz`

Flash the EEA firmware onto the phone, and extract all files from the China firmware.

After extracting the China firmware on Windows, copy:
`images\super.img`

to Ubuntu, for example:
`~/k80_nfc/super.img`

Install the sparse image utilities and prepare the partitions:
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

Use the device returned by the last command in the next command. In my case it was:
`/dev/loop12`

Therefore:
```
cd ~/k80_nfc/dynpart-src
sudo ./make-dynpart-mappings /dev/loop12 0
```

Then mount the required China ROM partition, for example:
```
sudo mkdir -p /mnt/k80_product
sudo mount -o ro /dev/mapper/product_a /mnt/k80_product
```

# 2. Extract the four components from the China ROM
## 2.1 Confirm that all four components exist
```Bash
sudo find /mnt/k80_product -type f \( \
-name 'MITSMClient.apk' -o \
-name 'UPTsmService.apk' -o \
-name 'MINextpay.apk' -o \
-name 'MIpay.apk' \
\)
```

Expected paths:
```Bash
/mnt/k80_product/app/MITSMClient/MITSMClient.apk
/mnt/k80_product/app/UPTsmService/UPTsmService.apk
/mnt/k80_product/app/MINextpay/MINextpay.apk
/mnt/k80_product/data-app/MIpay/MIpay.apk
```

Also check the native libraries for UPTsmService:
```Bash
sudo find /mnt/k80_product/app/UPTsmService -type f
```

In addition to the APK, you should see nine `.so` files under `lib/arm64/`.

## 2.2 Create the Magisk module directories
```Bash
mkdir -p ~/k80_nfc/K80-CN-NFC/system/product/app/MITSMClient
mkdir -p ~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService
mkdir -p ~/k80_nfc/K80-CN-NFC/system/product/app/MINextpay
mkdir -p ~/k80_nfc/K80-CN-NFC/system/product/data-app/MIpay
```

## 2.3 Copy MITSMClient
```Bash
cp -a \
/mnt/k80_product/app/MITSMClient/MITSMClient.apk \
~/k80_nfc/K80-CN-NFC/system/product/app/MITSMClient/
```

## 2.4 Copy UPTsmService, including its nine `.so` files
Copy the entire directory contents:
```Bash
cp -a \
/mnt/k80_product/app/UPTsmService/. \
~/k80_nfc/K80-CN-NFC/system/product/app/UPTsmService/
```

This copies both `UPTsmService.apk` and the `lib/arm64/` directory containing the `.so` files, rather than just the APK.

## 2.5 Copy MINextpay
```Bash
cp -a \
/mnt/k80_product/app/MINextpay/MINextpay.apk \
~/k80_nfc/K80-CN-NFC/system/product/app/MINextpay/
```

## 2.6 Copy MIpay
```Bash
cp -a \
/mnt/k80_product/data-app/MIpay/MIpay.apk \
~/k80_nfc/K80-CN-NFC/system/product/data-app/MIpay/
```

## 2.7 Check the extracted files
Run:
```Bash
find ~/k80_nfc/K80-CN-NFC/system/product -type f | sort
```

You should see:
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

# 3. Assemble the Magisk module
The finished working directory is:
`~/k80_nfc/K80-CN-NFC/`

Its structure is:
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

Magisk maps `system/product/...` to `/product/...` systemlessly. This preserves the original APK paths from the China ROM.

## 3.1 Contents of `module.prop`
```Text
id=k80_cn_nfc
name=K80 CN NFC TSM
version=0.1
versionCode=1
author=local
description=CN MiPay module: restore Xiaomi CN MITSMClient, UPTsmService, MINextpay and MIpay on K80 EEA
```

## 3.2 Contents of `system.prop`
```Text
ro.se.type=HCE,UICC,eSE
ro.vendor.se.type=eSE,HCE,UICC
```

We later found that `ro.vendor.se.type` was the key property, and `system.prop` alone did not resolve the startup timing issue. Therefore, `post-fs-data.sh` also runs `resetprop` early in the boot process.

## 3.3 Contents of `post-fs-data.sh`
```Text
#!/system/bin/sh

MODDIR=${0%/*}

resetprop ro.vendor.se.type HCE,UICC,eSE

```

The `resetprop ro.vendor.se.type HCE,UICC,eSE` change is essential.

## 3.4 Contents of `customize.sh`
This sets standard permissions on the module directories, APKs, and UPTsmService native libraries.
```Text
set_perm_recursive "$MODPATH/system/product/app/MITSMClient" 0 0 0755 0644
set_perm_recursive "$MODPATH/system/product/app/UPTsmService" 0 0 0755 0644
set_perm_recursive "$MODPATH/system/product/app/MINextpay" 0 0 0755 0644
set_perm_recursive "$MODPATH/system/product/data-app/MIpay" 0 0 0755 0644
```

# 4. Create the ZIP archive
Enter the module root directory before running `zip`. Do not include the outer `K80-CN-NFC` directory in the archive.

Run:
```Bash
cd ~/k80_nfc/K80-CN-NFC
zip -r ../K80-CN-NFC-v0.7.zip .
```

Output:
`~/k80_nfc/K80-CN-NFC-v0.7.zip`

# 5. Install on the phone
After rooting the phone with Magisk, copy the ZIP to the phone:
```Cmd
adb push ~/k80_nfc/K80-CN-NFC-v0.7.zip /sdcard/Download/
```

On the phone:
```Android
Magisk → Modules → Install from storage → Select ZIP (/sdcard/Download/K80-CN-NFC-v0.1.zip) → Reboot
```

After rebooting, we confirmed that an “Embedded Secure Element” option appeared in the NFC settings and Xiaomi Wallet worked again.

# 6. Prebuilt module
The project includes `K80-CN-NFC-v0.7.zip`, a prebuilt module that can be installed directly through Magisk.
