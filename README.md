## Создание стенда Vagrant с NFS.

### Цели домашнего задания:

Научиться самостоятельно развернуть сервис NFS и подключить к нему клиента

### Описание домашнего задания.

Основная часть:

- `vagrant up` должен поднимать 2 настроенных виртуальных машины (сервер NFS и клиента) без дополнительных ручных действий
- на сервере NFS должна быть подготовлена и экспортирована директория
- в экспортированной директории должна быть поддиректория с именем **upload** с правами на запись в неё
- экспортированная директория должна автоматически монтироваться на клиенте при старте виртуальноймашины (systemd, autofs или fstab - любым способом)
- монтирование и работа NFS на клиенте должна быть организована с использованием NFSv3 по протоколу UDP
- firewall должен быть включен и настроен как на клиенте, так и на сервере

Для самостоятельной реализации:

- настроить аутентификацию через KERBEROS с использованием NFSv4

### Создаем тестовые виртуальные машины.

- Для начала, использован этот шаблон для создания виртуальных машин:

- [Шаблон Vagrantfile](Vagrantfile_old)

> Результат выполнения команды `vagrant up` 2 виртуальных машины: **nfss** для сервера NFS и **nfsc** для клиента

```
elena_leb@ubuntunbleb:~/NFS_DZ$ vagrant global-status
id       name          provider   state   directory                           
------------------------------------------------------------------------------
a0e7ff8  nfss          virtualbox running /home/elena_leb/NFS_DZ              
6d592e5  nfsc          virtualbox running /home/elena_leb/NFS_DZ  
```
### Настраиваем сервер NFS.

- Заходим на сервер.

```
vagrant ssh nfss
```
> Дальнейшие действия выполняются **от имени пользователя имеющего повышенные привилегии**, разрешающие описанные действия.
```
vagrant ssh nfss 
[vagrant@nfss ~]$ sudo -i
```
- Сервер NFS уже установлен в CentOS 7 как часть дистрибутива, так что нам нужно лишь доустановить утилиты, которые облегчат отладку.

```
sudo -i
yum install nfs-utils -y
```
```
[root@nfss ~]# yum install nfs-utils
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Determining fastest mirrors
 * base: mirror.previder.nl
 * extras: mirror.nl.datapacket.com
 * updates: mirror.nl.datapacket.com
base                                                                           | 3.6 kB  00:00:00     
extras                                                                         | 2.9 kB  00:00:00     
updates                                                                        | 2.9 kB  00:00:00     
(1/4): base/7/x86_64/group_gz                                                  | 153 kB  00:00:00     
(2/4): extras/7/x86_64/primary_db                                              | 250 kB  00:00:00     
(3/4): base/7/x86_64/primary_db                                                | 6.1 MB  00:00:02     
(4/4): updates/7/x86_64/primary_db                                             |  24 MB  00:00:03     
Resolving Dependencies
--> Running transaction check
---> Package nfs-utils.x86_64 1:1.3.0-0.66.el7 will be updated
---> Package nfs-utils.x86_64 1:1.3.0-0.68.el7.2 will be an update
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================================================
 Package               Arch               Version                           Repository           Size
======================================================================================================
Updating:
 nfs-utils             x86_64             1:1.3.0-0.68.el7.2                updates             413 k

Transaction Summary
======================================================================================================
Upgrade  1 Package

Total download size: 413 k
Is this ok [y/d/N]: y
Downloading packages:
No Presto metadata available for updates
warning: /var/cache/yum/x86_64/7/updates/packages/nfs-utils-1.3.0-0.68.el7.2.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
Public key for nfs-utils-1.3.0-0.68.el7.2.x86_64.rpm is not installed
nfs-utils-1.3.0-0.68.el7.2.x86_64.rpm                                          | 413 kB  00:00:01     
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-release-7-8.2003.0.el7.centos.x86_64 (@anaconda)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Is this ok [y/N]: y
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : 1:nfs-utils-1.3.0-0.68.el7.2.x86_64                                                1/2 
  Cleanup    : 1:nfs-utils-1.3.0-0.66.el7.x86_64                                                  2/2 
  Verifying  : 1:nfs-utils-1.3.0-0.68.el7.2.x86_64                                                1/2 
  Verifying  : 1:nfs-utils-1.3.0-0.66.el7.x86_64                                                  2/2 

Updated:
  nfs-utils.x86_64 1:1.3.0-0.68.el7.2                                                                 

Complete!
```
- Включаем firewall и проверяем, что он работает.

```
systemctl enable firewalld --now
systemctl status firewalld
```
```
[root@nfss ~]# systemctl enable firewalld --now
Created symlink from /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service to /usr/lib/systemd/system/firewalld.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/firewalld.service to /usr/lib/systemd/system/firewalld.service.
[root@nfss ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2023-12-10 07:02:31 UTC; 22s ago
     Docs: man:firewalld(1)
 Main PID: 3499 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─3499 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid

Dec 10 07:02:31 nfss systemd[1]: Starting firewalld - dynamic firewall daemon...
Dec 10 07:02:31 nfss systemd[1]: Started firewalld - dynamic firewall daemon.
Dec 10 07:02:31 nfss firewalld[3499]: WARNING: AllowZoneDrifting is enabled. This is considered...now.
Hint: Some lines were ellipsized, use -l to show in full.
```
- Разрешаем в firewall доступ к сервисам NFS.
```
firewall-cmd --add-service="nfs3" \
--add-service="rpc-bind" \
--add-service="mountd" \
--permanent
firewall-cmd --reload
```
```
[root@nfss ~]# firewall-cmd --add-service="nfs3" \
> --add-service="rpc-bind" \
> --add-service="mountd" \
> --permanent
success
[root@nfss ~]# firewall-cmd —reload
success
```
- Включаем сервер NFS.
> Для конфигурации NFSv3 over UDP в данном случае не требует дополнительной настройки. Посмотреть настройки по-умолчаниюм - **/etc/nfs.conf**
```
systemctl enable nfs --now
```
```
[root@nfss ~]# systemctl enable nfs --now
Created symlink from /etc/systemd/system/multi-user.target.wants/nfs-server.service to /usr/lib/systemd/system/nfs-server.service.
```
- Проверяем наличие слушаемых портов 2049/udp, 2049/tcp, 20048/udp, 20048/tcp, 111/udp, 111/tcp. 
> Они не все будут использоваться далее, но их наличие является подтверждением, что необходимые сервисы готовы принимать внешние подключения.
```
[root@nfss ~]# ss -tnplu | grep -E '2049|20048|111'
udp    UNCONN     0      0         *:20048                 *:*                   users:(("rpc.mountd",pid=22438,fd=7))
udp    UNCONN     0      0         *:111                   *:*                   users:(("rpcbind",pid=339,fd=6))
udp    UNCONN     0      0         *:2049                  *:*                  
udp    UNCONN     0      0      [::]:20048              [::]:*                   users:(("rpc.mountd",pid=22438,fd=9))
udp    UNCONN     0      0      [::]:111                [::]:*                   users:(("rpcbind",pid=339,fd=9))
udp    UNCONN     0      0      [::]:2049               [::]:*                  
tcp    LISTEN     0      128       *:111                   *:*                   users:(("rpcbind",pid=339,fd=8))
tcp    LISTEN     0      128       *:20048                 *:*                   users:(("rpc.mountd",pid=22438,fd=8))
tcp    LISTEN     0      64        *:2049                  *:*                  
tcp    LISTEN     0      128    [::]:111                [::]:*                   users:(("rpcbind",pid=339,fd=11))
tcp    LISTEN     0      128    [::]:20048              [::]:*                   users:(("rpc.mountd",pid=22438,fd=10))
tcp    LISTEN     0      64     [::]:2049               [::]:*                  
```
 
- Создаем и настраиваем директорию, которая будет экспортирована в будущем.
```
mkdir -p /srv/share/upload
chown -R nfsnobody:nfsnobody /srv/share
chmod 0777 /srv/share/upload
```
- Создаем в файле /etc/exports структуру, которая позволит экспортировать ранее созданную директорию.
```
[root@nfss upload]# cat << EOF > /etc/exports
> /srv/share 192.168.56.11/32(rw,sync,root_squash)
> EOF
```
- Экспортируем ранее созданную директорию.
```bash
[root@nfss upload]# exportfs -r
```
- Проверяем экспортированную директорию.
```
[root@nfss home]# exportfs -s
/srv/share  192.168.56.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```
### Настраиваем клиент NFS.
- Заходим на клиент.
```bash
vagrant ssh nfsc
```
> Дальнейшие действия выполняются **от имени пользователя имеющего повышенные привилегии**, разрешающие описанные действия.
- Доустановим вспомогательные утилиты.
```bash
yum install nfs-utils -y
```
``` 
[root@nfsc ~]# yum install nfs-utils -y
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Determining fastest mirrors
 * base: mirror.docker.ru
 * extras: mirror.corbina.net
 * updates: mirror.corbina.net
base                                                                                                                       | 3.6 kB  00:00:00     
extras                                                                                                                     | 2.9 kB  00:00:00     
updates                                                                                                                    | 2.9 kB  00:00:00     
(1/4): base/7/x86_64/group_gz                                                                                              | 153 kB  00:00:00     
(2/4): extras/7/x86_64/primary_db                                                                                          | 250 kB  00:00:00     
(3/4): base/7/x86_64/primary_db                                                                                            | 6.1 MB  00:00:02     
(4/4): updates/7/x86_64/primary_db                                                                                         |  24 MB  00:00:02     
Resolving Dependencies
--> Running transaction check
---> Package nfs-utils.x86_64 1:1.3.0-0.66.el7 will be updated
---> Package nfs-utils.x86_64 1:1.3.0-0.68.el7.2 will be an update
--> Finished Dependency Resolution

Dependencies Resolved

==================================================================================================================================================
 Package                          Arch                          Version                                      Repository                      Size
==================================================================================================================================================
Updating:
 nfs-utils                        x86_64                        1:1.3.0-0.68.el7.2                           updates                        413 k

Transaction Summary
==================================================================================================================================================
Upgrade  1 Package

Total download size: 413 k
Downloading packages:
No Presto metadata available for updates
warning: /var/cache/yum/x86_64/7/updates/packages/nfs-utils-1.3.0-0.68.el7.2.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
Public key for nfs-utils-1.3.0-0.68.el7.2.x86_64.rpm is not installed
nfs-utils-1.3.0-0.68.el7.2.x86_64.rpm                                                                                      | 413 kB  00:00:00     
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-release-7-8.2003.0.el7.centos.x86_64 (@anaconda)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : 1:nfs-utils-1.3.0-0.68.el7.2.x86_64                                                                                            1/2 
  Cleanup    : 1:nfs-utils-1.3.0-0.66.el7.x86_64                                                                                              2/2 
  Verifying  : 1:nfs-utils-1.3.0-0.68.el7.2.x86_64                                                                                            1/2 
  Verifying  : 1:nfs-utils-1.3.0-0.66.el7.x86_64                                                                                              2/2 

Updated:
  nfs-utils.x86_64 1:1.3.0-0.68.el7.2                                                                                                             

Complete!
```
- Включаем firewall и проверяем, что он работает.
```bash
systemctl enable firewalld --now
systemctl status firewalld
```
``` 
[root@nfsc ~]# systemctl enable firewalld --now
Created symlink from /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service to /usr/lib/systemd/system/firewalld.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/firewalld.service to /usr/lib/systemd/system/firewalld.service.
[root@nfsc ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2023-12-10 08:25:28 UTC; 40s ago
     Docs: man:firewalld(1)
 Main PID: 22597 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─22597 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid

Dec 10 08:25:28 nfsc systemd[1]: Starting firewalld - dynamic firewall daemon...
Dec 10 08:25:28 nfsc systemd[1]: Started firewalld - dynamic firewall daemon.
Dec 10 08:25:28 nfsc firewalld[22597]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure configuration option. It w... it now.
Hint: Some lines were ellipsized, use -l to show in full.

```
- Добавляем в /etc/fstab строку:
```bash
[root@nfsc ~]# echo "192.168.56.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0" >> /etc/fstab
```
- Выполняем:
```bash
[root@nfsc ~]# systemctl daemon-reload
[root@nfsc ~]# systemctl restart remote-fs.target
```
> Отметим, что в данном случае происходит автоматическая генерация systemd units в каталоге `/run/systemd/generator/`, которые производят монтирование при первом обращении к каталогу `/mnt/`.

```
[root@nfsc ~]# cat /run/systemd/generator/mnt.mount 
# Automatically generated by systemd-fstab-generator

[Unit]
SourcePath=/etc/fstab
Documentation=man:fstab(5) man:systemd-fstab-generator(8)

[Mount]
What=192.168.56.10:/srv/share/
Where=/mnt
Type=nfs
Options=vers=3,proto=udp,noauto,x-systemd.automount
```
> Обратим внимание на `vers=3` и `proto=udp`- соответствует NFSv3 over UDP.
- Заходим в директорию `/mnt/` и проверяем успешность монтирования.
```bash
[root@nfsc ~]# ls /mnt
upload
[root@nfsc ~]# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=32,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=10988)
192.168.56.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.56.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.56.10)
```
### Проверка работоспособности.

- Заходим на сервер.
- Заходим в каталог.

```bash
[vagrant@nfss ~]$ cd /srv/share/upload
```
- Создаем тестовый файл.
```
[vagrant@nfss upload]$ touch check_file
[vagrant@nfss upload]$ ls
check_file
```
- Заходим на клиент в каталог.
- Проверяем наличие ранее созданного файла.
```bash
[root@nfsc ~]# cd /mnt/upload
[root@nfsc upload]# ls
check_file
```
- Создаем тестовый файл.
```
[root@nfsc upload]# touch client_file
```
- Проверяем, что файл успешно создан и доступен на сервере.
```
[root@nfss upload]# ls 
check_file  client_file
```
> **Если вышеуказанные проверки прошли успешно, это значит, что проблем с правами нет.**

### Предварительная проверка - клиент:

- Перезагружаем клиент.
- Заходим на клиент.
- Заходим в каталог.
- Проверяем наличие ранее созданных файлов.
```
[root@nfsc upload]# systemctl reboot
Connection to 127.0.0.1 closed by remote host.
elena_leb@ubuntunbleb:~/NFS_DZ$ vagrant ssh nfsc
Last login: Sun Dec 10 10:00:13 2023 from 10.0.2.2
[vagrant@nfsc ~]$ sudo -i
[root@nfsc ~]# cd /mnt/upload
[root@nfsc upload]# ls
check_file  client_file
```
### Предварительная проверка - сервер:

- Заходим на сервер в отдельном окне терминала.
- Перезагружаем сервер.
- Заходим на сервер.
- Проверяем наличие файлов в каталоге.

```
[root@nfss upload]# systemctl reboot
Connection to 127.0.0.1 closed by remote host.
elena_leb@ubuntunbleb:~/NFS_DZ$ vagrant ssh nfss
Last login: Sun Dec 10 09:57:38 2023 from 10.0.2.2
[vagrant@nfss ~]$ cd /srv/share/upload/
[vagrant@nfss upload]$ ls
check_file  client_file
```
- Проверяем статус сервера NFS.
```
[root@nfss upload]# systemctl status nfs
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
  Drop-In: /run/systemd/generator/nfs-server.service.d
           └─order-with-mounts.conf
   Active: active (exited) since Sun 2023-12-10 10:27:49 UTC; 5min ago
  Process: 820 ExecStartPost=/bin/sh -c if systemctl -q is-active gssproxy; then systemctl reload gssproxy ; fi (code=exited, status=0/SUCCESS)
  Process: 795 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
  Process: 791 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
 Main PID: 795 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/nfs-server.service

Dec 10 10:27:49 nfss systemd[1]: Starting NFS server and services...
Dec 10 10:27:49 nfss systemd[1]: Started NFS server and services.
```
- Проверяем статус firewall.
```
[root@nfss upload]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2023-12-10 10:27:46 UTC; 6min ago
     Docs: man:firewalld(1)
 Main PID: 403 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─403 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid

Dec 10 10:27:45 nfss systemd[1]: Starting firewalld - dynamic firewall daemon...
Dec 10 10:27:46 nfss systemd[1]: Started firewalld - dynamic firewall daemon.
Dec 10 10:27:46 nfss firewalld[403]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure configuration option. It wil... it now.
Hint: Some lines were ellipsized, use -l to show in full.
```
- Проверяем экспорты.
```
[root@nfss upload]# exportfs -s
/srv/share  192.168.56.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```
- Проверяем работу RPC.
```
[root@nfss upload]# showmount -a 192.168.56.10
All mount points on 192.168.56.10:
192.168.56.11:/srv/share
```
### Проверяем клиент:

- Возвращаемся на клиент и перезагружаем его.
- Заходим на клиент.
- Проверяем работу RPC.
```
[root@nfsc ~]# showmount -a 192.168.56.10
All mount points on 192.168.56.10:
```
- Заходим в каталог и проверяем статус монтирования.
```
[root@nfsc ~]# cd /mnt/upload
[root@nfsc upload]# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=33,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=11212)
192.168.56.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.56.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.56.10)
```
- Создаем тестовый файл final_check.
- Проверяем наличие ранее созданных файлов и нового.
```
[root@nfsc upload]# touch final_check
[root@nfsc upload]# ls
check_file  client_file  final_check
```
> Если вышеуказанные проверки прошли успешно, это значит, что стенд работоспособен и готов к работе.

### Создание автоматизированного Vagrantfile

- Изменяем Vagrantfile, дополнив его 2 bash-скриптами, nfss.sh - для конфигурирования сервера и nfsc.sh - для конфигурирования клиента. 

- [Автоматизированный Vagrantfile](Vagrantfile)
- [Скрипт для конфигурирования сервера](nfss.sh)
- [Скрипт для конфигурирования клиента](nfsc.sh)

- Необходимо удалить тестовый стенд командой vagrant destroy, далее создадить его заново с помощью нового Vagrantfile.
- После этог выполнить пункты проверки работоспособности сервера и клиента.

