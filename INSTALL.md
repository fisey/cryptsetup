## cryptsetup Ubuntu
---

### Description
I switched from Mac to Ubuntu. Due to company policies I need to encrypt my system setup. Because the standard ubuntu setup comes
with a partition schema I don't like, I decided to create for myself a short guideline. You can find more detailed guidelines on websearches as well.
---

### Partition layout
- This command will automatically create all needed partitions. The partition table after will look like (1 - efi - 550M / 2 - boot - 2G / 3 - cryptsetup - rest).

```bash
sed -e 's/\s*\([\+0-9a-zA-Z]*\).*/\1/' << EOF | sudo fdisk /dev/nvme0n1
  g
  n
  p
  1

  +550M
  t
  1
  n
  p
  2

  2G
  n
  p


  p
  w
  q
EOF
```
---

### Cryptsetup
- Format the created cryptsetup partition with luksFormat. You need to choose your disk encryption password at this point.
```bash
cryptsetup luksFormat -c aes-xts-plain64 -s 512 -h sha512 -y /dev/nvme0n1p3
```
- Now open the formated partition
```bash
cryptsetup luksOpen /dev/nvme0n1p3 cryptsetup
```
- Initialize physical volume
```bash
pvcreate /dev/mapper/cryptsetup
```
- Create volume group
```bash
vgcreate vgcryptsetup /dev/mapper/cryptsetup
```
- Create logical volumes for root and home dir
```bash
lvcreate -n root -L 20G vgcryptsetup
lvcreate -n home -l 100%FREE vgcryptsetup
```
---

### System installer
- Now you can close the terminal and click on *Install Ubuntu*. The icon should be on your desktop.
- Click through the installer.
- At the disk setup part of the setup choose *Something else*.
- Now you need to define the mount points and the file system types.
- For bootloader choose you primary disk */dev/nvme0n1*.
---

### Switch into cryptsetup
- First we need to get the uuid of our encrypted partition. Save the UUID cause it's needed in further installation process.
```bash
sudo blkid /dev/nvme0n1p3
```
- Now we can start by mounting our partitions
```bash
sudo mount /dev/mapper/vgcryptsetup-root /mnt
sudo mount /dev/nvme0n1p2 /mnt/boot
sudo mount /dev/nvme0n1p1 /mnt/boot/efi
sudo mount /dev/mapper/vgcryptsetup-home /mnt/home
sudo mount --bind /dev /mnt/dev
sudo chroot /mnt
mount -t proc proc /proc
mount -t sysfs sys /sys
mount -t devpts devpts /dev/pts
```
- Create a file called crypttab in /etc/crypttab. You need to replace the part *<UUIDFROMBLKID>* with the actual UUID of the partion you got earlier.
```bash
# <target name>	<source device>		<key file>	<options>
echo "cryptsetup UUID=<UUIDFROMBLKID> none luks" > /etc/crypttab
```
- Last but not least recreate initramfs
```bash
update-initramfs -k all -c
```

### End
Now just reboot and enjoy.
