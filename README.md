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
