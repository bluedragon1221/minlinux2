# Add a User
In this extension to the guide, we'll set up a user directly from within the running virtual machine using busybox's `adduser` command.

(Replace all occurences of `USERNAME` with the name of your user)

Since we're now using `switch_root` with a mutable root filesystem, we can make user management much easier by adding users directly from within the running system, rather than editing configuration files on the host.

## Boot the VM
Boot the virtual machine:
```sh
qemu-system-x86_64 \
  -kernel bzImage -initrd init.cpio -append "rdinit=/bin/init" \
  -drive file=./root,if=none,format=raw,id=drv0 \
  -device nvme,serial=1234,drive=drv0
```

## Add the user
Once you're logged in as root, use `adduser` to create the new user:
```sh
adduser -h /home/USERNAME -s /bin/sh USERNAME
```

This will:
- Create a home directory at `/home/USERNAME`
- Set the shell to `/bin/sh`
- Create entries in `/etc/passwd` and `/etc/group`
- Prompt you for a password

When prompted, enter your desired password.

## Shell profile
You can customize the shell environment for your user by creating `/home/USERNAME/.profile`:
```sh
PS1='[\[\e[32m\]\u@\h \W\[\e[0m\]]\$ '

alias ls="ls --color=auto"
alias ll="ls -l"
```
Note: This is POSIX shell, not bash. They're similar, but many bash features won't work here.

## Update inittab
Update `/etc/inittab` to log in as your new user instead of root. Change the line:
```
tty1::respawn:/bin/login -f root
```
to:
```
tty1::respawn:/bin/login -f USERNAME
```

## Set up agetty
By default, the system will automatically log in as `USERNAME`. If you want a login prompt instead, you can enable `agetty`:

Change the line in `/etc/inittab` from:
```
tty1::respawn:/bin/login -f USERNAME
```
to:
```
tty1::respawn:/bin/getty 38400 tty1
```
This will present a login prompt instead of automatically logging in. You can then log in with `USERNAME` and the password you set with `adduser`.

## Test it out
Reboot the VM to test. When you log in, you should be able to log in as `USERNAME` instead of root.
