+++
title = "時隔一年的桌面 Guix System 安裝"
author = ["Hilton Chain"]
description = "再有下次，怕是成本不低了。"
date = 2024-02-24T01:14:00+08:00
tags = ["Guix"]
categories = ["notes"]
draft = false
image = "cover.jpg"
+++

雖說輾轉中我才[提到]({{< relref "the-4th-year#冬" >}})「那時完成的系統如今還在使用」，不過隨着媒體文件存量增長，爲 Guix System 留下的 1TiB 存儲空間也逐漸捉襟見肘了起來。

這倒不是說 Guix 本身存儲佔用有多可怕：在我此前的桌面系統中，/gnu/store 佔有 115GiB（壓縮[^fn:1]前 241GiB），約 10%，（本文中將完成的系統剛安裝好時在 7GiB 左右），而另一臺使用中的服務器，我有在維護體積，則是 3GiB。機制使然，談不上輕量，但也尚可接受。

我的筆記本電腦只有一個硬盤位，目前安裝了 2TB 的硬盤，要想替換到更大容量，價格上並不理想。這次設置系統，主要是去掉曾留給 Windows 的分區，此外分區表仍有微調，因此重新安裝難以避免。

關於 GNU Guix 的資料並不算多，中文則更少，想到上次安裝詳情並不完整也未公開發布，不免遺憾。如今我便將其寫作博文，以期爲後來者參考。

這次安裝是在 x86_64 UEFI 系統上的手動安裝，會用到 GRUB 引導加載器，也會設置 LUKS2 加密分區並在其上創建 Btrfs 文件系統。如果對手動安裝不夠熟悉，可以同時參閱 ArchWiki 上 Arch Linux 的[安裝指南](https://wiki.archlinux.org/title/Installation_guide)，在基礎上是相通的，無論路徑如何，實現相同效果就好了。


## 預備 {#預備}

Guix System 的安裝，只要求先設置好 Guix 程序。不過我有重新分區的需要，所以得在 LiveCD 環境進行，但 Guix System 的 LiveCD 在需要非自由固件時較爲費神，因此我沿用了上次的選擇——Ubuntu LiveCD，啓動方面則是藉助 [Ventoy](https://www.ventoy.net/cn/index.html)。

備份上，我正好有一塊 1TB 的閒置硬盤，所幸留給 Guix System 的是 1TiB，除去 /gnu/store 還有完整備份的餘地，再有下次，恐怕會麻煩不少了。

此外內核與固件來自 [Nonguix](https://gitlab.com/nonguix/nonguix)，在遠東並無鏡像，可能需要注意網絡代理。


## 安裝環境 {#安裝環境}

進入 LiveCD 後，首先需要進行一些基本設置，如網絡、顯示縮放、時區等，本節將提及其他值得注意處，以及 Guix 安裝設置流程。


### 登入 root {#登入-root}

安裝過程中大多命令需要較高權限，所以下文默認使用 root 用戶。

```shell
sudo --login --user=root
```


### 鍵盤佈局 {#鍵盤佈局}

我使用的鍵盤佈局是 Dvorak，之後[設置 LUKS 分區](#luks-分區-btrfs)密碼時，爲了規避可能的 GRUB 設置問題，會回到 QWERTY 佈局一次，所以兩者一併提及。

Ubuntu LiveCD 的桌面環境是 Xorg 上的 GNOME，鍵盤佈局可以用 setxkbmap 設置。

```shell
# 使用 Dvorak 佈局，並將 CapsLock 改爲 Ctrl。
setxkbmap us dvorak ctrl:nocaps

# 回到 QWERTY 佈局
setxkbmap us
```

如果是在控制檯下則可使用 loadkeys，不過不能額外設置選項。

```shell
# 使用 Dvorak 佈局
loadkeys dvorak

# 回到 QWERTY 佈局
loadkeys us
```


### 安裝 Guix {#安裝-guix}

安裝 Guix 的推薦方式是使用安裝腳本，不過如果軟件包管理器中提供了 1.4.0 版本，也應當可以使用。

```shell
cd /tmp
wget https://git.savannah.gnu.org/cgit/guix.git/plain/etc/guix-install.sh

# 通過 ftp.gnu.org 下載可能較慢，這裏將下載源改至 BFSU 鏡像站。
sed --in-place 's/ftp.gnu.org/mirrors.bfsu.edu.cn/g' guix-install.sh

chmod +x guix-install.sh
./guix-install.sh
```

安裝會向 /etc/profile.d 添加配置文件，所以需要重新[登錄](#登入-root)。

安裝好的 Guix 由 guix-daemon 和 guix 兩個程序組成，前者由 root 運行，負責基礎管理功能，後者可以由非特權用戶運行，提供其餘絕大部分功能。


### 設置 Guix {#設置-guix}


#### 頻道 {#頻道}

Guix 程序由多個頻道組成（默認僅包含官方倉庫一個頻道），更新 Guix 大致上是拉取頻道更新、編譯頻道，並將產物合成爲新的 Guix 程序。這個編譯過程最高會用到 4GiB 內存，所以要想日常使用 Guix，至少要有一臺機器內存足夠。

我要添加 Nonguix 和 [Rosenthal](https://github.com/rakino/Rosenthal) 兩個頻道，前者在[預備](#預備)中提過，包含原始 Linux 內核與非自由固件，後者是我自己的頻道，有提供支持 Argon2（LUKS2 默認使用）的 GNU GRUB 引導加載器變體。

頻道配置文件的默認路徑爲 ~/.config/guix/channels.scm 和 /etc/guix/channels.scm，前者優先級更高。

爲了指代方便，本文選擇其中之一：/etc/guix/channels.scm。

```scheme
;; /etc/guix/channels.scm 由此開始：
(list (channel
       (name 'guix)
       ;; 這裏用了 SJTUG 的鏡像，頻道中也有記錄原始地址，使用鏡像時，更新會有 warning
       (url "https://mirror.sjtu.edu.cn/git/guix.git")
       (introduction
        (make-channel-introduction
         ;; Guix 程序會從這條 commit 開始驗證 OpenPGP 簽名
         "9edb3f66fd807b096b48283debdcddccfea34bad"
         (openpgp-fingerprint
          "BBB0 2DDF 2CEA F6A8 0D1D  E643 A2A0 6DF2 A33A 54FA"))))
      (channel
       (name 'nonguix)
       (url "https://gitlab.com/nonguix/nonguix")
       (introduction
        (make-channel-introduction
         "897c1a470da759236cc11798f4e0a5f7d4d59fbc"
         (openpgp-fingerprint
          "2A39 3FFF 68F4 EF7A 3D29  12AF 6F51 20A0 22FB B2D5"))))
      (channel
       (name 'rosenthal)
       (url "https://codeberg.org/hako/rosenthal.git")
       ;; 頻道以 Git 倉庫的形式存在，需要設置分支，默認爲 "master"，所以前兩個頻道沒有設置
       (branch "trunk")
       (introduction
        (make-channel-introduction
         "7677db76330121a901604dfbad19077893865f35"
         (openpgp-fingerprint
          "13E7 6CD6 E649 C28C 3385  4DF5 5E5A A665 6149 17F7")))))
;; /etc/guix/channels.scm 在此結束。
```


#### 二進制替代 {#二進制替代}

Guix 的頻道只負責分發定義，而不包含產物，但因爲產物的輸出路徑唯一，且在構建前已知，也就有了從網絡上獲取已構建產物作爲替代的機制。

例如用我當前的 Guix 程序構建 GNU Hello，產物爲：

```text
/gnu/store/6fbh8phmp3izay6c0dpggpxhcjn4xlm5-hello-2.12.1
```

如果替代服務器上存在這個產物，Guix 就可以直接下載，反之則在本地構建。

Guix 默認替代服務器爲 <https://ci.guix.gnu.org> 和 <https://bordeaux.guix.gnu.org>，二者獨立運行。SJTUG 有提供前者鏡像。

Nonguix 也有替代服務器，不過 Guix 在傳輸產物時必須簽名與驗證，所以首先需要授權 Nonguix 的公鑰：

```shell
cd /tmp
wget https://substitutes.nonguix.org/signing-key.pub

guix archive --authorize < signing-key.pub
```

（安裝 guix 時會在 /etc/guix 下生成一對密鑰：signing-key.pub 和 signing-key.sec，已認證的公鑰則記錄在 /etc/guix/acl 中。）

然後是爲 guix-daemon 設置替代服務器。

```shell
systemctl edit --full guix-daemon.service
```

在其 systemd 配置文件中 ExecStart 部分做出以下改動，除官方服務器外，添加 SJTUG 鏡像與 Nonguix。因爲查詢二進制替代有先後順序，所以建議鏡像優先，其餘按命中率從高到低排序：

```diff
diff --git a/guix.daemon.service b/guix.daemon.service
index b0f9237..a60232e 100644
--- a/guix.daemon.service
+++ b/guix.daemon.service
@@ -7,7 +7,11 @@ Description=Build daemon for GNU Guix

 [Service]
 ExecStart=/var/guix/profiles/per-user/root/current-guix/bin/guix-daemon \
-    --build-users-group=guixbuild --discover=yes
+    --build-users-group=guixbuild --discover=yes \
+    --substitute-urls='https://mirror.sjtu.edu.cn/guix \
+                       https://ci.guix.gnu.org \
+                       https://bordeaux.guix.gnu.org \
+                       https://substitutes.nonguix.org'
 Environment='GUIX_LOCPATH=/var/guix/profiles/per-user/root/guix-profile/lib/locale' LC_ALL=en_US.utf8
 StandardOutput=syslog
 StandardError=syslog
```

如果需要爲 guix-daemon 設置代理，則修改 Environment 部分如下，增加 http_proxy 和 https_proxy 環境變量，用於構建過程中的源碼獲取及替代下載：

```diff
diff --git a/guix.daemon.service b/guix.daemon.service
index a60232e..c3a593c 100644
--- a/guix.daemon.service
+++ b/guix.daemon.service
@@ -12,7 +12,8 @@ ExecStart=/var/guix/profiles/per-user/root/current-guix/bin/guix-daemon \
                        https://ci.guix.gnu.org \
                        https://bordeaux.guix.gnu.org \
                        https://substitutes.nonguix.org'
-Environment='GUIX_LOCPATH=/var/guix/profiles/per-user/root/guix-profile/lib/locale' LC_ALL=en_US.utf8
+Environment='GUIX_LOCPATH=/var/guix/profiles/per-user/root/guix-profile/lib/locale' LC_ALL=en_US.utf8 \
+            'http_proxy=http://127.0.0.1:1080' 'https_proxy=http://127.0.0.1:1080'
 StandardOutput=syslog
 StandardError=syslog
```

隨後重啓 guix-daemon。

```shell
systemctl restart guix-daemon.service
```

作爲對比，在 Guix System 中設置二進制替代大致如下：

```scheme
(service guix-service-type
         (guix-configuration
          (authorized-keys
           (append (list (plain-file
                          "nonguix-signing-key.pub" ;Nonguix 公鑰文件內容。
                          "(public-key (ecc (curve Ed25519) (q #C1FD53E5D4CE971933EC50C9F307AE2171A2D3B52C804642A7A35F84F3A4EA98#)))"))
                   %default-authorized-guix-keys))
          ;; 代理設置
          (http-proxy "http://127.0.0.1:1080")
          (substitute-urls
           (append (list "https://mirror.sjtu.edu.cn/guix")
                   %default-substitute-urls
                   (list "https://substitutes.nonguix.org")))))
```


### 更新 Guix {#更新-guix}

下一步便是更新，更新時會先拉取頻道，這部分若需設置代理，則在當前環境設置 http_proxy 和 https_proxy，如下：

```shell
export http_proxy=http://127.0.0.1:1080
export https_proxy=$http_proxy
```

萬事具備，更新！

```shell
guix pull
```

更新後，當前用戶的 Guix 程序會被鏈接到 ~/.config/guix/current。例如對於 root 用戶， `which guix` 命令的結果應爲：

```shell
/root/.config/guix/current/bin/guix
```

如果沒有類似結果，嘗試重新[登錄](#登入-root)或執行 `hash guix` ，確保之後會運行的 Guix 程序爲 ~/.config/guix/current/bin/guix 既可。


## 文件系統 {#文件系統}

分區和文件系統在安裝好系統後再修改會比較麻煩，應當最爲注意，不過本文並不會特別涉及。


### 分區表 {#分區表}

如前述：

> 這次安裝是在 x86_64 UEFI 系統上的手動安裝，會用到 GRUB 引導加載器，也會設置 LUKS2 加密分區並在其上創建 Btrfs 文件系統。

因此我計劃在硬盤上創建兩個分區：256MiB 用作 EFI 系統分區，剩餘部分用以 LUKS 加密。

分區使用 fdisk，結果如下：

```text
Disk /dev/nvme0n1: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: Samsung SSD 970 EVO Plus 2TB
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: ED118402-2913-49AC-8F20-4A50678BE202

Device          Start        End    Sectors  Size Type
/dev/nvme0n1p1   2048     526335     524288  256M EFI System
/dev/nvme0n1p2 526336 3907028991 3906502656  1.8T Linux filesystem
```

分區過程中可能會注意到一些像是「Linux root (x86-64)」的類型，這些類型來自 [Discoverable Partitions Specification](https://uapi-group.org/specifications/specs/discoverable_partitions_specification/)，用於一些啓動時自動掛載工具，除此以外同 fdisk 默認「Linux filesystem」無異。


### EFI 系統分區（FAT32） {#efi-系統分區-fat32}

```shell
mkfs.fat -F 32 /dev/nvme0n1p1
```


### LUKS 分區（Btrfs） {#luks-分區-btrfs}

在 `cryptsetup --help` 輸出尾端可以看到各項參數預設。

```text
<...>
Default compiled-in metadata format is LUKS2 (for luksFormat action).

Default compiled-in key and passphrase parameters:
        Maximum keyfile size: 8192kB, Maximum interactive passphrase length 512 (characters)
Default PBKDF for LUKS1: pbkdf2, iteration time: 2000 (ms)
Default PBKDF for LUKS2: argon2id
        Iteration time: 2000, Memory required: 1048576kB, Parallel threads: 4

Default compiled-in device cipher parameters:
        loop-AES: aes, Key 256 bits
        plain: aes-cbc-essiv:sha256, Key: 256 bits, Password hashing: ripemd160
        LUKS: aes-xts-plain64, Key: 256 bits, LUKS header hashing: sha256, RNG: /dev/urandom
        LUKS: Default keysize with XTS mode (two internal keys) will be doubled.
```

預設對我來說已經足夠好了，不過 XTS 模式[缺乏數據驗證](https://en.wikipedia.org/wiki/Disk_encryption_theory#XTS_weaknesses)，建議配合自帶數據校驗的文件系統使用，正好我之後會用 Btrfs。

```shell
cryptsetup luksFormat --type=luks2 /dev/nvme0n1p2
```

GRUB 會在開機時解鎖 LUKS 分區，但使用的鍵盤佈局卻可能是 QWERTY，可以新增一個在 QWERTY 下按鍵相同的密碼來規避此類問題。

（由於新增密碼時需要輸入已有密碼，所以注意先輸入，再新開終端[切換佈局](#鍵盤佈局)。）

```shell
cryptsetup luksAddKey /dev/nvme0n1p2
```

解鎖 LUKS 分區時需要一個名字，解鎖後的分區會出現在 /dev/mapper/&lt;名字&gt;。

```shell
cryptsetup open /dev/nvme0n1p2 encrypted
```

將解鎖後的 LUKS 分區格式化爲 Btrfs 文件系統。

```shell
mkfs.btrfs /dev/mapper/encrypted
```

掛載文件系統並創建 Btrfs 子卷。

```shell
mkdir --parents /media/encrypted

mount --options compress=zstd,discard=async \
      /dev/mapper/encrypted /media/encrypted

btrfs subvolume create /media/encrypted/@Data
btrfs subvolume create /media/encrypted/@Home
btrfs subvolume create /media/encrypted/@Snapshot
btrfs subvolume create /media/encrypted/@System
btrfs subvolume create /media/encrypted/@System/@Guix
```

由此創建的 Btrfs 子卷佈局如下，子卷名可以是任何合法文件名， `@` 在此沒有特殊含義：

```text
/media/encrypted/
├── @Data
├── @Home
├── @Snapshot
└── @System
    └── @Guix
```

我會將 `@System/@Guix` 掛載到 /， `@Data` 掛載到 /var/lib， `@Home` 掛載到 /home，而先前設置的 EFI 系統分區則會被掛載到 /efi。

我的安裝過程將在 /mnt 下進行，這裏將文件系統掛載到對應位置：

```shell
mount --options compress=zstd,discard=async,subvol=@System/@Guix \
      /dev/mapper/encrypted /mnt

mkdir --parents /mnt{/efi,/var/lib,/home}

mount /dev/nvme0n1p1 /mnt/efi

mount --options compress=zstd,discard=async,subvol=@Data \
      /dev/mapper/encrypted /mnt/var/lib
mount --options compress=zstd,discard=async,subvol=@Home \
      /dev/mapper/encrypted /mnt/home
```

作爲對比，以上 LUKS 分區解鎖和掛載點配置，在 Guix System 中如下：

```scheme
(mapped-devices
 (list (mapped-device
        (source "/dev/nvme0n1p2")
        (target "encrypted")
        (type luks-device-mapping))))
```

（dependencies 處的 mapped-devices 就是上述 LUKS 分區解鎖配置，後面[設置 &amp; 安裝](#guix-system-設置-and-安裝)部分完整配置文件中也會提到。）

```scheme
(file-systems
 (list (file-system
         (type "btrfs")
         (mount-point "/")
         (device "/dev/mapper/encrypted")
         (options "compress=zstd,discard=async,subvol=@System/@Guix")
         (create-mount-point? #t)
         (dependencies mapped-devices))

       (file-system
         (type "fat")
         (mount-point "/efi")
         (device "/dev/nvme0n1p1")
         (create-mount-point? #t))

       (file-system
         (type "btrfs")
         (mount-point "/var/lib")
         (device "/dev/mapper/encrypted")
         (options "compress=zstd,discard=async,subvol=@Data")
         (check? #f)
         (create-mount-point? #t)
         (dependencies mapped-devices))

       (file-system
         (type "btrfs")
         (mount-point "/home")
         (device "/dev/mapper/encrypted")
         (options "compress=zstd,discard=async,subvol=@Home")
         (check? #f)
         (create-mount-point? #t)
         (dependencies mapped-devices))))
```

上述掛載點配置其實還可以減少一些重複，當然以下內容只是演示，並不會在本文涉及：

```scheme
(file-systems
 (let ((file-system-base (file-system
                           (type "btrfs")
                           (mount-point "/")
                           (device "/dev/mapper/encrypted")
                           (create-mount-point? #t)
                           (dependencies mapped-devices)))
       (options-for-subvolume
        (cut string-append "compress=zstd,discard=async,subvol=" <>)))
   (append
    (list (file-system
            (type "fat")
            (mount-point "/efi")
            (device "/dev/nvme0n1p1")
            (create-mount-point? #t)))
    (map (match-lambda
           ((subvolume . mount-point)
            (file-system
              (inherit file-system-base)
              (mount-point mount-point)
              (options (options-for-subvolume subvolume))
              (check? (string=? "/" mount-point)))))
         '(("@System/@Guix" . "/")
           ("@Data"         . "/var/lib")
           ("@Home"         . "/home"))))))
```


## Guix System 設置 &amp; 安裝 {#guix-system-設置-and-安裝}

終於來到正題了，Guix System 的操作系統設置和前面的頻道十分相像，都還算直觀。不過一些 Scheme 基礎如列表操作難以避免，因此我限制了配置文件中的 Scheme 含量，[在附錄中也有簡單解釋](#列表操作)。


### 配置文件 {#配置文件}

下面大體上是我這次安裝使用的系統配置文件，使用了 GNOME 桌面環境，對於初次設置還算方便，至少開機能夠上網，帶有基礎工具。如果未來系統設置出現問題，也能回滾到一個能工作的狀態。鍵盤佈局和代理的部分註釋掉了，可以根據情況取消註釋，在引導加載器、文件系統以及用戶設置上稍作調整就可以直接使用。

配置文件可以是任何名字，也可以保存到任意位置，爲了指代方便，本文選擇 /etc/config.scm。

```scheme
;; /etc/config.scm 由此開始：
;; Guix 頻道中的功能，是以模塊的形式提供的。
(use-modules (gnu)
             (gnu packages certs)
             (gnu packages fonts)
             (gnu services xorg)
             (gnu services desktop)
             (nongnu packages linux)
             (nongnu system linux-initrd)
             (rosenthal bootloader grub))

;; https://guix.gnu.org/manual/devel/en/guix.html#Using-the-Configuration-System
;; https://guix.gnu.org/manual/devel/en/guix.html#operating_002dsystem-Reference
(operating-system
  (host-name "dorphine")
  (timezone "Asia/Hong_Kong")
  (locale "en_US.utf8")

  ;; linux 是原始的 Linux 內核，包含使用非自由固件的驅動及非自由固件的加載功能，
  ;; linux-firmware 是非自由固件，二者在 (nongnu packages linux) 定義。
  ;; microcode-initrd 會創建一個包含 AMD 與 Intel 非自由微碼更新的 initrd，在
  ;; (nongnu system linux-initrd) 定義。
  (kernel linux)
  (firmware (list linux-firmware))
  (initrd microcode-initrd)

  ;; ;; 控制檯鍵盤佈局配置
  ;; (keyboard-layout
  ;;  ;; https://guix.gnu.org/manual/devel/en/guix.html#Keyboard-Layout
  ;;  (keyboard-layout "us" "dvorak" #:options (list "ctrl:nocaps")))

  ;; grub-efi-luks2-bootloader 是一個支持 Argon2 的 GRUB 引導加載器變體，在
  ;; (rosenthal bootloader grub) 定義。
  (bootloader
   ;; https://guix.gnu.org/manual/devel/en/guix.html#Bootloader-Configuration
   (bootloader-configuration
    (bootloader grub-efi-luks2-bootloader)
    ;; ;; 引導加載器鍵盤佈局配置
    ;; ;; 這裏的第一個 keyboard-layout 是 bootloader-configuration 配置
    ;; ;; 的一部分，第二個則是 bootloader 配置之前出現的同名配置。
    ;; (keyboard-layout keyboard-layout)
    (targets (list "/efi"))))

  (mapped-devices
   ;; https://guix.gnu.org/manual/devel/en/guix.html#Mapped-Devices
   (list (mapped-device
          (source "/dev/nvme0n1p2")
          (target "encrypted")
          (type luks-device-mapping))))

  (file-systems
   ;; https://guix.gnu.org/manual/devel/en/guix.html#File-Systems
   (append (list (file-system
                   (type "fat")
                   (mount-point "/efi")
                   (device "/dev/nvme0n1p1"))
                 (file-system
                   (type "btrfs")
                   (mount-point "/")
                   (device "/dev/mapper/encrypted")
                   (options "compress=zstd,discard=async,subvol=@System/@Guix")
                   ;; 這裏的 mapped-devices 是 file-systems 配置之前出現的同名配置。
                   (dependencies mapped-devices))
                 (file-system
                   (type "btrfs")
                   (mount-point "/var/lib")
                   (device "/dev/mapper/encrypted")
                   (options "compress=zstd,discard=async,subvol=@Data")
                   (check? #f)
                   (dependencies mapped-devices))
                 (file-system
                   (type "btrfs")
                   (mount-point "/home")
                   (device "/dev/mapper/encrypted")
                   (options "compress=zstd,discard=async,subvol=@Home")
                   (check? #f)
                   (dependencies mapped-devices)))
           ;; %base-file-systems 包含一些用戶通常不會主動配置的文件系統，需要注
           ;; 意的是 % 其實並沒有任何特殊含義。
           ;; 操作系統的 file-systems 配置只需要一個列表，所以上面另外創建了一個
           ;; 列表，再用 append 把兩個列表合爲一個。
           %base-file-systems))

  (users
   ;; https://guix.gnu.org/manual/devel/en/guix.html#User-Accounts
   (append (list (user-account
                  (name "myuser")
                  (group "users")
                  (supplementary-groups (list "audio" "video" "wheel"))))
           %base-user-accounts))

  ;; font-google-noto 是一套支持所有語言的字體。
  ;; nss-certs 是 CA 證書，上網需要。
  (packages
   (append (list font-google-noto nss-certs)
           %base-packages))

  (services
   (append
    ;; https://guix.gnu.org/manual/devel/en/guix.html#Desktop-Services
    ;; https://guix.gnu.org/manual/devel/en/guix.html#X-Window
    (list (service gnome-desktop-service-type))
    (modify-services %desktop-services
      ;; modify-services 接受一個服務列表，其結果也是一個服務列表。
      ;; 將 %desktop-services 中 gdm-service-type 種類服務的原有配置綁定到
      ;; config（這個名字可以隨便起），「=>」 後面是 gdm-service-type 的新配置。
      (gdm-service-type
       config => (gdm-configuration
                  ;; gdm-service-type 的配置就是一個 gdm-configuration，
                  ;; 同種結構可以繼承。
                  (inherit config)
                  ;; (xorg-configuration
                  ;;  ;; https://guix.gnu.org/manual/devel/en/guix.html#index-Xorg_002c-configuration
                  ;;  (xorg-configuration
                  ;;   ;; Xorg 鍵盤佈局配置
                  ;;   (keyboard-layout keyboard-layout)))
                  (wayland? #t)))
      ;; https://guix.gnu.org/manual/devel/en/guix.html#index-guix_002dservice_002dtype
      (guix-service-type
       config => (guix-configuration
                  (inherit config)
                  (authorized-keys
                   ;; https://guix.gnu.org/manual/devel/en/guix.html#G_002dExpressions
                   (append (list (plain-file
                                  "nonguix-signing-key.pub" ;Nonguix 公鑰文件內容。
                                  "(public-key (ecc (curve Ed25519) (q #C1FD53E5D4CE971933EC50C9F307AE2171A2D3B52C804642A7A35F84F3A4EA98#)))"))
                           %default-authorized-guix-keys))
                  ;; ;; 代理設置
                  ;; (http-proxy "http://127.0.0.1:1080")
                  (substitute-urls
                   (append (list "https://mirror.sjtu.edu.cn/guix")
                           %default-substitute-urls
                           (list "https://substitutes.nonguix.org")))))))))
;; /etc/config.scm 在此結束。
```


### 安裝 Guix System {#安裝-guix-system}

指定配置文件和安裝路徑就可以了。

```shell
guix system init /etc/config.scm /mnt
```

Guix System 在安裝上，會先構建引導加載器配置[^fn:2]，而產物存放在 /gnu/store 下，對於 LiveCD 環境，文件系統存儲在內存，可能會內存不足。

Guix System LiveCD 的解決方案是 [cow-store](https://guix.gnu.org/manual/devel/en/guix.html#Proceeding-with-the-Installation) 服務：掛載外部文件系統到 /gnu/store，這樣對其寫入也就不會影響內存了。本文附錄附有[手動實現 cow-store 流程](#cow-store)。

安裝過程可能因爲網絡問題失敗，不過已經下載好的內容之後不會重複下載，所以失敗了也請放心，重試就好。

爲了方便在新系統中的使用，可以把 [Guix System](#配置文件) 和[頻道](#頻道)的配置文件一併放進安裝路徑：

```shell
mkdir --parents /mnt/etc/guix
# /etc/guix 會存儲私鑰，所以有權限要求
chmod 0511 /mnt/etc/guix

cp {,/mnt}/etc/config.scm
cp {,/mnt}/etc/guix/channels.scm
```

至此安裝流程也就結束，可以重啓了。


## 安裝之後 {#安裝之後}

啓動後會需要輸入兩次密碼，至於原因參見附錄[啓動流程](#啓動流程)。


### 設置密碼 {#設置密碼}

完成啓動後後會進入 GDM 登錄介面，不過因爲還沒有設置密碼，此時登錄介面中並無用戶可選。

Ctrl+Alt+F1 進入控制檯，以 root 登錄，可以直接登入。

爲用戶設置密碼：

```shell
passwd myuser
```

登入用戶，驗證 sudo 正常後工作再登出用戶：

```shell
su --login myuser
sudo echo
logout
```

鎖定 root 賬戶，再登出 root。

```shell
password --lock root
logout
```

Ctrl+Alt+F7 回到登錄介面，現在就有用戶了，輸入密碼進入桌面。


### 接下來？ {#接下來}

先前安裝時已經設置了頻道配置文件，所以可以接收更新了。

```shell
guix pull
```

應用系統配置的命令如下，只需要一個配置文件路徑，對其路徑和名稱沒有要求：

```shell
sudo guix system reconfigure /etc/config.scm
```

Guix 的 sudo 會保留 PATH 環境變量，也就是說 `sudo guix` 會正確使用當前用戶的 Guix，當然初次使用還是需要確認 guix 命令指向 ~/.config/guix/current/bin/guix。

此外建議將系統配置文件存放到版本控制系統。附錄中也包含了 [GNU Shepherd 使用說明](#gnu-shepherd-使用說明)。

參考手冊中包含的內容可能比想象中還要多，可以以 [Getting Started](https://guix.gnu.org/manual/devel/en/guix.html#Getting-Started) 這一節爲入口開始。

最後的最後，附圖一張。Happy hacking！

![Guix System 上的 GNOME 桌面環境](gnome-on-guix.png)


## 參考 {#參考}

-   [Booting process of Linux - Wikipedia](https://en.wikipedia.org/wiki/Booting_process_of_Linux)
-   [Disk encryption theory - Wikipedia](https://en.wikipedia.org/wiki/Disk_encryption_theory)
-   [Frequently Asked Questions Cryptsetup/LUKS - cryptsetup Wiki](https://gitlab.com/cryptsetup/cryptsetup/-/wikis/FrequentlyAskedQuestions)
-   [GNU Guix Reference Manual](https://guix.gnu.org/manual/devel/en/guix.html)
-   [You Don't Want XTS — Quarrelsome](https://sockpuppet.org/blog/2014/04/30/you-dont-want-xts/)
-   [dm-crypt/Device encryption - ArchWiki](https://wiki.archlinux.org/title/Dm-crypt/Device_encryption)
-   [Using the initial RAM disk (initrd) - The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/admin-guide/initrd.html)


## 附錄 {#附錄}


### 列表操作 {#列表操作}

這裏提供一些列表操作的例子，我在配置文件中只使用了 list 和 append，不過 GNU Guix 參考手冊中也有用到 cons，雖說 Guix 手冊中代碼部分都有超鏈接到 GNU Guile 參考手冊，但初見可能不太直觀，所以我也一併做個並不準確的解釋：

```scheme
;; list 從任意個元素創建一個列表
(list)                                  ; ()
(list 1)                                ; (1)
(list 1 2)                              ; (1 2)
(list 1 2 3)                            ; (1 2 3)

;; append 將任意個列表追加爲一個
(append)                                ; ()
(append (list 1))                       ; (1)
(append (list 1) (list 2))              ; (1 2)
(append (list 1) (list 2) (list 3))     ; (1 2 3)

;; cons 將一個元素放到一個列表頭部
(cons 0 (list      ))                   ;       (0)
(cons 1 (list     0))                   ;     (1 0)
(cons 2 (list   1 0))                   ;   (2 1 0)
(cons 3 (list 2 1 0))                   ; (3 2 1 0)

;; cons* 將任意個元素放到一個列表頭部
(cons*       (list 0))                  ;       (0)
(cons*     1 (list 0))                  ;     (1 0)
(cons*   2 1 (list 0))                  ;   (2 1 0)
(cons* 3 2 1 (list 0))                  ; (3 2 1 0)

;; 假設要構造 (bash coreutils findutils grep) 這樣一個列表，以下爲幾種可能：
(list bash coreutils findutils grep)

(append (list bash) (list coreutils findutils) (list grep))

(cons bash (list coreutils findutils grep))

(cons* bash coreutils findutils (list grep))
```


### cow-store {#cow-store}

以下爲 cow-store 手動實現：

```shell
# 先前在 /mnt 路徑掛載了外部文件系統，所以就在這個路徑操作。
target=/mnt

tmpdir=$target/tmp
rw_dir=$tmpdir/guix-inst
work_dir=$rw_dir/../.overlayfs-workdir

mkdir --parents $tmpdir
mkdir --parents $rw_dir
mkdir --parents $work_dir

# Guix 的構建發生在 /tmp，構建時可能會有較多佔用，所以將外部文件系統上的目錄掛載過去。
mount --bind $tmpdir /tmp

# rw_dir 會被用作 /gnu/store，而 /gnu/store 有特殊權限要求。
chown 0:30000 $rw_dir
chmod 1775 $rw_dir

# 創建一個 OverlayFS，包含 /gnu/store 和 rw_dir 的內容，寫入這個文件系統會寫進 rw_dir。
# 掛載到 /gnu/store。
mount --types=overlay \
      --options=lowerdir=/gnu/store,upperdir=$rw_dir,workdir=$work_dir \
      none /gnu/store
```

手動實現 cow-store 後若要抵消操作：

```shell
# 卸載先前從外部文件系統掛載的 /tmp
umount /tmp

# 卸載先前掛載的 OverlayFS
umount /gnu/store
# 刪除先前向 OverlayFS 寫入的文件
rm --recursive $rw_dir

# /gnu/store 的內容由數據庫索引，gc --verify 會驗證 /gnu/store，從而清理對不存在內容的索引。
guix gc --verify
```


### 啓動流程 {#啓動流程}

UEFI 系統中使用 GRUB 作爲引導加載器時，GNU/Linux 啓動流程大致如下：

```text
UEFI -> GRUB（核心鏡像 -> 配置文件 + 模塊）-> Linux + initrd -> PID 1
```

UEFI 標準支持使用 FAT 文件系統的 EFI 系統分區，所以 GRUB 核心鏡像要被安裝到這樣一個文件系統。

GRUB 採用模塊化設計，在安裝時會需要指定啓動目錄（默認爲 /boot），用以安裝配置文件和模塊。
同時，提供啓動目錄所在文件系統支持的模塊也會被放進核心鏡像中，這是爲了保證 GRUB 核心鏡像能夠讀取到自己的配置文件。

在我的系統中，GRUB 的啓動目錄在 LUKS 分區（LUKS2 格式）上的 Btrfs 文件系統，所以 GRUB 核心鏡像中需要同時有 LUKS2 和 Btrfs 支持。而讀取配置文件前需要先解密其所在分區，這就是開機時第一次密碼輸入。

GRUB 的配置文件包含啓動 Linux 內核的條件：Linux 內核與 initrd 路徑，以及啓動參數。自然，GRUB 必須支持內核和 initrd 所在的文件系統，對於 Guix System 來說，就是 /gnu/store 所在的文件系統。

Linux 內核也是採用模塊化設計，initrd 裏放了啓動過程中需要的模塊，內核啓動後會解壓 initrd 並運行其中的 init 程序，這個 init 程序負責掛載 `/` 和其他在配置中標記爲啓動時需要的文件系統，創建根文件系統中的剩餘部分，並運行 PID 1，在 Guix System 中也就是 GNU Shepherd，自此結束啓動流程。

initrd 中的 init 程序負責掛載 `/` ，他也是在 LUKS 分區上，需要先解密，這也就是開機時第二次密碼輸入。

在 Guix System 的啓動流程中，需要注意的問題主要和 GRUB 有關：

1.  GRUB 需要支持 /boot 和 /gnu/store 所在的文件系統。
2.  GRUB 目前不支持 Argon2，所以沒有完整的 LUKS2 支持。
3.  Guix 並沒有干預 GRUB 核心鏡像的生成，最後安裝的核心鏡像會使用 QWERTY 鍵盤佈局。

對於第一點，不需要太多考慮，第二點可以由[前述](#頻道)支持 Argon2 的 GRUB 變體解決。

至於第三點，日常在 GRUB 中輸入的機會不多，主要可能是在解密 LUKS 分區時輸入密碼，所以可以爲 LUKS 分區設置兩個密碼：一個用需要的鍵盤佈局，另一個用 QWERTY，兩者使用相同按鍵。當然最好是讓 Guix 干預 GRUB 核心鏡像生成，從根本上解決問題，但這是之後的事了。


### GNU Shepherd 使用說明 {#gnu-shepherd-使用說明}

Shepherd 包含四個程序：

-   shepherd：運行服務，監聽 socket。
-   herd：連接 socket，控制 shepherd。
-   halt：連接 socket，關機。
-   reboot：連接 socket，重啓。

Shepherd 在認證上依賴文件系統的權限管理能力。比如 Guix System 的 Shepherd，socket 在 /var/run/shepherd/socket，socket 的權限是 0755，其所在目錄則爲 0700。

如果能連接到 socket 就能控制 Shepherd，所以 halt、reboot、用 herd 連接系統 Shepherd 都需要 sudo。

herd 的語法爲： `herd ACTION [SERVICE [OPTIONS...]]`

`herd status` 顯示指定 Shepherd 服務狀態信息，省略服務時則顯示自身信息，Shepherd 自身也叫 root 服務，所以 `herd status root` 會輸出相同結果，如下（有省略）：

```text
Started:
 + bluetooth
 + file-systems
 + guix-daemon
 + root
 + root-file-system
One-shot:
 * host-name
 * user-homes
```

常規服務狀態信息格式不同，如 `herd status bluetooth` ：

```text
Status of bluetooth:
  It is running since 03:01:10 PM (8 hours ago).
  Running value is 1341.
  It is enabled.
  Provides (bluetooth).
  Requires (dbus-system udev).
  Will be respawned.
```

`herd log` 或 `herd log root` 顯示服務的狀態變化記錄：

```text
23 Feb 2024 15:01:17    service root is being started
23 Feb 2024 15:01:17    service root is running
23 Feb 2024 15:01:17    service pipewire is being started
23 Feb 2024 15:01:17    service pipewire is running
23 Feb 2024 15:01:17    service wireplumber is being started
23 Feb 2024 15:01:17    service wireplumber is running
23 Feb 2024 15:01:17    service mcron is being started
23 Feb 2024 15:01:17    service mcron is running
23 Feb 2024 15:01:17    service gpg-agent is being started
23 Feb 2024 15:01:17    service gpg-agent is running
23 Feb 2024 15:01:17    service dbus is being started
23 Feb 2024 15:01:17    service dbus is running
```

其餘基礎操作爲 `herd start <服務>` 、 `herd stop <服務>` 、 `herd restart <服務>` 、 `herd enable <服務>` 和 `herd disable <服務>` ，分別爲啓動、停止、重啓、啓用、禁用服務。重啓服務的邏輯是停止並啓動服務，所以重啓 root 服務是不可能的，下爲 `herd restart root` 輸出：

```text
You must be kidding.
```

`herd doc` 顯示服務描述，例如 `herd doc root` 結果如下：

```text
The root service is used to operate on shepherd itself.
```

`herd doc <服務> list-actions` 則可列出指定服務的自定義操作，如 `herd doc root list-actions` ：

```text
root (help status halt power-off load eval unload reload daemonize restart)
```

[^fn:1]: Btrfs，zstd 壓縮，壓縮等級爲預設（即 3），非強制壓縮。
[^fn:2]: 引導加載器配置包含（依賴）Linux 內核、initrd 及啓動參數，啓動參數又依賴 init 程序。正好是操作系統存在的充分條件。
