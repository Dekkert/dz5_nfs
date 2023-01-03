#  Vagrant стенд для NFS

Задание:
```text
    1. vagrant up должен поднимать 2 виртуалки: сервер и клиент;
    2. на сервер должна быть расшарена директория;
    3. на клиента она должна автоматически монтироваться при старте (fstab или autofs);
    4. в шаре должна быть папка upload с правами на запись;
    5. требования для NFS: NFSv3 по UDP, включенный firewall.
```

Выполнение:

Развернём две виртуальные машину с помощью преднастроенного Vagrantfile:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure(2) do |config|
    config.vm.box = "centos/7"
    config.vm.box_version = "2004.01"
    config.vm.provider "virtualbox" do |v|
    v.memory = 256
    v.cpus = 1
    end
    config.vm.define "nfss" do |nfss|
    nfss.vm.network "private_network", ip: "192.168.56.10",
    virtualbox__intnet: "net1"
    nfss.vm.hostname = "nfss"
    nfss.vm.provision "shell", path: "nfss_script.sh"
    end
    config.vm.define "nfsc" do |nfsc|
    nfsc.vm.network "private_network", ip: "192.168.56.11",
    virtualbox__intnet: "net1"
    nfsc.vm.hostname = "nfsc"
    nfsc.vm.provision "shell", path: "nfsc_script.sh"
    end
    end
```
С помощью скрипта на сервере настроим:
```text
1. доустановим утилиты
2. включим firewall
3. включим сервер NFS (для конфигурации NFSv3 over UDP он не требует
дополнительной настройки)
4. cоздаём и настраиваем директорию, которая будет экспортирована
в будущем
5. создаём в файле /etc/exports структуру, которая позволит
экспортировать ранее созданную директорию
6. экспортируем ранее созданную директорию
```
nfss_script.sh
```bash
#!/bin/bash

yum install -y nfs-utils
systemctl enable firewalld --now
firewall-cmd --add-service="nfs3" \
--add-service="rpc-bind" \
--add-service="mountd" \
--permanent
firewall-cmd --reload
systemctl enable nfs --now
mkdir -p /srv/share/upload
chown -R nfsnobody:nfsnobody /srv/share
chmod 0777 /srv/share/upload
cat << EOF > /etc/exports
/srv/share 192.168.56.11/24(rw,sync,root_squash)
EOF
exportfs -r
```
С помощью скрипта на клиенте настроим:
```text
1. доустановим утилиты
2. включим firewall
3. добавляем в /etc/fstab строку
4. выполняем
systemctl daemon-reload
systemctl restart remote-fs.target
```

nfsc_script.sh
```bash
#!/bin/bash

yum install -y nfs-utils
systemctl enable firewalld --now
echo "192.168.56.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0" >> /etc/fstab
systemctl daemon-reload
systemctl restart remote-fs.target
```
После выполнения vagrant up запускаются две настроенные виратульные машины (сервер и клиент). Все проверки по домашнему заданию проходят. 

