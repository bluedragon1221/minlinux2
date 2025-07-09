# Persistent Home Directory
In this extension to the guide, we'll make the user's home directory persistent by mounting it like a partition.
Previously, all changes were discarded on reboot

Enable kernel support for block devices and filesystems:
- `[*] Enable the block layer --->`
  - `[ ] Legacy autoloading support`
- `Device Drivers --->`
  - `[*] PCI Support --->`
  - `SCSI device Support --->`
    - `[*] SCSI device support`
    - `[*] SCSI disk support`
  - `[*] Serial ATA and Parallel ATA drivers (libata) --->`
    - (keep `ATA SFF support` and `ATA MBDMA support` enabled)
    - `[*] Intel ESB, ICH, PIIX3, PIIX4, PATA/SATA support`
- `File systems --->`
  - `[*] The Extended 4 (ext4) filesystem`
Then rebuild the kernel.

Create a 1GB virtual disk image for persistent storage:
```
dd if=/dev/zero of=./home bs=1M count=1000
mkfs.ext4 ./home
```

Configure the system to mount the disk at boot by adding this line to /etc/fstab:
```
/dev/sda     /home/USERNAME    ext4        rw,relatime           0 1
```

Boot with the virtual disk attached:
```
qemu-system-x86_64 -kernel bzImage -initrd init.cpio -append "rdinit=/bin/init" -drive file=./home,format=raw
```
