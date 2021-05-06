# raid_linux

## Добавить в Vagrantfile еще дисков

Начальный Vagrantfile берем из этого [репозитория](https://github.com/erlong15/otus-linux)  

Добавим еще один диск :

```Vagrantfile
                :sata5 => {
                        :dfile => './sata5.vdi',
                        :size => 250, # Megabytes
                        :port => 5
                }
```

Запустим виртулальную машину : 

```
 vagrant up
```

Посмотрим какие блочные устройства у нас есть :

```console
[root@otuslinux vagrant]# lshw -short | grep disk
/0/100/1.1/0.0.0    /dev/sda  disk        42GB VBOX HARDDISK
/0/100/d/0          /dev/sdb  disk        262MB VBOX HARDDISK
/0/100/d/1          /dev/sdc  disk        262MB VBOX HARDDISK
/0/100/d/2          /dev/sdd  disk        262MB VBOX HARDDISK
/0/100/d/3          /dev/sde  disk        262MB VBOX HARDDISK
/0/100/d/0.0.0      /dev/sdf  disk        262MB VBOX HARDDISK
```

Занулим на всякий случай суперблоки:

```console

[root@otuslinux vagrant]# mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
mdadm: Unrecognised md component device - /dev/sdb
mdadm: Unrecognised md component device - /dev/sdc
mdadm: Unrecognised md component device - /dev/sdd
mdadm: Unrecognised md component device - /dev/sde
mdadm: Unrecognised md component device - /dev/sdf

```
Создадим Raid5 следуещей командой: 

```console
[root@otuslinux vagrant]# mdadm --create --verbose /dev/md0 -l 5 -n 5 /dev/sd{b,c,d,e,f}
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 253952K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```

Проверим что RAID собрался нормально:

```console
[root@otuslinux vagrant]# cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdf[5] sde[3] sdd[2] sdc[1] sdb[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]
      
unused devices: <none>
```

```console
[root@otuslinux vagrant]# mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Thu May  6 18:23:43 2021
        Raid Level : raid5
        Array Size : 1015808 (992.00 MiB 1040.19 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Thu May  6 18:23:50 2021
             State : clean 
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 9c80b4c4:2dfa8dc9:a828b81d:2cc801c0
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       5       8       80        4      active sync   /dev/sdf
```

## Создание конфигурационного файла mdadm.conf

Сначала убедимся, что информация верна:

```console
[root@otuslinux vagrant]# mdadm --detail --scan --verbose
ARRAY /dev/md0 level=raid5 num-devices=5 metadata=1.2 name=otuslinux:0 UUID=9c80b4c4:2dfa8dc9:a828b81d:2cc801c0
   devices=/dev/sdb,/dev/sdc,/dev/sdd,/dev/sde,/dev/sdf
[root@otuslinux vagrant]# 

```

Создадим mdadm.conf : 

```console
[root@otuslinux ~]# echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
[root@otuslinux ~]# mdadm --detail --scan --verbose | awk '/ARRAY/ {print}'
ARRAY /dev/md0 level=raid5 num-devices=5 metadata=1.2 name=otuslinux:0 UUID=9c80b4c4:2dfa8dc9:a828b81d:2cc801c0
[root@otuslinux ~]# mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
```

## Сломать/починить RAID

Фейлим одно из устройств :

```console
[root@otuslinux ~]#  mdadm /dev/md0 --fail /dev/sde
mdadm: set /dev/sde faulty in /dev/md0
```

Cмотрим как это отразилось на RAID :

```console
[root@otuslinux ~]# cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdf[5] sde[3](F) sdd[2] sdc[1] sdb[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/4] [UUU_U]

[root@otuslinux ~]#  mdadm -D /dev/md0 | grep Devices
      Raid Devices : 5
     Total Devices : 5
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 1
     Spare Devices : 0
```

Удалим “сломанный” диск из массива:

```console
[root@otuslinux ~]# mdadm /dev/md0 --remove /dev/sde
mdadm: hot removed /dev/sde from /dev/md0
```

"Вставим" новый диск :

```console
[root@otuslinux ~]# mdadm /dev/md0 --add /dev/sde
mdadm: added /dev/sde
``` 

Смотрим процесс ребилда :

```console
[root@otuslinux ~]# cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[6] sdf[5] sdd[2] sdc[1] sdb[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]
      
unused devices: <none>
```
```
