# stands-03-lvm

Стенд для домашнего занятия "Файловые системы и LVM"

Загрузить машину с : [Vagrant box a3buki/lvm-homework v0.0.2 - Vagrant Cloud (vagrantup.com)](https://app.vagrantup.com/a3buki/boxes/lvm-homework/versions/0.0.2)    

либо запустить:  vagrant init a3buki/lvm-homework

Либо с: https://gitlab.com/otus_linux/stands-03-lvm

Далее после в терминале выполнить команды:

yum install xfsdump

pvcreate /dev/sdb

vgcreate vg_root /dev/sdb

lvcreate -n lv_root -l +100%FREE /dev/vg_root

mkfs.xfs /dev/vg_root/lv_root

mount /dev/vg_root/lv_root /mnt

xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt



#Реконфиг Груб2
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;s/.img//g"` --force; done

## В /boot/grub2/grub.cfg заменит  rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root
reboot

#### Скрипт2
lvremove /dev/VolGroup00/LogVol00
lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00

#Создаем ФС
mkfs.xfs /dev/VolGroup00/LogVol00
mount /dev/VolGroup00/LogVol00 /mnt

#Реконфиг Груб
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;s/.img//g"` --force; done

#Переносим var
pvcreate /dev/sdc /dev/sdd
vgcreate vg_var /dev/sdc /dev/sdd
lvcreate -L 950M -m1 -n lv_var vg_var
mkfs.ext4 /dev/vg_var/lv_var
mount /dev/vg_var/lv_var /mnt
cp -aR /var/* /mnt/
rsync -avHPSAX /var/ /mnt/
mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
umount /mnt
mount /dev/vg_var/lv_var /var
echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
exit
reboot


#### Скрипт 3
lvremove /dev/vg_root/lv_root
vgremove /dev/vg_root
pvremove /dev/sdb

#Создание home
lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
mkfs.xfs /dev/VolGroup00/LogVol_Home
mount /dev/VolGroup00/LogVol_Home /mnt/
cp -aR /home/* /mnt/
rm -rf /home/*
umount /mnt
mount /dev/VolGroup00/LogVol_Home /home/
echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

#/home - сделать том для снапшотов
touch /home/file{1..20}
lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
rm -f /home/file{11..20}

#Процесс востоновления
umount -l /home
lvconvert --merge /dev/VolGroup00/home_snap
mount /home
