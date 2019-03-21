## cryptsetup Ubuntu
---

### Partition layout
EFI partition (only needed if secure boot is activated and use uefi), boot partition, cryptsetup partition
*Type* 		*fstype*	*size*
efi 		vfat		550M
boot 		ext4		2G
cryptsetup 	luks 		100% rest

- Create this partition schema via parted or disk
```bash
sudo fdisk /dev/nvme0n1
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
lvcreate -n vgroot -L 20G vgcryptsetup
lvcreate -n vghome -l 100%FREE vgcryptsetup
```
---

### System installer
- Run the system installer. When it comes to the point for system partition choose something else
- After successfull install click continue testing
---

### Switch into cryptsetup
- First we need to get the uuid of our encrypted partition
```bash
sudo blkid /dev/nvme0n1p3
```
- Now we can start by mounting our partitions
```bash
sudo mount /dev/mapper/vgcryptsetup-vgroot /mnt
sudo mount /dev/nvme0n1p2 /mnt/boot
sudo mount /dev/nvme0n1p1 /mnt/boot/efi
sudo mount /dev/mapper/vgcryptsetup-vghome /mnt/home
sudo mount --bind /dev /mnt/dev
sudo chroot /mnt
mount -t proc proc /proc
mount -t sysfs sys /sys
mount -t devpts devpts /dev/pts
```
- Create a file called crypttab in /etc/crypttab
```bash
# <target name>	<source device>		<key file>	<options>
echo "cryptsetup UUID=UUIDFROMBLKID none luks" > /etc/crypttab
```
- Last but not least recreate initramfs
```bash
update-initramfs -k all -c
```
