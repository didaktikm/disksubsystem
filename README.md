# Дисковая подсистема Linux

[Habr](https://habr.com/ru/post/248073/)

## Задача №1 Добавить диски в VagrantFile и создать RAAID который собирается при старте виртуальной машины.

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

