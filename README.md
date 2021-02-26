# Cryptsetup + GPG Protected Keys

This guide was created in case Sakaki's guide ever dissapears. Ultimately she has abandoned maintaining it and the scope and purpose is much different.

This guide is intended to help a user setup an encrypted drive in Linux using Cryptsetup with password protected keys leveraging GPG.

This guide is intended for drives you might use for your rootfs and perhaps other partitions your system depends on. 

Therefore we use a LVM-on-top-of-LUKs setup. This allows us to encrypt a large physical partition and divy it up later.


## 1 - Partition the Drive

First we partition our drive using parted

`parted -a optimal /dev/sdc`

Using parted is beyond the scope of this guide, but going forward we will use `/dev/sdc1` as an example for our physical partition.

## 2 - Create the GPG Password Protected Key

To allow gpg to work properly in our terminal we must set the following environment variable

`export GPG_TTY=$(tty)`

Now we use `dd` to generate a key we'll use for encryption and password protect it with `gpg`

`dd if=/dev/urandom bs=8388607 count=1 | gpg --symmetric --cipher-algo AES256 --output /tmp/efiboot/luks-key.gpg`

# Use key to encrypt drive partition
#gpg --decrypt /tmp/efiboot/luks-key.gpg | cryptsetup --cipher serpent-xts-plain64 --key-size 512 --hash whirlpool --key-file - luksFormat /dev/sdZn 

# Show cryptsetup headers
#cryptsetup luksDump /dev/sdc1

# 3 - Backup luksHeaders & alternate keys
#cryptsetup luksHeaderBackup /dev/sdc1 --header-backup-file stor0-luks-header.img

# Adding a fallback password
#mkfifo /tmp/gpgpipe
#echo RELOADAGENT | gpg-connect-agent
#gpg --decrypt /tmp/efiboot/luks-key.gpg | cat - >/tmp/gpgpipe
#cryptsetup --key-file /tmp/gpgpipe luksAddKey /dev/sdZn
#rm -vf /tmp/gpgpipe

# 4 - Create LVM Groups + Volumes
# Open Luks Partition
#gpg --decrypt /tmp/efiboot/luks-key.gpg | cryptsetup --key-file - luksOpen /dev/sdZn gentoo

# Create physical volume
#pvcreate /dev/mapper/gentoo

# Create volume group
#vgcreate vg1 /dev/mapper/gentoo 

# Create Logical Volumes
# swap
#lvcreate --size 10G --name swap vg1
# root
#lvcreate --size 50G --name root vg1
# home
#lvcreate --extents 95%FREE --name home vg1

# 5 - Create Filesystems & Mount
#mkswap -L "swap" /dev/mapper/vg1-swap
#swapon /dev/mapper/vg1-swap
#mkfs.btrfs -L "root" /dev/mapper/vg1-root

