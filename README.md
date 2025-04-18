# Set Up Encrypted Swap on Debian 12 / Proxmox VE 8

## Description

This guide explains how to set up an encrypted swap backed by a swap-file.
The steps are based on the [Arch Wiki: dm-crypt/Swap encryption](https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption) guide.
I verified the method on Debian 11, 12, and Proxmox VE 8, but it should work on other Linux systems.

### Step 1

create 8GiB swap file
```bash
dd bs=1M count=8192 if=/dev/zero of=/swap8.encrypted
```

### Step 2a (file)

create crypttab
```
# <name>  <device>       <password>    <options>
swap8  /swap8.encrypted  /dev/urandom  swap,cipher=aes-xts-plain64,size=256,sector-size=4096
```

### Step 2b (partition)

Make sure to not use `/dev/sdb2` or similar (such identifier can change on reboot).

create crypttab
```
# <name>  <device>                                       <password>    <options>
swap8     PARTUUID=6889ce20-0eea-41a9-afed-65e25277cc3a  /dev/urandom  swap,cipher=aes-xts-plain64,size=256,sector-size=4096
```

### Step 3

update fstab
```
/dev/mapper/swap8  none  swap  defaults  0  0
```