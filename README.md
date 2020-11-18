#��������� Vagrantfile

git clone https://github.com/MaximMiklyaev/newLVM.git

#��������� � ����������

cd /newLVM

#��������� �����

vagrant up

#��������� ������ � ����������
#xvfdump(��������� ����������� � �������������� �������� �������)
#nano(���������� ��������� ��������)
sudo yum install -y xfsdump
sudo yum install -y nano.x86_64

#������� VM
vagrant ssh

#��������� ��� root �����
sudo -i
#������� ������ ���� ������ �������� ����������
lsblk
       �����:
             NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
               sda                       8:0    0   40G  0 disk
               +-sda1                    8:1    0    1M  0 part
               +-sda2                    8:2    0    1G  0 part /boot
               L-sda3                    8:3    0   39G  0 part
                  +-VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
                  L-VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
                sdb                       8:16   0   10G  0 disk
                sdc                       8:32   0    2G  0 disk
                sdd                       8:48   0    1G  0 disk
                sde                       8:64   0    1G  0 disk

#���������� ����
pvcreate /dev/sdb
vgcreate vg_root /dev/sdb
lvcreate -n lv_root -l +100%FREE /dev/vg_root
#�������� �������� �������
*mkfs.xfs /dev/vg_root/lv_root
       �����:
             meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
                      =                       sectsz=512   attr=2, projid32bit=1
                      =                       crc=1        finobt=0, sparse=0
             data     =                       bsize=4096   blocks=2620416, imaxpct=25
                      =                       sunit=0      swidth=0 blks
             naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
             log      =internal log           bsize=4096   blocks=2560, version=2
                      =                       sectsz=512   sunit=0 blks, lazy-count=1
             realtime =none                   extsz=4096   blocks=0, rtextents=0

#������������ �������� �������
mount /dev/vg_root/lv_root /mnt
#����������� ������ � 
xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
#�������� ������������ ������
ls /mnt
        �����:
              bin   dev  home  lib64  mnt  proc  run   srv  tmp  vagrant
              boot  etc  lib   media  opt  root  sbin  sys  usr  var
#�������� ������� root 
*for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
#��������� ����� �����
*chroot /mnt/
#��������� grub
*grub2-mkconfig -o /boot/grub2/grub.cfg
#���������� ������ initrd
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

#��� ���� ��� �� ���������� ������ root ������ ���� /boot/grub2/grub.cfg
nano /boot/grub2/grub.cfg
#������� ������ rd.lvm.lv=VolGroup00/LogVol00
#� ������ � �� rd.lvm.lv=vg_root/lv_root
    #�������� - ctrl+w
     #������ - rd.lvm - enter - ������� - rd.lvm.lv=VolGroup00/LogVol00 ������ �� - rd.lvm.lv=vg_root/lv_root
      #������ - ctrl+x 
       #������ - y 

#������������� �������
cd ..
reboot

#������������� � VM
vagrant ssh

#��������� ��� root �����
sudo -i

#��������� ��� ������� ����������� � ����� root �����
lsblk
      �����:

            NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
             sda                       8:0    0   40G  0 disk
              +-sda1                    8:1    0    1M  0 part
              +-sda2                    8:2    0    1G  0 part /boot
              L-sda3                    8:3    0   39G  0 part
                 +-VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
                 L-VolGroup00-LogVol00 253:2    0 37.5G  0 lvm
                sdb                       8:16   0   10G  0 disk
                 L-vg_root-lv_root       253:0    0   10G  0 lvm  /
                sdc                       8:32   0    2G  0 disk
                sdd                       8:48   0    1G  0 disk
                sde                       8:64   0    1G  0 disk

#�������� ������� ���� 40gb 
lvremove /dev/VolGroup00/LogVol00

#�������� ������ ���� 8gb
lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00

#c������ �������� ������� �� ���� �������� ����
mkfs.xfs /dev/VolGroup00/LogVol00
        �����:
              meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
                       =                       sectsz=512   attr=2, projid32bit=1
                       =                       crc=1        finobt=0, sparse=0
              data     =                       bsize=4096   blocks=2097152, imaxpct=25
                       =                       sunit=0      swidth=0 blks
              naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
              log      =internal log           bsize=4096   blocks=2560, version=2
                       =                       sectsz=512   sunit=0 blks, lazy-count=1
              realtime =none                   extsz=4096   blocks=0, rtextents=0


#��������� ���
mount /dev/VolGroup00/LogVol00 /mnt

#�������� ��� ������ � / ������� � /mnt
xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt

#����� ����������������� grub
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg

#������� ����� initrd
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; *s/.img//g"` --force; done

#�� ������ �� chroot � �� �������������� ������� ��� ��� /var
#�������������� ��������� �����,������� �������
#�������� �������
pvcreate /dev/sdc /dev/sdd
vgcreate vg_var /dev/sdc /dev/sdd
lvcreate -L 950M -m1 -n lv_var vg_var

#������� �������� ������� � ���������� ���� /var
mkfs.ext4 /dev/vg_var/lv_var
mount /dev/vg_var/lv_var /mnt
cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/

#��������� ������ ������� /var
mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

#��������� ���y� var � ������� /var
umount /mnt
mount /dev/vg_var/lv_var /var

#������ fstab ��� ��������������� ������������ /var
echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

#������������� �������
#������ Ctrl+D ��� ������ �� ������������ root
reboot

#������������� � VM
vagrant ssh

#��������� ��� root �����
sudo -i

#��������� ��� ������� ����������� � ����� root ����� �� 8gb
lsblk
      �����:
            NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
            sda                        8:0    0   40G  0 disk
             +-sda1                     8:1    0    1M  0 part
             +-sda2                     8:2    0    1G  0 part /boot
             L-sda3                     8:3    0   39G  0 part
                +-VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
                L-VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
               sdb                        8:16   0   10G  0 disk
                L-vg_root-lv_root        253:7    0   10G  0 lvm
               sdc                        8:32   0    2G  0 disk
                +-vg_var-lv_var_rmeta_0  253:2    0    4M  0 lvm
                � L-vg_var-lv_var        253:6    0  952M  0 lvm  /var
                L-vg_var-lv_var_rimage_0 253:3    0  952M  0 lvm
                  L-vg_var-lv_var        253:6    0  952M  0 lvm  /var
               sdd                        8:48   0    1G  0 disk
                +-vg_var-lv_var_rmeta_1  253:4    0    4M  0 lvm
                � L-vg_var-lv_var        253:6    0  952M  0 lvm  /var
                L-vg_var-lv_var_rimage_1 253:5    0  952M  0 lvm
                  L-vg_var-lv_var        253:6    0  952M  0 lvm  /var
               sde                        8:64   0    1G  0 disk


#e������� ���������� Volume Group
lvremove /dev/vg_root/lv_root
vgremove /dev/vg_root
pvremove /dev/sdb

#��������� ���� ��� /home
lvcreate -n LogVol_Home -L 2G /dev/VolGroup00

#�������� �������� ������� � ���������� ���� /home
mkfs.xfs /dev/VolGroup00/LogVol_Home
mount /dev/VolGroup00/LogVol_Home /mnt/
cp -aR /home/* /mnt/
rm -rf /home/*
umount /mnt
mount /dev/VolGroup00/LogVol_Home /home/

#�������� fstab ��� ��������������� ������������ /home
echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

#������ ��� ��� ���������
#����������� ����� � home
touch /home/file{1..20}

#�������� ��������������� �����
ls /home/
    �����:
          file1   file11  file13  file15  file17  file19  file20  file4  file6  file8  vagrant
          file10  file12  file14  file16  file18  file2   file3   file5  file7  file9

 
#������� �������
lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home

#������� ����� ������ � /home � �������� � ����
rm -f /home/file{11..20}
ls /home/
    �����:
          file1  file10  file2  file3  file4  file5  file6  file7  file8  file9  vagrant

#��������������� /home �� ��������
umount /home
lvconvert --merge /dev/VolGroup00/home_snap
      �����:
           Merging of volume VolGroup00/home_snap started.
           VolGroup00/LogVol_Home: Merged: 100.00%

mount /home

#�������� �������������� �� �����
ls /home/
      �����:
            file1   file11  file13  file15  file17  file19  file20  file4  file6  file8  vagrant
            file10  file12  file14  file16  file18  file2   file3   file5  file7  file9


#�������� ����������� �� ������������ � fstab (������ ������� /home � /var � ������� ��������� ���������)
cat /etc/fstab
      �����:
            #
            # /etc/fstab
            # Created by anaconda on Sat May 12 18:50:26 2018
            #
            # Accessible filesystems, by reference, are maintained under '/dev/disk'
            # See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
            #
            /dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0
            UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
            /dev/mapper/VolGroup00-LogVol01 swap                    swap    defaults        0 0
            UUID="49226b57-e7f3-4146-91a6-a206dae54a1b" /var ext4 defaults 0 0
            UUID="9bc80a31-bee9-4c42-bf51-2d15daafe957" /home xfs defaults 0 0

