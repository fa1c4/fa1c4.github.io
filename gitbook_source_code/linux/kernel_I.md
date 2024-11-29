# Linux Kernel

### download

```shell
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

or download `linux-x.xx.tar.gz` from [kernel.org](https://www.kernel.org)



### compile

dependencies

```shell
sudo apt install libncurses5-dev libssl-dev
sudo apt install build-essential openssl
sudo apt install pkg-config
sudo apt install libc6-dev
sudo apt install bison
sudo apt install flex
sudo apt install libelf-dev
sudo apt install zlibc minizip
sudo apt install libidn11-dev libidn11
sudo apt install zstd
```

disable specific config parameters

```shell
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS
scripts/config --set-str CONFIG_SYSTEM_TRUSTED_KEYS ""
scripts/config --set-str CONFIG_SYSTEM_REVOCATION_KEYS ""
```

make

```shell
git tag # see the tags
git checkout tag-xx # switch to specific version 
make menuconfig # config the compiling parameters for kernel
make olddefconfig
make -j$(nproc) # $(($(nproc) - 1))

# after making
cd arch/x86/boot
file bzImage
# bzImage: Linux kernel x86 boot executable bzImage, version 5.19.0 (fa1c4@fa1c4-virtual-machine) #5 SMP PREEMPT_DYNAMIC Fri Nov 29 16:52:40 CST 2024, RO-rootFS, swap_dev 0x9, Normal VGA
```

errors & solutions

```shell
# certs/x509_certificate_list
make[1]: *** No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list'.  Stop.
make: *** [Makefile:1868: certs] Error 2
# execute command 
scripts/config --disable SYSTEM_TRUSTED_KEYS

# make[1]: *** No rule to make target 'debian/canonical-revoked-certs.pem', needed by 'certs/x509_revocation_list'.  Stop.
make: *** [Makefile:1868: certs] Error 2
# execute command 
scripts/config --disable SYSTEM_REVOCATION_KEYS

# zstd error
/bin/sh: 1: zstd: not found
make[2]: *** [arch/x86/boot/compressed/Makefile:141: arch/x86/boot/compressed/vmlinux.bin.zst] Error 127
make[2]: *** Deleting file 'arch/x86/boot/compressed/vmlinux.bin.zst'
make[2]: *** Waiting for unfinished jobs....
make[1]: *** [arch/x86/boot/Makefile:115: arch/x86/boot/compressed/vmlinux] Error 2
make: *** [arch/x86/Makefile:272: bzImage] Error 2
# execute command
sudo apt-get install zstd
```





## Busybox

download from [busybox.net](https://www.busybox.net/downloads)

```shell
tar -xf busybox-xx.tar.bz2
# cd into it
make menuconfig
# turn on the --build option-- -> build static binary (no shared libs)
make -j$(nproc) # $(($(nproc) - 1))
make install
_install/bin/busybox --help
# BusyBox v1.35.0 (2024-11-29 09:48:15 CST) multi-call binary.
# ...
# Usage: busybox [function [arguments]...]
#   or: busybox --list[-full]
```



