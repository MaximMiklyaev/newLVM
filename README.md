#скачиваем Vagrantfile

git clone https://github.com/MaximMiklyaev/newLVM.git

#переходим в директорию

cd /newLVM

#запускаем образ

vagrant up

#Устанавка утилит в авторежиме

#xvfdump(резервное копирование и восстановление файловой системы)

#nano(консольный текстовый редактор)

sudo yum install -y xfsdump

sudo yum install -y nano.x86_64

#переход VM

vagrant ssh

#переходим под root права

sudo -i

#выводит список всех блоков хранения информации

lsblk
       вывод:
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

#подготовка тома

pvcreate /dev/sdb

vgcreate vg_root /dev/sdb

lvcreate -n lv_root -l +100%FREE /dev/vg_root

#создание файловой системы

mkfs.xfs /dev/vg_root/lv_root
       
      вывод:
             meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
                      =                       sectsz=512   attr=2, projid32bit=1
                      =                       crc=1        finobt=0, sparse=0
             data     =                       bsize=4096   blocks=2620416, imaxpct=25
                      =                       sunit=0      swidth=0 blks
             naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
             log      =internal log           bsize=4096   blocks=2560, version=2
                      =                       sectsz=512   sunit=0 blks, lazy-count=1
             realtime =none                   extsz=4096   blocks=0, rtextents=0


#монтирование файловой системы

mount /dev/vg_root/lv_root /mnt

#копирование данных в 

xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt

#проверка скопированых файлов

ls /mnt

        вывод:
              bin   dev  home  lib64  mnt  proc  run   srv  tmp  vagrant
              boot  etc  lib   media  opt  root  sbin  sys  usr  var

#эмитация текущий root 

for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

#временная смена корня

chroot /mnt/

#обновляем grub

grub2-mkconfig -o /boot/grub2/grub.cfg

#обновление образа initrd

cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

#для того что бы загрузился нужный root правим файл /boot/grub2/grub.cfg

nano /boot/grub2/grub.cfg

#Находим строку rd.lvm.lv=VolGroup00/LogVol00

#и правим её на rd.lvm.lv=vg_root/lv_root

    #нажимаем - ctrl+w

     #вводим - rd.lvm - enter - находим - rd.lvm.lv=VolGroup00/LogVol00 правим на - rd.lvm.lv=vg_root/lv_root

      #вводим - ctrl+x 

       #вводим - y 

#перезагружаем систему

cd ..

reboot

#подколючаемся к VM

vagrant ssh

#переходим под root права

sudo -i

#проверяем что система загрузилась с новым root томом

lsblk

      вывод:
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

#удаление старого тома 40gb 

lvremove /dev/VolGroup00/LogVol00

#создание нового тома 8gb

lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00

#cоздаем файловую систему на томе созданом выше

mkfs.xfs /dev/VolGroup00/LogVol00

        вывод:
              meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
                       =                       sectsz=512   attr=2, projid32bit=1
                       =                       crc=1        finobt=0, sparse=0
              data     =                       bsize=4096   blocks=2097152, imaxpct=25
                       =                       sunit=0      swidth=0 blks
              naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
              log      =internal log           bsize=4096   blocks=2560, version=2
                       =                       sectsz=512   sunit=0 blks, lazy-count=1
              realtime =none                   extsz=4096   blocks=0, rtextents=0


#монтируем том

mount /dev/VolGroup00/LogVol00 /mnt

#копируем все данные с / раздела в /mnt

xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt

#снова переконфигурируем grub

for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

chroot /mnt/

grub2-mkconfig -o /boot/grub2/grub.cfg

#обновим образ initrd

cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; *s/.img//g"` --force; done

#не выходя из chroot и не перезагружаясь выделим том под /var

#подготавливаем свободные диски,создаем зеркало

#создание зеркала

pvcreate /dev/sdc /dev/sdd

vgcreate vg_var /dev/sdc /dev/sdd

lvcreate -L 950M -m1 -n lv_var vg_var

#создаем файловую систему и перемещаем туда /var

mkfs.ext4 /dev/vg_var/lv_var

mount /dev/vg_var/lv_var /mnt

cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/

#сохраняем данные старого /var

mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

#монтируем новyй var в каталог /var

umount /mnt

mount /dev/vg_var/lv_var /var

#правим fstab для автоматического монтирование /var

echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

#перезагружаем систему

#вводим Ctrl+D для выхода из пользователя root

reboot

#подколючаемся к VM

vagrant ssh

#переходим под root права

sudo -i

#Проверяем что система загрузилась с новым root томом на 8gb

lsblk
      вывод:
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
                ¦ L-vg_var-lv_var        253:6    0  952M  0 lvm  /var
                L-vg_var-lv_var_rimage_0 253:3    0  952M  0 lvm
                  L-vg_var-lv_var        253:6    0  952M  0 lvm  /var
               sdd                        8:48   0    1G  0 disk
                +-vg_var-lv_var_rmeta_1  253:4    0    4M  0 lvm
                ¦ L-vg_var-lv_var        253:6    0  952M  0 lvm  /var
                L-vg_var-lv_var_rimage_1 253:5    0  952M  0 lvm
                  L-vg_var-lv_var        253:6    0  952M  0 lvm  /var
               sde                        8:64   0    1G  0 disk


#eдаление временного Volume Group

lvremove /dev/vg_root/lv_root

vgremove /dev/vg_root

pvremove /dev/sdb

#выделение тома под /home

lvcreate -n LogVol_Home -L 2G /dev/VolGroup00

#создание файловой системы и перемещаем туда /home

mkfs.xfs /dev/VolGroup00/LogVol_Home

mount /dev/VolGroup00/LogVol_Home /mnt/

cp -aR /home/* /mnt/

rm -rf /home/*

umount /mnt

mount /dev/VolGroup00/LogVol_Home /home/

#изменяем fstab для автоматического монтирование /home

echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

#делаем том для снапшотов

#сгенерируем файлы в home

touch /home/file{1..20}

#проверим сгенерировались файлы

ls /home/

    вывод:
          file1   file11  file13  file15  file17  file19  file20  file4  file6  file8  vagrant
          file10  file12  file14  file16  file18  file2   file3   file5  file7  file9

 
#снимаем снапшот

lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home

#удаляем часть файлов в /home и убедимся в этом

rm -f /home/file{11..20}

ls /home/

    вывод:
          file1  file10  file2  file3  file4  file5  file6  file7  file8  file9  vagrant

#восстанавливаем /home со снапшота

umount /home

lvconvert --merge /dev/VolGroup00/home_snap

      вывод:
            Merging of volume VolGroup00/home_snap started.
            VolGroup00/LogVol_Home: Merged: 100.00%

mount /home

#проверим восстановились ли файлы

ls /home/

      вывод:
            file1   file11  file13  file15  file17  file19  file20  file4  file6  file8  vagrant
            file10  file12  file14  file16  file18  file2   file3   file5  file7  file9


#проверим прописалось ли монтирование в fstab (Должны увидеть /home и /var с разными файловыми системами)

cat /etc/fstab

      вывод:
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

