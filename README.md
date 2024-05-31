# Reproducer for mkosi's XBOOTLDR partition issue

Suppose [mkosi](https://github.com/systemd/mkosi) should create a disk image with systemd-boot, UKIs and three
partitions:
* [EFI system partition for `/efi`](mkosi.repart/00-esp.conf)
* [XBOOTLDR partition for `/boot`](mkosi.repart/10-xbootldr.conf)
* [root partition for `/`](mkosi.repart/20-root.conf)

Using above's [mkosi.repart](mkosi.repart) configuration, `mkosi build` will create a non-bootable system:
`systemd-boot` will not be able to find the UKI at the ESP.

The disk will have three partitions:

```
$> losetup --find --show -P mkosi.output/image.raw

$> sgdisk  --print /dev/loop0
Disk /dev/loop0: 20973608 sectors, 10.0 GiB
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): 94F1B98E-61F9-428E-A180-421874475669
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 2048, last usable sector is 20973574
Partitions will be aligned on 2048-sector boundaries
Total free space is 7 sectors (3.5 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         2099199   1024.0 MiB  EF00  esp
   2         2099200         4196351   1024.0 MiB  EA00  xbootldr
   3         4196352        20973567   8.0 GiB     8304  root-x86-64
```

For Debian Sid, the ESP contains the following:
```
./EFI
./EFI/systemd
./EFI/systemd/systemd-bootx64.efi
./EFI/BOOT
./EFI/BOOT/BOOTX64.EFI
./EFI/BOOT/grubx64.EFI
./EFI/BOOT/mmx64.efi
./loader
./loader/loader.conf
./loader/random-seed
```

XBOOTLDR contains the following:
```
./System.map-6.8.11-amd64
./config-6.8.11-amd64
./vmlinuz-6.8.11-amd64
./loader
./loader/entries
./loader/entries.srel
./EFI
./EFI/Linux
./EFI/Linux/debian-6.8.11-amd64.efi
```

The last file, `./EFI/Linux/debian-6.8.11-amd64.efi` is supposed to be stored at ESP.

## Workarounds

One dirty workaround is to copy all files from `/boot` to `/efi`, e.g. by adding `CopyFiles=/boot:/` to
[mkosi.repart/00-esp.conf](mkosi.repart/00-esp.conf) as suggested [here](
https://github.com/systemd/mkosi/commit/efbca7d01307a319f270ab65c30871ac31b43e4a).

However, a cleaner solution would be to install the UKI to `/efi` instead of `/boot`:
```
diff --git a/mkosi/__init__.py b/mkosi/__init__.py
index 5b49a82f..b41d36f1 100644
--- a/mkosi/__init__.py
+++ b/mkosi/__init__.py
@@ -2314,9 +2314,9 @@ def install_uki(context: Context, kver: str, kimg: Path, token: str, partitions:
     else:
         if roothash:
             _, _, h = roothash.partition("=")
-            boot_binary = context.root / f"boot/EFI/Linux/{token}-{kver}-{h}{boot_count}.efi"
+            boot_binary = context.root / f"efi/EFI/Linux/{token}-{kver}-{h}{boot_count}.efi"
         else:
-            boot_binary = context.root / f"boot/EFI/Linux/{token}-{kver}{boot_count}.efi"
+            boot_binary = context.root / f"efi/EFI/Linux/{token}-{kver}{boot_count}.efi"
 
     # Make sure the parent directory where we'll be writing the UKI exists.
     with umask(~0o700):
```

**NOTE:** Will this patch affect other build formats?!?

After applying the patch and rebuilding the image for Debian Sid, the ESP contains the following:
```
./EFI
./EFI/systemd
./EFI/systemd/systemd-bootx64.efi
./EFI/BOOT
./EFI/BOOT/BOOTX64.EFI
./EFI/BOOT/grubx64.EFI
./EFI/BOOT/mmx64.efi
./EFI/Linux
./EFI/Linux/debian-6.8.11-amd64.efi
./loader
./loader/loader.conf
./loader/random-seed
```

XBOOTLDR will contain the following:
```
./System.map-6.8.11-amd64
./config-6.8.11-amd64
./vmlinuz-6.8.11-amd64
./loader
./loader/entries
./loader/entries.srel
./EFI
./EFI/Linux
```

Afterwards, the image can be booted with `mkosi qemu`.
