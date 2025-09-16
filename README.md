# Minimal Linux Guide 2.0

Since writing my first guide on creating minimal linux, I've learned a lot and fixed several errors. Here's what's different:
- Includes proper instructions for creating a user and automatically logging in as them
- Uses a precompiled busybox (You can still compile it yourself if you want to)
- Uses busybox's init instead of writing our own janky one
- Has more config settings for the kernel, hopefully ironing out some bugs from the last guide

Here we go!

## Setup files
Create the basic directory structure for our minimal system:
```sh
mkdir -p initfiles/{bin,etc,dev,root,home/USERNAME}
```

## Busybox
Busybox will serve as our userspace.
```sh
# download the binary
wget https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox
chmod +x busybox

# move it to the correct location
mv busybox initfiles/bin/busybox

# install busybox
cd initfiles/bin
./busybox --list | xargs -n1 -P8 ln -s busybox
chmod +s su
```
- Busybox is a multi-call binary. This means that when you symlink many programs to it, it will figure out which one is being called and run that one. Our command simply loops through every busybox program, and creates a symlink to busybox for it. See [`man 1 busybox`](https://man.archlinux.org/man/busybox.1.en#USAGE) for more information.
- `su` needs special permissions because its job is to elevate permissions from a user to root, which most programs aren't allowed to do.

## `etc` files
Set up configuration files that the system needs to boot and handle users.

### /etc/inittab
```
tty1::respawn:/bin/login -f USERNAME
::sysinit:/bin/hostname -F /etc/hostname
::sysinit:mount -a
::sysinit:/bin/chown c /home/c
```
This tells busybox's init to set the hostname and automatically log in as USERNAME on the first terminal.
After setting your user's password, you can change the first line to `tty1::respawn:/bin/getty 38400 tty1` to require the user to log in.
`mount -a` mounts everything in /etc/fstab
`chown c /home/c` makes sure that the user can create, edit, and delete files in their home directory

### /etc/fstab
```
devtmpfs /dev devtmpfs mode=0755,nosuid 0 0
```
This tells the system to mount a device filesystem at `/dev` so device files are available.

### /etc/hostname
```
HOSTNAME
```
Contains the system's hostname that gets set at boot.

### /etc/environment
```sh
PATH="/bin"
```
This sets the PATH so the system can find our busybox programs.
Later, you can put binaries in other paths, and separate them with colons here (ex. `PATH=/bin:/home/c/bin`)

### /etc/group
```
root:x:0:
tty:x:5:USERNAME
USERNAME:x:1030:
```
Each line defines a group in the format `name:password:GID:members`.
The tty group allows terminal access.

### /etc/passwd
```
root:x:0:0:root:/root:/bin/sh
USERNAME:x:1030:1030::/home/USERNAME:/bin/sh
```
Format: `username:password:UID:GID:description:home:shell`.
The x means passwords are in /etc/shadow.

### /etc/shadow
Define user passwords:
```
USERNAME:PASSWORD HASH:20005::::::
root:PASSWORD HASH:20005::::::
```
Format: `username:hash:last_changed:min:max:warn:inactive:expire:reserved`

When writing this file, leave the password hash blank.
Once you boot the vm for the first time, use the `mkpasswd` command (part of busybox) to generate your password hash.
Then go back and update it in initfiles/etc/shadow.

### /home/USERNAME/.profile
You can put any shell customization here:
```
PS1='[\[\e[32m\]\u@\h \W\[\e[0m\]]\$ '

alias ls="ls --color=auto"
alias ll="ls -l"
```
Beware this is not bash! It's POSIX shell, which bash is based on. They're similar, but many features are missing from bash.

## Compiling the kernel
Clone the kernel source code:
```sh
git clone --depth 1 --branch "v6.15" https://github.com/torvalds/linux
cd linux
```
`--depth 1` makes it so we don't download the entire history with the clone. Since we're just compiling it, we don't need the history.

Let's install some dependencies that we'll need to build the kernel. Here's a nix-shell command for everything:
```sh
nix-shell -p gnumake ncurses flex bison gawk bc elfutils pkg-config glibc stdenv.cc.libc.static
```
For other distros, the package names should be about the same

Now, lets configure the kernel.
Since we're going for a very mininal configuration, lets start with `tinyconfig`, the smallest possible working kernel, and enable stuff to get a working result.
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
    - `[ ] Unix98 PTY support`
    - `[ ] Legacy (BSD) PTY support`
- `Executable File Formats --->`
  - `[*] Kernel support for ELF binaries`

Now build it, and copy the resulting file:
```sh
make -j$(nproc)
cp arch/x86/boot/bzImage ..
```

## Create Initrd
Package the filesystem into an initrd archive:
```sh
cd initfiles
find . | cpio -o -H newc --owner=+0:+0 > ../init.cpio
```
- `newc` is the format that the kernel expects initrd file to be in. See [the kernel docs](https://docs.kernel.org/admin-guide/initrd.html#compressed-cpio-images) for more information
- `--owner=+0:+0` makes it so the files are owned by UID 0 (root), and not our current user. Alternatively, we could `chown -R root initfiles/`, but then we would need `sudo` to edit them. This is a cleaner way.

## Run it!
Boot the system with QEMU:
```sh
qemu-system-x86_64 -kernel bzImage -initrd init.cpio
```

---

# Extensions
- [`part2`](./part2.md): Set up a persistent home directory
- [`part3`](./part3.md): Set up a layman's package manager and install cowsay

## Contributing
Is there something else that you did with your `minlinux` that you want to share?
Write a guide for it! Open a PR! I'll be happy to accept it, after I ensure that it works.

---

# Sources
- https://www.youtube.com/watch?v=QlzoegSuIzg
- https://github.com/damianoognissanti/bbl
- https://blinry.org/tiny-linux
- https://gist.github.com/bluedragon1221/a58b0e1ed4492b44aa530f4db0ffef85
