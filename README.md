# Linux-From-Scratch
Linux from scratch notes, commands, and files.

Linux Journal provides their own ["Minimal Linux Distro from Source"](https://www.linuxjournal.com/content/diy-build-custom-minimal-linux-distribution-source).
That article is linked at [DIY_MIN.md](DIY_MIN.md) 
## Required packages on host

* Bash-3.2 (/bin/sh should be a symbolic or hard link to bash)
* Binutils-2.25 (Versions greater than 2.32 are not recommended as they have not been tested)
* Bison-2.7 (/usr/bin/yacc should be a link to bison or small script that executes bison)
* Bzip2-1.0.4
* Coreutils-6.9
* Diffutils-2.8.1
* Findutils-4.2.31
* Gawk-4.0.1 (/usr/bin/awk should be a link to gawk)
* GCC-6.2 including the C++ compiler, g++ (Versions greater than 9.2.0 are not recommended as they have not been tested)
* Glibc-2.11 (Versions greater than 2.30 are not recommended as they have not been tested)
* Grep-2.5.1a
* Gzip-1.3.12
* Linux Kernel-3.2<
* M4-1.4.10
* Make-4.0
* Patch-2.5.4
* Perl-5.8.8
* Python-3.4
* Sed-4.1.5
* Tar-1.22
* Texinfo-4.7
* Xz-5.0.0

To see whether your host system has all the appropriate versions, and the ability to compile programs, run the following:
```
cat > version-check.sh << "EOF"
#!/bin/bash
# Simple script to list version numbers of critical development tools
export LC_ALL=C
bash --version | head -n1 | cut -d" " -f2-4
MYSH=$(readlink -f /bin/sh)
echo "/bin/sh -> $MYSH"
echo $MYSH | grep -q bash || echo "ERROR: /bin/sh does not point to bash"
unset MYSH

echo -n "Binutils: "; ld --version | head -n1 | cut -d" " -f3-
bison --version | head -n1

if [ -h /usr/bin/yacc ]; then
  echo "/usr/bin/yacc -> `readlink -f /usr/bin/yacc`";
elif [ -x /usr/bin/yacc ]; then
  echo yacc is `/usr/bin/yacc --version | head -n1`
else
  echo "yacc not found" 
fi

bzip2 --version 2>&1 < /dev/null | head -n1 | cut -d" " -f1,6-
echo -n "Coreutils: "; chown --version | head -n1 | cut -d")" -f2
diff --version | head -n1
find --version | head -n1
gawk --version | head -n1

if [ -h /usr/bin/awk ]; then
  echo "/usr/bin/awk -> `readlink -f /usr/bin/awk`";
elif [ -x /usr/bin/awk ]; then
  echo awk is `/usr/bin/awk --version | head -n1`
else 
  echo "awk not found" 
fi

gcc --version | head -n1
g++ --version | head -n1
ldd --version | head -n1 | cut -d" " -f2-  # glibc version
grep --version | head -n1
gzip --version | head -n1
cat /proc/version
m4 --version | head -n1
make --version | head -n1
patch --version | head -n1
echo Perl `perl -V:version`
python3 --version
sed --version | head -n1
tar --version | head -n1
makeinfo --version | head -n1  # texinfo version
xz --version | head -n1

echo 'int main(){}' > dummy.c && g++ -o dummy dummy.c
if [ -x dummy ]
  then echo "g++ compilation OK";
  else echo "g++ compilation failed"; fi
rm -f dummy.c dummy
EOF

bash version-check.sh
```

## Setup
### Partitions & File System
Partition the drive into `/boot`, `/`, and `/home`. More are possible but not necessary.

On partitions, use `ext4` filesystem.
```
mkfs -v -t ext4 /dev/<XXX>
```
If you are using a swap parition, there is no need to format, but you can initialize it by `mkswap /dev/<yyy>`.

### Environment
Set `export LFS=/mnt/lfs`

#### Mounting
Create mount(s)
```
mkdir -pv $LFS
mount -v -t ext4 /dev/<xxx> $LFS
mkdir -v $LFS/usr
mount -v -t ext4 /dev/<yyy> $LFS/usr

# Enable swap
/sbin/swapon -v /dev/<zzz>
```
### Packages and Patches
Store all source code in mount. Packages are available at [http://www.linuxfromscratch.org/patches/downloads/].
```
mkdir -v $LFS/sources

# Only file owner can delete - 'sticky'
chmod -v a+wt $LFS/sources

wget --input-file=wget-list --continue --directory-prefix=$LFS/sources

# Verify checksums.
mv md5sums $LFS/sources
pushd $LFS/sources
md5sum -c md5sums
popd
```

All required packages are listed in `src/wget-list`. All required patches are listed in `src/patches`.
 Create a symlink to `/tools/` on host system.
 ```
 mkdir -v $LFS/tools
 ln -sv $LFS/tools /
 ```
 The created symlink enables the toolchain to be compiled so that it always refers to /tools, meaning that the compiler, assembler, and linker will work both in Chapter 5 (when we are still using some tools from the host) and in the next (when we are “chrooted” to the LFS partition).

### User
Create a lfs user for isolation.
```
groupadd lfs
useradd -s /bin/bash -g lfs -m -k /dev/null lfs
passwd lfs
chown -v lfs $LFS/tools
chown -v lfs $LFS/sources

# Login as user.
su - lfs
```
#### Login Env
Logged in as `lfs`
```
# Non-Login Env.
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF

# Isolate Env.
cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX # controls localization
LFS_TGT=$(uname -m)-lfs-linux-gnu

PATH=/tools/bin:/bin:/usr/bin
export LFS LC_ALL LFS_TGT PATH
EOF

# Source Env for building.
source ~/.bash_profile
```
*through 4.5*