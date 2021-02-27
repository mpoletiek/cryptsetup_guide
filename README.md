# Cryptsetup + GPG Protected Keys

This guide was created in case Sakaki's guide ever dissapears. Ultimately she has abandoned maintaining it and the scope and purpose is much different.

This guide is intended to help a user setup an encrypted drive in Linux using Cryptsetup with password protected key-files leveraging GPG.

This guide is intended for drives you might use for your rootfs and perhaps other partitions your system depends on. 

Therefore we use a LVM-on-top-of-LUKS setup. This allows us to encrypt a large physical partition and divy it up later.


## 1 - Partition the Drive

First we partition our drive using parted

`parted -a optimal /dev/sdc`

Using parted is beyond the scope of this guide, but going forward we will use `/dev/sdc1` as an example for our physical partition.

## 2 - Create the GPG Password Protected Key

To allow gpg to work properly in our terminal we must set the following environment variable

`export GPG_TTY=$(tty)`

Now we use `dd` to generate a key-file we'll use for encryption and password protect it with `gpg`.

Store this key somewhere safe.

`dd if=/dev/urandom bs=8388607 count=1 | gpg --symmetric --cipher-algo AES256 --output /tmp/luks-key.gpg`

## 3 - Format the physical partition with cryptsetup

Using the gpg protected key-file we'll format our physical partition

`gpg --decrypt /tmp/luks-key.gpg | cryptsetup --key-size 512 --key-file - luksFormat /dev/sdc1`

We can verify the success of this operation by checking the devices luks headers. 

`cryptsetup luksDump /dev/sdc1`

## 4 - Backup luksHeaders & alternate keys

As an additional precaution, we will backup our luks Headers.

Again, store this file someplace safe.

`cyptsetup luksHeaderBackup /dev/sdc1 --header-backup-file /tmp/luks-header.img`

### Adding a fallback password

Just in case all else fails, we can setup a fallback password to decrypt our physical partition.

To prevent secrets from being written to the disk we'll use a named pipe.
```
mkfifo /tmp/gpgpipe
echo RELOADAGENT | gpg-connect-agent
gpg --decrypt /tmp/luks-key.gpg | cat - >/tmp/gpgpipe
cryptsetup --key-file /tmp/gpgpipe luksAddKey /dev/sdZn
rm -vf /tmp/gpgpipe
```
## 5 - Create LVM Groups + Volumes

Now we're ready to open the LUKS partition and build our LVM tables on top of it.

The `--allow-discards` option is to protect precious SSDs.

`gpg --decrypt /tmp/luks-key.gpg | cryptsetup --key-file - luksOpen --allow-discards /dev/sdc1 drive`

Now create the physical volume
`pvcreate /dev/mapper/drive`

Create volume group
`vgcreate vg1 /dev/mapper/drive`

Create Logical Volumes
swap
`lvcreate --size 10G --name swap vg1`
root
`lvcreate --size 50G --name root vg1`
home
`lvcreate --extents 95%FREE --name home vg1`

# 6 - Create Filesystems & Mount
```
mkswap -L "swap" /dev/mapper/vg1-swap
swapon /dev/mapper/vg1-swap
mkfs.btrfs -L "root" /dev/mapper/vg1-root
mkfs.btrfs -L "home" /dev/mapper/vg1-home
```
# 7 - Mounting and Booting

In order to mount these filesystems via fstab you can follow the example below.

```
/dev/mapper/vg0-swap    none            swap            sw              0 0
/dev/mapper/vg0-root    /               btrfs           noatime         0 1
/dev/mapper/vg0-home    /home           btrfs           noatime         0 3
```

In order to pass the right commands to your initramfs when booting with grub we set `GRUB_CMDLINE_LINUX` value to something like the following.

`GRUB_CMDLINE_LINUX="crypt_root=UUID=3f10f458-b151-4620-a47c-4a6427e6aede dolvm dobtrfs real_root=/dev/mapper/vg0-root root_trim=yes root_keydev=UUID=1DB9-8480 root_key=root0-luks-key.gpg real_resume=/dev/mapper/vg0-swap"`
