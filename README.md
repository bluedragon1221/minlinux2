# Minimal Linux Guide 2.0

Since writing my first guide on creating minimal linux, I've learned a lot and fixed several errors. Here's what's different:
- Includes proper instructions for creating a user and automatically logging in as them
- Uses a precompiled busybox (You can still compile it yourself if you want to)
- Uses busybox's init instead of writing our own janky one
- Has more config settings for the kernel, hopefully ironing out some bugs from the last guide

Here we go!

---

First we'll set up a directory to store the files that will go in our initramfs:
```sh
mkdir -p initfiles/{bin,etc,dev,proc,root}
```

## Busybox
Busybox will serve as our userspace. Let's download it:
```sh
wget https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox
chmod +x busybox
```

We'll copy busybox to our initramfs and install it there:
```sh
cp busybox initfiles/bin
pushd initfiles/bin
./busybox --list | xargs -n1 -P8 ln -s busybox
cp busybox su && chmod +s su
popd
```
- Busybox is a multi-call binary. This means that when you symlink many programs to it, it will figure out which one is being called and run that one. Our command simply loops through every busybox program, and creates a symlink to busybox for it. See [`man 1 busybox`](https://man.archlinux.org/man/busybox.1.en) for more information.
- `su` needs special permissions because it elevates permissions from a user to root, which most programs aren't allowed to do

## `etc` files
Set up configuration files for initfiles/etc that the system needs to boot and launch a root terminal.

### etc/inittab
```
tty1::respawn:/bin/login -f root
::sysinit:/bin/hostname HOSTNAME
::sysinit:/bin/mount -a
```
- `login -f root` will launch a terminal logged in as the root user
- `hostname HOSTNAME` sets the hostname
- `mount -a` mounts everything in /etc/fstab

### etc/fstab
```
devtmpfs /dev devtmpfs defaults 0 0
proc /proc proc defaults 0 0
```
- This tells the system to mount the devices filesystem at `/dev` so device files are available, and the processes filesystem at `/proc` so that running processes information is available

### etc/environment
```sh
PATH="/bin"
```
- This sets the PATH so the system can find our busybox programs.
- You can put binaries in other paths, and separate them with colons here (ex. `PATH=/bin:/usr/bin`)

### etc/group
```
root:x:0:
```
Each line defines a group in the format `name:password:GID:members`.
The tty group allows terminal access.

### etc/passwd
```
root:x:0:0:root:/root:/bin/sh
```
Format: `username:password:UID:GID:description:home:shell`.
The x means passwords are in /etc/shadow.

### etc/shadow
Define user passwords:
```
root:PASSWORD HASH:20005::::::
```
- Format: `username:hash:last_changed:min:max:warn:inactive:expire:reserved`
- (Don't worry about replacing PASSWORD HASH, we'll fill it in later)

## Compiling the kernel
Clone the kernel source code:
```sh
git clone --depth 1 --branch "v6.19" https://github.com/torvalds/linux
```
- `--depth 1` makes it so we don't download the entire history with the clone. Since we're just compiling it, we don't need the history.

Let's install some dependencies that we'll need to build the kernel. Here's a nix-shell command for everything:
```sh
nix-shell -p gnumake ncurses flex bison gawk bc elfutils pkg-config glibc stdenv.cc.libc.static
cd linux
```
- (For other distros, the package names should be about the same, idk)

Now, lets configure the kernel.
Since we're going for a very mininal configuration, let's start with `tinyconfig`, the smallest possible working kernel, and enable stuff to get a working result.
```sh
make tinyconfig
make menuconfig
```

Here's the options to enable:
- `[*] 64-bit kernel`
- `General Setup --->`
  - `[*] Initial RAM filesystem and RAM disk (initramfs/initrd) support`, but disable the compression types (`gzip`, `bzip2`, `LZMA`, `XZ`, `LZO`, `LZ4`, `ZSTD`), as we're making an uncompressed initrd
  - `[*] Configure standard kernel features (expert users) --->`
    - `[*] Multiple users, groups and capabilities support`
    - `[*] Enable support for printk`
- `Device Drivers --->`
  - `Generic Driver Options --->`
    - `[*] Maintain a devtmpfs filesystem to mount at /dev`
  - `Character Devices --->`
    - `[*] Enable TTY`
    - `[*]   Virtual terminal`
    - `[ ]     Enable character translations in console`
    - `[*]     Support for console on virtual terminal`
    - `[ ]   Unix98 PTY support`
    - `[ ]   Legacy (BSD) PTY support`
- `Executable File Formats --->`
  - `[*] Kernel support for ELF binaries`
- `File systems --->`
  - `Pseudo filesystems --->`
    - `[*] /proc file system support`
    - `[ ] Sysctl support (/proc/sys)`
    - `[ ] Enable /proc page monitoring`

Now build the kernel and copy the resulting file:
```sh
make -j$(nproc)
exit
cp linux/arch/x86/boot/bzImage .
```

## Create Initrd
Package the filesystem into an initrd archive:
```sh
pushd initfiles
find . | cpio -o -H newc --owner=+0:+0 > ../init.cpio
popd
```
- `newc` is the format that the kernel expects initrd file to be in. See [the kernel docs](https://docs.kernel.org/admin-guide/initrd.html#compressed-cpio-images) for more information
- `--owner=+0:+0` makes it so the files are owned by UID 0 (root), and not our current user. Alternatively, we could `chown -R root initfiles/`, but then we would need `sudo` to edit them. This is a cleaner way.

## Run it!
Boot the system with QEMU:
```sh
qemu-system-x86_64 -kernel bzImage -initrd init.cpio
```
- (For NixOS, use `nix-shell -p qemu` to use qemu without installing it)

## Set up agetty
This will give you a log in prompt when you boot instead of dropping straight into the shell
- in the VM, run `mkpasswd`. This will prompt you to type in a password, and it will give you a hash (might look something like this: "w2f8YcyDLrC9Y")
- Open initfiles/etc/shadow, and update "PASSWORD HASH" with that hash
- replace the line in initfiles/etc/inittab that starts with `tty1::` with `tty1::respawn:/bin/getty 38400 tty1`

---

# Extensions
- [`2_switch_root`](./2_switch_root.md): Configures `switch_root` for a mutable rootfs
- [`3_add_user`](./3_add_user.md): Adds a user

## Contributing
Is there something else that you did with your `minlinux` that you want to share?
Write a guide for it! Open a PR! I'll be happy to accept it, after I ensure that it works.

---

# Sources
- https://www.youtube.com/watch?v=QlzoegSuIzg
- https://github.com/damianoognissanti/bbl
- https://blinry.org/tiny-linux
- https://gist.github.com/bluedragon1221/a58b0e1ed4492b44aa530f4db0ffef85
