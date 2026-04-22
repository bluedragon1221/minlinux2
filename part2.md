# Persistent Home Directory
In this extension to the guide, we'll make the user's home directory persistent by mounting it like a partition.
Previously, all changes were discarded on reboot

Enable kernel support for block devices and filesystems:
- `[*] Enable the block layer --->`
  - `[ ] Legacy autoloading support`
  - `[ ] Allow writing to mounted block devices`
- `Device Drivers --->`
  - `[*] PCI Support --->`
  - `SCSI device Support --->`
    - `[*] SCSI device support`
    - `[*] SCSI disk support`
  - `[*] Serial ATA and Parallel ATA drivers (libata) --->`
    - `[ ] Verbose ATA error reporting`
    - `[ ] "libata.force=" kernel parameter support`
    - `[*] ATA SFF support (for legacy IDE and PATA)`
    - `[*] ATA BMDMA support`
    - `[*] Intel ESB, ICH, PIIX3, PIIX4, PATA/SATA support`
- `File systems --->`
  - `[*] The Extended 4 (ext4) filesystem`
    - `[ ] Use ext4 for ext2 file systems`
Then rebuild the kernel.

Create a 1GB virtual disk image for persistent storage:
```
dd if=/dev/zero of=./home bs=1M count=1000
mkfs.ext4 ./home
```

Have busybox mount the disk on system startup by adding this line to initfiles/etc/fstab:
```
/dev/sda /home/USERNAME ext4 rw,relatime 0 1
```

Also remove any files that you had previously in initfiles/home/USERNAME since these will be unaccessable by the mount anyway.

Boot with the virtual disk attached:
```
qemu-system-x86_64 -kernel bzImage -initrd init.cpio -append "rdinit=/bin/init" -drive file=./home,format=raw
```

Note that without the `-append "rdinit=/bin/init"` flag now, the vm will fail to boot because it will attempt to look in the new filesystem for the init script.
(not really sure why it does that, but this fixes it!)
