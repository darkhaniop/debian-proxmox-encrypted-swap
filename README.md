# Set Up Encrypted Swap on Debian 12 / Proxmox VE 8

## Description

This guide explains how to set up an encrypted swap backed by a swap-file.
The steps are based on the [Arch-wiki: dm-crypt/Swap encryption](https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption) guide.
I verified the method on Debian 11, 12, and Proxmox VE 8, but it should work on other Linux systems.

## Motivation

Sometimes virtualization host systems (currently my host environments include Proxmox VE and vanilla Debian), run nested containerization and virtualization environments (e.g., VMs that host Kubernetes and similar clusters in dev environments).
Some of these envionments may require the host OS to run without swap enabled due to security considerations.

However, when such "host OS" itself is nested and if the parent host system uses a swap partition (without the knowledge of the nested OS), this could lead to potential security vunerabilities.
Therefore, configuring the virtualization host system with encrypted swap is more secure than using an unencrypted swap partition or file.

## Caveats of Using Encrypted Swap

In addition to the CPU overhead needed for encryption, there are additional caveats of using encrypted swap partitions on machines with the  suspend-to-disk mode (a.k.a. "hybernation") support such as laptops and desktop PCs.
In short, to ensure the desired security benefits of encrypted swap, the suspend-to-disk mode must be diabled.
For more details please see [the Arch Wiki page](https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption).

## Solution

### 1. Disable existing swap

If other persistent-storage-backed swap is enabled on the system, make sure to disable those and remove them from the system configuration.

### 2. Create a backing swap file

Create a backing swap file of the desired size with `dd` or `truncate` (truncate is quicker because it does not write the bytes to the disk).

**File size must be divisible by the encryption block size in the following step.** In this guide we use 4096-byte sector size. For example, if we want 2G swap, instead of 2'000'000'000 bytes, we should create a file of size 2 GiB or 2'147'483'648 (=2*1024^3) bytes.

With `dd`:
```bash
dd bs=1M count=2048 if=/dev/zero of=/swap0.encrypted
```

With `truncate`:
```bash
truncate -s 2147483648 /swap0.encrypted
```

Example sizes:

In GiB | In 1M blocks | In bytes
--- | --- | ---
2 | 2048 | 2147483648
4 | 4096 | 4294967296
8 | 8192 | 8589934592
16 | 16384 | 17179869184
24 | 24576 | 25769803776
32 | 32768 | 34359738368
48 | 49152 | 51539607552
64 | 65536 | 68719476736
72 | 73728 | 77309411328
96 | 98304 | 103079215104
128 | 131072 | 137438953472

### 3. Configure `crypttab`

Similar to how `/etc/fstab` is used to define filesystems, `/etc/crypttab` is used to define encrypted partitions to be opened at system boot.

Put the following configuration in `/etc/crypttab` (create a new file if it does not already exist)
```
# <name>  <device>          <password>    <options>
swap0     /swap0.encrypted  /dev/urandom  swap,cipher=aes-xts-plain64,size=256,sector-size=4096
```

It instructs `dm-crypt` to create a mapping `/dev/mapper/swap0` for `/swap0.encrypted` encrypted with a random password without storing the encryption headers in the file itself.

**Warning: When using a partition instead of a file, DO NOT use the partition path in the form of `/dev/sdb2`! Device numbering can change between boots (and when additional devices, e.g. USB are added or removed) and `dm-crypt` could overwrite a different disk partition.** Instead, we can use the partition identifier, which is (most likely) unique and it does not change when the partition is reformatted (i.e., when it is re-encrypted with a new password after every reboot):
```
# <name>  <device>                                       <password>    <options>
swap0     PARTUUID=6889ce20-0eea-41a9-afed-65e25277cc3a  /dev/urandom  swap,cipher=aes-xts-plain64,size=256,sector-size=4096
```

### 4. Configure `fstab`

Finally, we can configure the swap space in `/etc/fstab` by appending:
```
/dev/mapper/swap0  none  swap  defaults  0  0
```

### 5a. Optional: Apply Changes without rebooting

Works on Debian 12 and Proxmox 8, may work on other `systemd`-managed systems, but I have not tested it.

```bash
/usr/lib/systemd/system-generators/systemd-cryptsetup-generator /run/systemd/generator
systemctl daemon-reload
systemctl restart cryptsetup.target
```

This tells `systemd` to generate unit cryptsetup unit files based on the current `/etc/crypttab`, reload the configuration, and execute them.


### 5b. Optional: Reboot to Apply Changes

The configuration changes will be applied after reboot.

### 6. Verify

The changes can be verified with `lsblk` and `swapon`.
The following example shows a Proxmox VE 8 node with 16 GiB encrypted swap configured:
```
# lsblk
NAME                 MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
loop0                  7:0    0    16G  0 loop  
└─swap0              252:6    0    16G  0 crypt [SWAP]
...
# swapon
NAME      TYPE      SIZE  USED PRIO
/dev/dm-0 partition  16G    0B   -2
#
```
