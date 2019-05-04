# Дисковая подсистема Linux

## Задача №1 Добавить диски в VagrantFile и создать RAID1 который собирается при старте виртуальной машины.

#### Добавляем диски к существующему массиву.

```
:disks => {
		:sata1 => {
			:dfile => './sata1.vdi',
			:size => 20000,
			:port => 1
		}
		:sata2 => {
                        :dfile => './sata2.vdi',
                        :size => 250, # Megabytes
			:port => 2
		},
                :sata3 => {
                        :dfile => './sata3.vdi',
                        :size => 250,
                        :port => 3
                },
                :sata4 => {
                        :dfile => './sata4.vdi',
                        :size => 250, # Megabytes
                        :port => 4
                },
				:sata5 => {
                        :dfile => './sata5.vdi',
                        :size => 250, # Megabytes
                        :port => 5
                },
				:sata6 => {
                        :dfile => './sata6.vdi',
                        :size => 250, # Megabytes
                        :port => 6
                }
```                

#### Добавляем в VagrantFile функционал для сборки RAID при старте машины.

в секцию:
```
box.vm.provision "shell", inline: <<-SHELL
```	          
добавляем: 

```
yum install -y mdadm smartmontools hdparm gdisk mc curl ansible # устанавливаем необходимые пакеты
		  mdadm --zero-superblock --force /dev/sd{b,c,d,e,f,g} # обнуляем суперблок
		  mdadm --create  /dev/md0 -l 10 -n 6 /dev/sd{b,c,d,e,f,g} # создаем RAID 10 уровня из 6 дисков
		  mkdir /etc/mdadm # содаем директорию для конфига mdadm (у меня она почему то не создалась при установке пакета)
		  echo "DEVICE partitions" > /etc/mdadm/mdadm.conf # создаем конфиг mdadm
		  mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf # добавляем информацию о массиве
		  parted -s /dev/md0 mklabel gpt # создаем таблицу разделов на массиве
		  parted /dev/md0 mkpart primary ext4 0% 20% # 
		  parted /dev/md0 mkpart primary ext4 20% 40% ##
		  parted /dev/md0 mkpart primary ext4 40% 60% ### создаем разделы
		  parted /dev/md0 mkpart primary ext4 60% 80% ##
		  parted /dev/md0 mkpart primary ext4 80% 100% #
		  for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done # создаем файловую систему ext4 на разделах 
		  mkdir -p /raid/part{1,2,3,4,5} # создаем директорию для каждого раздела
		  for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done # монтируем разделы, каждый в свою директорию
		  df -h | grep /raid | awk '{print($1 FS $6)}' >> /etc/fstab # добавляем информацию в fstab для того чтобы разделы монтировались при загрузке
		  ansible-galaxy install viasite-ansible.zsh --force # устанавливаем роль zsh
		  curl https://raw.githubusercontent.com/didaktikm/ansible-role-zsh/master/playbook.yml > /tmp/zsh.yml # загружаем playbook
		  ansible-playbook -i "localhost," -c local /tmp/zsh.yml 
		  ansible-playbook -i "localhost," -c local /tmp/zsh.yml --extra-vars="zsh_user=$(whoami)" # в качестве бонуса, если зайти под root получим красивый шелл с настройками и плагинами.
```
В результате при старте получаем CeontOS 7 с дополнительными 6 дисками, которые при старте соберуться в RAID 10.

## Задача №2 Перенести рабочую систему CentOS 7 на програмный RAID.
В системе установленны два диска. ```/dev/sda``` с системой, и ```/dev/sdb``` дополнительный диск по зеркало RAID1

#### Создаем раздел на sdb любым удобрым способом

```
fdisk /dev/sdb
n
p
1
w
```

#### Создаем RAID1 и указываем что один диск отсутствует

```
mdadm --create --verbose /dev/md0 -l 1 -n 2 missing /dev/sdb1
```

#### Форматируем массив

```
mkfs.xfs /dev/md0
```

#### Монтируем массив в ```/mnt```

```
mount /dev/md0 /mnt/
```

#### Копируем рабочую систему в ```/mnt```

```
rsync -axu / /mnt/
```

#### Монтируем служебные файловые системы в ```/mnt```, и CHROOTимся в новый корень

```
mount --bind /proc /mnt/proc && mount --bind /dev /mnt/dev && mount --bind /sys /mnt/sys && mount --bind /run /mnt/run && chroot /mnt/
```

#### Находим uuid нашего массива и вставляем его с заменой в ```/etc/fstab/``` для корневого раздела

```
blkid | grep md
nano /ecc/fstab
```

#### Создаем конфиг для mdadm

```
mkdir /etc/mdadm
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
```

#### Пересоздаем initramfs. В CentOS это делается следующей командой:

```
dracut --force /boot/initramfs-$(uname -r).img $(uname -r)
```
#### Добовляем опцию ядра ```rd.auto=1``` явно, для этого, добавляем ее в ```GRUB_CMDLINE_LINUX```
```
nano /etc/default/grub
```

#### Установим GRUB на второй диск и пересоздадим конфигурацию

```
grub2-mkconfig -o /boot/grub2/grub.cfg && grub2-install /dev/sdb
```
И проверяем:

```
cat /boot/grub2/grub.cfg | grep -E "rd.auto|mduuid"                                                                                                                              
	set root='**mduuid**/4f287a04e5a3190ba47f6c579d9cb04a'
	  search --no-floppy --fs-uuid --set=root --hint='mduuid/4f287a04e5a3190ba47f6c579d9cb04a'  b058d7c6-55af-4ab7-8f91-d95459e7a7c9
	linux16 /boot/vmlinuz-3.10.0-957.5.1.el7.x86_64 root=UUID=b058d7c6-55af-4ab7-8f91-d95459e7a7c9 ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto **rd.auto=1**
```

Теперь можно перегрузить машину и в загрузочном меню указать грузиться со второго диска.
У меня после загрузки не пускало в систему ни локально ни по ssh. В документации советуют в корне создать файл ```touch /.autorelabel```, но мне не помогло. Я не нашел пока решения кроме как отключить ```SELINUX```
Меняем в настройках политики ```/etc/selinux/config``` с ```enforcing``` на ```permissive```
Теперь после перезагрузки система пустит в консоль

```
exit
reboot
```

#### После загрузки со второго диска, добавляекм первый в наш массив

```
mdadm --manage /dev/md0 --add /dev/sda1
```

Дожидаемся окончания синхронизации

```
watch cat /proc/mdstat
```

и переустанавливаем GRUB на первом диске

```
grub2-install /dev/sda
```

Деперь у нас система на RAID1. При желании можно перезагрузиться с первого диска и проверить работоспособность.

```
mdadm -D /dev/md0
```

```
[root@localhost ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0         40G   23G   15G  62% /
devtmpfs        489M     0  489M   0% /dev
tmpfs           496M     0  496M   0% /dev/shm
tmpfs           496M  6.7M  489M   2% /run
tmpfs           496M     0  496M   0% /sys/fs/cgroup
tmpfs           100M     0  100M   0% /run/user/1000
[root@localhost ~]$ lsblk 
NAME    MAJ:MIN RM SIZE RO TYPE  MOUNTPOINT
sda       8:0    0  40G  0 disk  
└─sda1    8:1    0  40G  0 part  
  └─md0   9:0    0  40G  0 raid1 /
sdb       8:16   0  40G  0 disk  
└─sdb1    8:17   0  40G  0 part  
  └─md0   9:0    0  40G  0 raid1 /
```