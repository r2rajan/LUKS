# LUKS - Encrypt/decrypt disk, Partition, Volume or Volume Group. 

I am using a volume as an volume. You can use a volume group, disk, disk-partition, volume or volume group itself.

**1. Prepare a key file with random data**
```
dd if=/dev/random bs=32 count=1 of=/root/fs.key
chown root:root /root/fs.key
chmod 600 /root/fs.key
```
**2. Initialize a LUKS volume, setting the key-slot 0 from the new key file**

Note: My storage background forces me to use only volumes from lvm. You can use disk partitions as well.

```
cryptsetup luksFormat /dev/mapper/cryptodg-cryptvol01 /root/fs.key
```
**3. Initialize the volume for metadata storage**

After a LUKSv1 volume is initialized using cryptsetup(8), it must also be initialized for metadata storage by luksmeta init. Once this is complete, the device is ready to store metadata.
```
luksmeta init -d /dev/mapper/cryptodg-cryptvol01

```
The luksmeta utility enables an administrator to store metadata in the gap between the end of the LUKSv1 header and the start of the encrypted data. This is useful for storing data that is available before the volume is unlocked, usually for use during the volume unlock process.
```
luksmeta show -d /dev/mapper/cryptodg-cryptvol01
0   active empty
1 inactive empty
2 inactive empty
3 inactive empty
4 inactive empty
5 inactive empty
6 inactive empty
7 inactive empty
```
**4. Use a human readable name to address the LUKS formatted volume.**

This is the mapping we are creating to the original volume formatted in step#2
```
luks-cryptvol01
```
**5. Plan to automatically decrypt the volume during reboot using the key created in step#1.** 
You need to add the encrypted volume to /etc/crypttab file along with its UUID and the key so the init process can unlock the volume automatically
```
echo "luks-cryptvol01 UUID=$(cryptsetup luksUUID /dev/mapper/cryptodg-cryptvol01) /root/fs.key" >> /etc/crypttab
```

**6. [Optional] - If you want to unlock the volume manually, you can use the below command**
```
cryptsetup luksOpen /dev/mapper/cryptodg-cryptvol01 luks-cryptvol01  --key-file /root/fs.key
```
**7. Reboot the system just to make sure everything works as expected**
**8. Check the mapped encrypted volume is visible. You can also check /dev/mapper**
```
lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sdb                        8:16   0   16G  0 disk  
└─sdb1                     8:17   0   16G  0 part  
  ├─cryptodg-lvol01       253:5    0    8G  0 lvm   
    └─luks-cryptovol01    253:6    0    8G  0 crypt /mnt
```

**9. Now lay a file system over the the mapped volume.** 
You cannot lay a file system over the base volume as it will throw an "in-use" error.
```
mkfs.xfs /dev/mapper/luks-cryptovol01
```
