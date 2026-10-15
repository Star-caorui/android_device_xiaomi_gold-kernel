# android_device_xiaomi_gold-kernel

Prebuilt stock kernel for the Redmi Note 13 5G (`gold`): the kernel image, the device trees and the kernel modules, and a newer GKI build from Google under `gki/`. It is checked out at `device/xiaomi/gold-kernel`.

| | |
| --- | --- |
| Device tree | [android_device_xiaomi_gold](https://github.com/Star-caorui/android_device_xiaomi_gold) |
| How to build | [local_manifests](https://github.com/Star-caorui/local_manifests) |

## Source

Every file comes from Xiaomi's full OTA for the global model:

```
gold_global-ota_full-OS3.0.5.0.VNQMIXM-user-15.0-df9ff3aa93.zip
SHA-256 35c9f1d98b28538ac10c319ad4cba993cd5a632960d9111162e3efa494dae5c9
```

| Here | In the package |
| --- | --- |
| `kernel` | the kernel image in the `boot` partition |
| `dtb/gold.dtb` | the device tree blob in the `vendor_boot` partition |
| `dtbo.img` | the `dtbo` partition, without the padding and the AVB footer |
| `vendor_ramdisk/*.ko` | `lib/modules/` in the vendor ramdisk of `vendor_boot` |
| `modules.load.vendor_ramdisk` | `lib/modules/modules.load` in the same ramdisk |
| `modules.load.recovery` | `lib/modules/modules.load.recovery` in the same ramdisk |
| `vendor_dlkm/*.ko` | `lib/modules/` in the `vendor_dlkm` partition |
| `modules.load.vendor` | `lib/modules/modules.load` in the `vendor_dlkm` partition |
| `system_dlkm/lib/modules/` | the same path in the `system_dlkm` partition |
| `kernel_aarch64_Module.symvers` | not in the package: the symbol versions of the same GKI build, from Google's build artifacts |

- The kernel image is Google's GKI build `6.6.118-android15-8-ge56cf6b09cca-ab15511674-4k` (`android15-6.6-2026-01_r34`), byte for byte the `Image.gz` Google publishes for it. Its source is the Android common kernel at commit [e56cf6b09cca](https://android.googlesource.com/kernel/common/+/e56cf6b09cca). The GKI modules in `system_dlkm` come from the same build.
- The other modules are Xiaomi's and MediaTek's, as shipped. Xiaomi has published kernel source for this phone only for its Android 13 release (5.10).

## History

| Commit | What it does |
| --- | --- |
| 1 | Imports the files exactly as the package has them |
| 2 | Adds `wmt_drv.ko` to `modules.load.vendor`. The Wi-Fi, Bluetooth, GPS and FM drivers depend on it |
| 3 | Changes one byte in `hq_charger_sysfs.ko`, to stop a thread that only serves MIUI's charging animation |
| 4 | Adds `kernel_aarch64_Module.symvers` of the stock GKI build, for building vendor modules against it |
| 5 | Adds Google's GKI build 16498190 (`android15-6.6-2026-10_r1`) in `gki/16498190/` |

Each commit message gives the exact command.

## Verifying

### The first commit

You need `unzip`, `cpio`, `lz4`, `fsck.erofs` (erofs-utils) and a LineageOS tree for `ota_extractor`, `unpack_bootimg.py` and `avbtool.py`. Set `T` to the top of that tree and run this from the top of a clone of this repository:

```sh
git checkout $(git rev-list --max-parents=0 HEAD)

mkdir -p work/img work/ramdisk
unzip -o gold_global-ota_full-OS3.0.5.0.VNQMIXM-user-15.0-df9ff3aa93.zip payload.bin -d work
$T/prebuilts/extract-tools/linux-x86/bin/ota_extractor --payload work/payload.bin \
  --output_dir work/img --partitions boot,vendor_boot,dtbo,vendor_dlkm,system_dlkm
python3 $T/system/tools/mkbootimg/unpack_bootimg.py --boot_img work/img/boot.img \
  --out work/boot >/dev/null
python3 $T/system/tools/mkbootimg/unpack_bootimg.py --boot_img work/img/vendor_boot.img \
  --out work/vendor_boot >/dev/null
for f in work/vendor_boot/vendor_ramdisk0*; do
    lz4 -dc "$f" | (cd work/ramdisk && cpio -idmu --quiet)
done
fsck.erofs --extract=work/vendor_dlkm work/img/vendor_dlkm.img >/dev/null
fsck.erofs --extract=work/system_dlkm work/img/system_dlkm.img >/dev/null
python3 $T/external/avb/avbtool.py erase_footer --image work/img/dtbo.img

cmp kernel work/boot/kernel
cmp dtb/gold.dtb work/vendor_boot/dtb
cmp dtbo.img work/img/dtbo.img
cmp modules.load.vendor_ramdisk work/ramdisk/lib/modules/modules.load
cmp modules.load.recovery work/ramdisk/lib/modules/modules.load.recovery
cmp modules.load.vendor work/vendor_dlkm/lib/modules/modules.load
check() { (cd "$1" && find . -type f) | while read -r f; do
    cmp -s "$1/$f" "$2/$f" || echo "DIFFERENT: $1/$f"; done; }
check vendor_ramdisk work/ramdisk/lib/modules
check vendor_dlkm work/vendor_dlkm/lib/modules
check system_dlkm/lib/modules work/system_dlkm/lib/modules
```

After the progress lines of `ota_extractor`, nothing more is printed when every file matches.

### The whole branch

`extract-kernel.sh` in the device tree makes this directory from the package, the two changes included. From the top of a LineageOS tree that has both checked out:

```sh
device/xiaomi/gold/extract-kernel.sh gold_global-ota_full-OS3.0.5.0.VNQMIXM-user-15.0-df9ff3aa93.zip /tmp/gold-kernel
diff -r --exclude=.git --exclude=README.md --exclude=kernel_aarch64_Module.symvers --exclude=gki /tmp/gold-kernel device/xiaomi/gold-kernel
```

`diff` prints nothing when they are the same.

## GKI builds

The kernel image and the `system_dlkm` modules are Google's GKI, not Xiaomi's: any later GKI build of the same KMI generation (`android15-8`) runs the same vendor modules. The device tree picks one with `GOLD_GKI`:

| `GOLD_GKI` | Files |
| --- | --- |
| `stock` | `kernel`, `system_dlkm/` and `kernel_aarch64_Module.symvers` at the top: build 15511674, as stock ships it |
| `latest` | the same three in `gki/16498190/`: build 16498190, `6.6.143-android15-8-g170954b20ad9-ab16498190-4k` (`android15-6.6-2026-10_r1`, [release builds](https://source.android.com/docs/core/architecture/kernel/gki-android15-6_6-release-builds)) |

`kernel_aarch64_Module.symvers` is what vendor modules built from source are linked against, so that they carry the CRCs this kernel checks.

`gki/<build>/` is made by `fetch-gki.sh` in the device tree from Google's build artifacts: `Image.gz` as `kernel`, the symbol versions, and from `system_dlkm_staging_archive.tar.gz` the modules stock's `system_dlkm/lib/modules/modules.load` names, with that list and Google's `modules.dep` and `modules.alias` cut down to them. Build 16498190 no longer has `tls.ko` (`CONFIG_TLS` is gone from `gki_defconfig`); nothing depends on it. To check it, from the top of a LineageOS tree with both repositories checked out:

```sh
device/xiaomi/gold/fetch-gki.sh 16498190 /tmp/gold-kernel-check
```

with a copy of this repository at `/tmp/gold-kernel-check`, then `git -C /tmp/gold-kernel-check status` shows no change. The stock build's files can be checked the same way against its artifacts:

```sh
B=https://ci.android.com/builds/submitted/15511674/kernel_aarch64/latest/raw
curl -sL $B/Image.gz | cmp - kernel
curl -sL $B/kernel_aarch64_Module.symvers | cmp - kernel_aarch64_Module.symvers
```

## Regenerating

From the top of a LineageOS tree with the device tree checked out:

```sh
device/xiaomi/gold/extract-kernel.sh /path/to/gold_global-ota_full-OS3.0.5.0.VNQMIXM-user-15.0-df9ff3aa93.zip
```

- It needs `unzip`, `cpio`, `lz4` and `fsck.erofs` (erofs-utils) on the host.
- It checks the package's checksum and stops on any other package. Set `ANY_OTA=1` to try one anyway.
- It deletes `device/xiaomi/gold-kernel` before writing it. Use it in a tree that does not have this repository checked out, or give it another output directory as a second argument.

## Credits

- [Dhterech](https://github.com/Dhterech/android_kernel_xiaomi_gold): the layout of this directory is that of his android_kernel_xiaomi_gold, and loading `wmt_drv.ko` from `modules.load.vendor` is his change. Our device tree is a fork of his.
- The files themselves were taken from Xiaomi's package again, not from that repository.
