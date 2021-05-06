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
Создаем
