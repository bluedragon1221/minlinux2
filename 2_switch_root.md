# Mutable Root Filesystem
## Talk about boot process
Right now, the Minlinux boot process looks something like this:
- QEMU loads the linux kernel, also passing in our initial filesystem (initramfs) which gets fed directly to the kernel
- the kernel mounts the initramfs and executes /bin/init (busybox's init program)
- /bin/init sets up basic system stuff (hostname, setting up users, mounting other volumes, etc)
- /bin/init loads a login program (agetty) on tty1, which the user interfaces with to log in and use the shell

However, _real_ distros, such as Arch Linux, are much more complicated:
- UEFI or BIOS does some complicated stuff that I don't really understand, which loads systemd-boot
- systemd-boot loads the linux kernel, passing in the initramfs (generated with mkinitcpio)
- the kernel mounts the initramfs and executes /init (a shell script)
- /init sets up basic system stuff: mounts kernel filesystems (like /dev), load drivers
- /init mounts /dev/sda1, which contains the actual root file system for Arch, at /new_root
- /init executes `switch_root`, which replaces the root filesystem with /new_root, passing off the boot process to new_root, and executing /sbin/init (the systemd PID-1 binary)
- /sbin/init mounts remaining filesystems and starts a bunch of processes that are required for Arch linux
- /sbin/init launches agetty on tty1, giving the user an interfcae to log in and use the shell

There are a few benefits to the approach (which is why literally every distro ever does it this way):
- all files on / are mutable, making it easy to install programs and configure stuff from within the host
- smaller runtime footprint (initramfs must be loaded completely into memory, which can take up ram)
- isolation, security, buzzword, buzzword

## Set up disks
First lets set up disks support in the linux kernel.
```sh
nix-shell -p gnumake ncurses flex bison gawk bc elfutils pkg-config glibc stdenv.cc.libc.static
cd linux
make menuconfig
```

Enable these options:
- `[*] Enable the block layer --->`
  - `[ ] Legacy autoloading support`
  - `[ ] Allow writing to mounted block devices`
- `Device Drivers --->`
  - `[*] PCI Support --->`
    - `[ ] Enable PCI quirk workarounds`
  - `NVME Support --->`
    - `[*] NVM Express block device`
- `File systems --->`
  - `[*] The Extended 4 (ext4) filesystem`
  - `[ ]   Use ext4 for ext2 file systems`

While we're at it, also enable shebang scripts, since we'll need that for later:
- `Executable file formats --->`
  - `[*] Kernel support for scripts starting with #!`

Now rebuild the kernel:
```sh
make -j$(nproc)
exit
cp linux/arch/x86/boot/bzImage .
```

## initfiles
First, we'll remove inittab, since we'll be replacing busybox init with a bash script for our initramfs:
```sh
rm initfiles/etc/inittab
```

Now write this as initfiles/init:
```sh
#!/bin/sh
mount -a

mkdir /new_root
mount -t ext4 /dev/nvme0n1 /new_root

mount --move /dev /new_root/dev
mount --move /proc /new_root/proc

exec switch_root /new_root /bin/init
```
- `mount -a` mounts `/dev` and `/proc` as per etc/fstab
- then we mount our new root
- next we prepare the `/dev` and `/proc` filesystems to be `switch_root`ed
- finally, we execute `switch_root`, replacing /new_root with /, and executing /bin/init located in the new_root
- (the location is now initfiles/init instead of initfiles/bin/init because of a weird behavior of `switch_root` which requires the init executable to be at `/init`. Not really sure why it does that, but this fixes it!)

Also make the script executable:
```sh
chmod +x initfiles/init
```

## Create a minimal rootfs
At this point, all we need to do is prepare the root filesystem (that gets mounted at /new_root).

First, we'll make the volume that will be mounted, and mount it:
```sh
dd if=/dev/zero of=./root bs=1M count=1000
mkfs.ext4 ./root

mkdir r
sudo mount root ./r
```

Next we'll set up the basic filesystem (very similar to setting up the initramfs)
```sh
mkdir -p r/{bin,etc,dev,proc,root}
```

And install busybox like earlier:
```sh
cp busybox r/bin
pushd r/bin
./busybox --list | xargs -n1 -P8 ln -s busybox
cp busybox su && chmod +s su
popd
```

### `etc` files (again)
#### etc/inittab
```
tty1::respawn:/bin/login -f root
::sysinit:/bin/hostname HOSTNAME
::sysinit:/bin/mount -a
```

#### etc/fstab
```
devtmpfs /dev devtmpfs defaults 0 0
proc /proc proc defaults 0 0
```

#### etc/environment
```sh
PATH="/bin"
```

#### etc/group
```
root:x:0:
```

#### etc/passwd
```
root:x:0:0:root:/root:/bin/sh
```

#### etc/shadow
```
root:PASSWORD HASH:20005::::::
```

## Run it!
Finally, unmount the root volume, then launch the vm with qemu with extra flags for attaching the root volume:
```sh
sudo umount r/
qemu-system-x86_64 \
  -kernel bzImage -initrd init.cpio -append "rdinit=/bin/init" \
  -drive file=./root,if=none,format=raw,id=drv0 \
  -device nvme,serial=1234,drive=drv0
```

