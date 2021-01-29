# UNRAID Linux USB disk with GRUB

The default syslinux boot method does not work with my system - Gigabyte
GA-MA790FXT-UD5P (some 10+ years old now). A re-purposed desktop system.

This board is BIOS only (no UEFI)

# WARNING / DISCLAIMER

This is not a supported or even recommended method of booting Unraid. Any
deviation from the official path should be treated with appropriate caution
and healthy scepticism. This process is no exception.

I assume that, if you are reading this, there is no other option for your
system and therefore have no data to be lost.

## Partition with type c

 - FAT32 partition.
 - Partition 1 marked active.
 - Label with UNRAID.

Alternatively, the Unraid installer can do all of the above.

```
# fdisk -l /dev/sdc
Disk /dev/sdc: 28.87 GiB, 31001149440 bytes, 60549120 sectors
Disk model: DataTraveler 2.0
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device     Boot Start      End  Sectors  Size Id Type
/dev/sdc1  *     2048 60549119 60547072 28.9G  c W95 FAT32 (LBA)
```

# Mount USB stick

Mount the new partition. In my case it auto mounted at `/media/grant/UNRAID`

## Install Unraid software

This is already done if the official installer is used to create an Unraid
USB drive.

```
unzip -d /media/grant/UNRAID/ unRAIDServer-6.8.3-x86_64.zip
```

## Device Map

Important to set the device appropriately or the 2nd stage boot image
(core,.img) may not be found. Seen by a hanging `GRUB     _` line on boot.

```
mkdir -p UNRAID/boot/grub/
blkid -t LABEL=UNRAID
```

The UD5P only seems to like booting this with USB-FDD, hence the use of
`fd0`

```
echo '(fd0)    /dev/disk/by-uuid/1234-ABCD' > UNRAID/boot/grub/device.map
```

## Syslinux cfg

A handy tool to convert `syslinux.cfg` to `grub.cfg` called
`grub-syslinux2cfg`.

```
grub-syslinux2cfg UNRAID/syslinux/syslinux.cfg > UNRAID/boot/grub/grub.cfg
```

## Install grub

Installs the grub boot loader into `/dev/sdc` and the modules into
`boot/grub/i386-pc`. This installs the bare minimum of modules that I found
were required to boot UNRAID.

```
grub-install --target=i386-pc --boot-directory UNRAID/boot/ --modules="usb part_msdos" --allow-floppy --install-modules="test usb normal linux linux16" -v /dev/sdc
```

Key areas of interest from the output of that process:

```
Installing for i386-pc platform.
grub-install: info: adding `fd0' -> `/dev/sdc1' from device.map.
...
grub-install: info: grub-mkimage --directory '/usr/lib/grub/i386-pc' --prefix '(,msdos1)/boot/grub' --output '/media/grant/UNRAID/boot/grub/i386-pc/core.img'  --dtb '' --format 'i386-pc' --compression 'auto'  'usb' 'part_msdos' 'fat' 'part_msdos' 'biosdisk'
...
grub-install: info: grub-bios-setup --allow-floppy  --verbose     --directory='/media/grant/UNRAID/boot/grub/i386-pc' --device-map='/media/grant/UNRAID/boot/grub/device.map' '/dev/sdc'.
...
grub-install: info: the size of hostdisk//dev/sdc is 60549120.
grub-install: info: guessed root_dev `hostdisk//dev/sdc' from dir `/media/grant/UNRAID/boot/grub/i386-pc'.
grub-install: info: setting the root device to `hostdisk//dev/sdc,msdos1'.
```

# Testing

QEMU can be used to test quickly whether the stick is now bootable before
trying on the real system.

- First we need the USB Bus (`hostbus`) and Device (`hostaddr`) numbers:

```
$ lsusb | grep Kingston
Bus 001 Device 006: ID 0930:6544 Toshiba Corp. TransMemory-Mini / Kingston DataTraveler 2.0 Stick
```

- Translating to QEMU

```
sudo qemu-system-x86_64 -m 512 -usb -device usb-host,hostbus=1,hostaddr=6
```

If all is well, the GRUB menu will display, 5 seconds later the Unraid will
bootstrap with something like _Booting into 'Unraid OS'_.
