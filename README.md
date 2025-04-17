# debian-proxmox-encrypted-swap

### Step 1

create 8GiB swap file
```
dd bs=1M count=8192 if=/dev/zero of=/swap8.encrypted
```

### Step 2

create crypttab
```
# <name>  <device>       <password>    <options>
swap8  /swap8.encrypted  /dev/urandom  swap,cipher=aes-xts-plain64,size=256,sector-size=4096
```

### Step 3

update fstab
```
/dev/mapper/swap8  none  swap  defaults  0  0
```