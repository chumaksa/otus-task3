# otus-task3

# Работа с LVM

1. уменьшить том под / до 8G
2. выделить том под /home
3. выделить том под /var (/var - сделать в mirror)
4. для /home - сделать том для снэпшотов
5. прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)
6. Работа со снапшотами:
	* сгенерировать файлы в /home/
	* снять снэпшот
	* удалить часть файлов
	* восстановиться со снэпшота

## Уменьшить том под / до 8G;

### Решение

Посмотрим какие диски, разделы и их точки монтирования есть в нашей системе.
```

[root@lvm ~]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk
```

Из полученной информации видно, что в качестве PV выступает раздел sda3 диска sda. \
На этом PV находится VG - VolGroup00 c двумя LV(LogVol00 и LogVol01). \
Корневой раздел, который требуется уменьшить, находится на LogVol00. \

Определим файловую систему корневого раздела. Для этого используем ту же команду lsblk с ключиком -f.
```

[root@lvm ~]# lsblk -f
NAME                    FSTYPE      LABEL UUID                                   MOUNTPOINT
sda
├─sda1
├─sda2                  xfs               570897ca-e759-4c81-90cf-389da6eee4cc   /boot
└─sda3                  LVM2_member       vrrtbx-g480-HcJI-5wLn-4aOf-Olld-rC03AY
  ├─VolGroup00-LogVol00 xfs               b60e9498-0baa-4d9f-90aa-069048217fee   /
  └─VolGroup00-LogVol01 swap              c39c5bed-f37c-4263-bee8-aeb6a6659d7b   [SWAP]
sdb
sdc
sdd
sde
```

Видим, что наш корневой раздел имеют файловую систему XFS. XFS не умеет уменьшаться. Для выполнения задания \
перенесём данные на дргуой раздел, сделаем его корневым, а после уменьшим размер нашего исходного раздела и перенесём \
все данные обратно. В качестве диска для переноса будем использовать sdb. Подготовим его. Пометим диск в качестве PV
```

[root@lvm ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
```

Создаём VG
```

[root@lvm ~]# vgcreate vg_temp_root /dev/sdb
  Volume group "vg_temp_root" successfully created
```

Создаём LV
```

[root@lvm ~]# lvcreate -n lv_temp_root -l +100%FREE /dev/vg_temp_root
  Logical volume "lv_temp_root" created.
```

Создадим файловую систему XFS и смонтируем ее в каталог /mnt:
```

[root@lvm ~]# mkfs.xfs /dev/vg_temp_root/lv_temp_root
meta-data=/dev/vg_temp_root/lv_temp_root isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@lvm ~]# mount /dev/vg_temp_root/lv_temp_root /mnt
```

Для копирования раздела используем утилиту xfsdump. Устанавливаем её.
```

[root@lvm ~]# yum install xfsdump
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: centos-mirror.rbc.ru
 * extras: centos-mirror.rbc.ru
 * updates: mirror.docker.ru
Resolving Dependencies
--> Running transaction check
---> Package xfsdump.x86_64 0:3.1.7-3.el7_9 will be installed
--> Processing Dependency: attr >= 2.0.0 for package: xfsdump-3.1.7-3.el7_9.x86_64
--> Running transaction check
---> Package attr.x86_64 0:2.4.46-13.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

===============================================================================================================================================================================================================================================================================
 Package                                                         Arch                                                           Version                                                                  Repository                                                       Size
===============================================================================================================================================================================================================================================================================
Installing:
 xfsdump                                                         x86_64                                                         3.1.7-3.el7_9                                                            updates                                                         309 k
Installing for dependencies:
 attr                                                            x86_64                                                         2.4.46-13.el7                                                            base                                                             66 k

Transaction Summary
===============================================================================================================================================================================================================================================================================
Install  1 Package (+1 Dependent package)

Total download size: 374 k
Installed size: 1.1 M
Is this ok [y/d/N]: y
Is this ok [y/d/N]: y
Downloading packages:
(1/2): xfsdump-3.1.7-3.el7_9.x86_64.rpm                                                                                                                                                                                                                 | 309 kB  00:00:00
(2/2): attr-2.4.46-13.el7.x86_64.rpm                                                                                                                                                                                                                    |  66 kB  00:00:01
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                                                                                          267 kB/s | 374 kB  00:00:01
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : attr-2.4.46-13.el7.x86_64                                                                                                                                                                                                                                   1/2
  Installing : xfsdump-3.1.7-3.el7_9.x86_64                                                                                                                                                                                                                                2/2
  Verifying  : attr-2.4.46-13.el7.x86_64                                                                                                                                                                                                                                   1/2
  Verifying  : xfsdump-3.1.7-3.el7_9.x86_64                                                                                                                                                                                                                                2/2

Installed:
  xfsdump.x86_64 0:3.1.7-3.el7_9

Dependency Installed:
  attr.x86_64 0:2.4.46-13.el7

Complete!
```

Делаем копию.
```

[root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Wed Sep 27 18:59:55 2023
xfsdump: session id: 304d14d2-fddd-4203-8a9e-9aa4044318bd
xfsdump: session label: ""
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 876297728 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description:
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/VolGroup00-LogVol00
xfsrestore: session time: Wed Sep 27 18:59:55 2023
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: b60e9498-0baa-4d9f-90aa-069048217fee
xfsrestore: session id: 304d14d2-fddd-4203-8a9e-9aa4044318bd
xfsrestore: media id: f6d5f81b-e273-4461-bb49-76d765d392be
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 2718 directories and 23640 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 853343696 bytes
xfsdump: dump size (non-dir files) : 840158728 bytes
xfsdump: dump complete: 36 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 36 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```

Меняем точку монтирования для необходимых каталогов.
```

[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
```

Переходим в окружение временного корня и обновляем конфигурацию grub.
```

[root@lvm ~]# chroot /mnt/
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
```

Обновляем образы загрузки:
```

[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64 --force
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```

Далее меняем конфигурационном файле grub (/boot/grub2/grub.cfg). \
Меняем rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_temp_root/lv_temp_root \
Возвращаемся в старое окружение и перезагружаемся.
```

[root@lvm boot]# exit
exit
[root@lvm ~]# shutdown -r now
```

После перезагрузки проверяем, что наши изменения применились.
```

[vagrant@lvm ~]$ lsblk
NAME                        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                           8:0    0   40G  0 disk
├─sda1                        8:1    0    1M  0 part
├─sda2                        8:2    0    1G  0 part /boot
└─sda3                        8:3    0   39G  0 part
  ├─VolGroup00-LogVol01     253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00     253:2    0 37.5G  0 lvm
sdb                           8:16   0   10G  0 disk
└─vg_temp_root-lv_temp_root 253:0    0   10G  0 lvm  /
sdc                           8:32   0    2G  0 disk
sdd                           8:48   0    1G  0 disk
sde                           8:64   0    1G  0 disk
```

Теперь можно удалить старый логический том.
```

[root@lvm ~]# lvremove /dev/VolGroup00/LogVol00
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed
```

Создаём новый логический том с размером 8G.
```

[root@lvm ~]# lvcreate -n LogVol00 -L 8G /dev/VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.
```

Далее проделываем все те же операции, что и при создании временного корневого раздела.
```

[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol00
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@lvm ~]# mount /dev/VolGroup00/LogVol00 /mnt/

[root@lvm ~]# xfsdump -J - /dev/vg_temp_root/lv_temp_root | xfsrestore -J - /mnt
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Wed Oct 25 17:03:50 2023
xfsdump: session id: 04b180cc-30ba-43b4-af8d-860a7efc46ce
xfsdump: session label: ""
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 881928704 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description:
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/vg_temp_root-lv_temp_root
xfsrestore: session time: Wed Oct 25 17:03:50 2023
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: ad96261c-dc2f-4753-b918-27e002287cf7
xfsrestore: session id: 04b180cc-30ba-43b4-af8d-860a7efc46ce
xfsrestore: media id: 9b5e7bba-8a3c-4e43-86fc-b5705ad40aa4
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 2763 directories and 23816 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 858697112 bytes
xfsdump: dump size (non-dir files) : 845396928 bytes
xfsdump: dump complete: 31 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 31 seconds elapsed
xfsrestore: Restore Status: SUCCESS

[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

[root@lvm boot]# chroot /mnt/

[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done

[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64 --force
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```

После перезагрузки проверяем, что наш корневой раздел поменялся успешно.
```

[vagrant@lvm ~]$ lsblk
NAME                        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                           8:0    0   40G  0 disk
├─sda1                        8:1    0    1M  0 part
├─sda2                        8:2    0    1G  0 part /boot
└─sda3                        8:3    0   39G  0 part
  ├─VolGroup00-LogVol00     253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01     253:1    0  1.5G  0 lvm  [SWAP]
sdb                           8:16   0   10G  0 disk
└─vg_temp_root-lv_temp_root 253:2    0   10G  0 lvm
sdc                           8:32   0    2G  0 disk
sdd                           8:48   0    1G  0 disk
sde                           8:64   0    1G  0 disk
```

## выделить том под /home;

### Решение

Создаём новый LV в VG VolGroup00 размером 2G, который будем использовать под /home. \
Создаём файловую систему и монтируем в /mnt.
```

[root@lvm ~]# lvcreate -n lv_home -L 2G /dev/VolGroup00
  Logical volume "lv_home" created.

[root@lvm ~]# mkfs.xfs /dev/VolGroup00/lv_home
meta-data=/dev/VolGroup00/lv_home isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@lvm ~]# mount /dev/VolGroup00/lv_home /mnt
```

Далее скопируем все данные с текущего /home в /mnt.
```

[root@lvm ~]# cp -aR /home/* /mnt/
```

Удаляем все данные с текущего /home
```

[root@lvm ~]# rm -rf /home/*
```

Далее меняем точку монтирования нашего LV - lv_home.
```

[root@lvm ~]# umount /mnt/
[root@lvm ~]# mount /dev/VolGroup00/lv_home /home/
[root@lvm ~]# df -h
Filesystem                       Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol00  8.0G  879M  7.2G  11% /
devtmpfs                         110M     0  110M   0% /dev
tmpfs                            118M     0  118M   0% /dev/shm
tmpfs                            118M  4.5M  114M   4% /run
tmpfs                            118M     0  118M   0% /sys/fs/cgroup
/dev/sda2                       1014M   61M  954M   6% /boot
tmpfs                             24M     0   24M   0% /run/user/1000
/dev/mapper/VolGroup00-lv_home   2.0G   33M  2.0G   2% /home
```

Для автоматического монтирования /home правим файл /etc/fstab. Добавляем следующее содержимое:
```

/dev/mapper/VolGroup00-lv_home 		/home        xfs     defaults        0 0
```
