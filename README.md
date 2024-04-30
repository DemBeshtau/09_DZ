# Инициализация системы. Systemd. #
1. Написать сервис, который будет раз в 10 секунд мониторить лог на предмет наличия ключевого<br/>
слова. Файл и слово задаются в /etc/sysconf;<br/>
2. Установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться<br/> 
так же: spawn-fcgi);
3. Дополнить unit-файл httpd (он же apache2) возможностью запустить несколько инстансов сервера<br/> 
с разными конфигурационными файлами.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.14 и образа<br/> 
&ensp;&ensp;CentOS 8 Stream (generic/centos8s) версии 4.3.12.
### Ход решения ###
#### 1. Подготовка и исполнение сервиса, предназначенного для мониторинга лога ####
1.1. Подготовлен файл со значениями переменных сервиса в директории /etc/sysconfig:
```shell
[root@inittest ~]# cat /etc/sysconfig/watchlog
# Configuration file for watchlog service

#File and keyword
WORD="ALERT"
LOG=/var/log/watchlog.log
```
1.2. Создан файл /var/log/watchlog.log, в котором содежатся произвольные строки с ключевым словом "ALERT":
```shell
[root@inittest ~]# cat /var/log/watchlog.log
Test string, test string, test string ALERT
Warning message, warning messageALERT
Attention, attention, attention ALERT
```
1.3. Написан скрипт, выводящий в системный журнал сообщение о нахождении строк, содержащих ключевое слово:
```shell
[root@inittest ~]# cat /opt/watchlog.sh
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word!"
else
exit 0
fi
```
1.4. Создан юнит для сервиса:
```shell
[root@inittest ~]# cat /etc/systemd/system/watchlog.service 
[Unit]
Description=Testing watchlog service
Wants=watchlog.timer

[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG	

[Install]
WantedBy=multi-user.target
```
1.5. Создан юнит для таймера:
```shell
[root@inittest ~]# cat /etc/systemd/system/watchlog.timer
[Unit]
Description=Run watchlog script every 10 seconds
Requires=watchlog.service

[Timer]
AccuracySec=1sec
OnUnitActiveSec=10
Unit=watchlog.service

[Install]
WantedBy=timers.target
```
1.6. Старт сервиса и проверка его состояния:
```shell
[root@inittest ~]# systemctl start watchlog.service
[root@inittest ~]# systemctl status watchlog.service
● watchlog.service - Testing watchlog service
   Loaded: loaded (/etc/systemd/system/watchlog.service; disabled; vendor preset: disabled)
   Active: inactive (dead) since Tue 2024-04-30 17:15:56 UTC; 6s ago
  Process: 3502 ExecStart=/opt/watchlog.sh $WORD $LOG (code=exited, status=0/SUCCESS)
 Main PID: 3502 (code=exited, status=0/SUCCESS)

Apr 30 17:15:56 inittest systemd[1]: Starting Testing watchlog service...
Apr 30 17:15:56 inittest systemd[1]: watchlog.service: Succeeded.
Apr 30 17:15:56 inittest systemd[1]: Started Testing watchlog service.
```
1.7. Проверка работы сервиса:
```shell
[root@inittest ~]# tail -f /var/log/messages 
Apr 30 17:19:13 inittest systemd[1]: watchlog.service: Succeeded.
Apr 30 17:19:13 inittest systemd[1]: Started Testing watchlog service.
Apr 30 17:19:24 inittest systemd[1]: Starting Testing watchlog service...
Apr 30 17:19:24 inittest root[3606]: Tue Apr 30 17:19:24 UTC 2024: I found word!
Apr 30 17:19:24 inittest systemd[1]: watchlog.service: Succeeded.
Apr 30 17:19:24 inittest systemd[1]: Started Testing watchlog service.
Apr 30 17:19:35 inittest systemd[1]: Starting Testing watchlog service...
Apr 30 17:19:35 inittest root[3611]: Tue Apr 30 17:19:35 UTC 2024: I found word!
Apr 30 17:19:35 inittest systemd[1]: watchlog.service: Succeeded.
Apr 30 17:19:35 inittest systemd[1]: Started Testing watchlog service.
Apr 30 17:19:46 inittest systemd[1]: Starting Testing watchlog service...
Apr 30 17:19:46 inittest root[3619]: Tue Apr 30 17:19:46 UTC 2024: I found word!
Apr 30 17:19:46 inittest systemd[1]: watchlog.service: Succeeded.
Apr 30 17:19:46 inittest systemd[1]: Started Testing watchlog service.
Apr 30 17:19:57 inittest systemd[1]: Starting Testing watchlog service...
Apr 30 17:19:57 inittest root[3624]: Tue Apr 30 17:19:57 UTC 2024: I found word!
Apr 30 17:19:57 inittest systemd[1]: watchlog.service: Succeeded.
Apr 30 17:19:57 inittest systemd[1]: Started Testing watchlog service.
```
#### 2. Установка spawn-fcgi и переписывание init-скрипта на unit-файл ####
2.1. Установка spawn-fcgi и необходимых для него пакетов:
```shell
[root@inittest ~]# yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
[root@inittest ~]# yum list installed epel-release
Installed Packages
epel-release.noarch                                                8-19.el8                                                 @epel

[root@inittest ~]# yum list installed spawn-fcgi
Installed Packages
spawn-fcgi.x86_64                                               1.6.3-17.el8                                                @epel
[root@inittest ~]# yum list installed php
Installed Packages
php.x86_64                                    7.2.24-1.module_el8.2.0+313+b04d0a66                                     @appstream
[root@inittest ~]# yum list installed php-cli
Installed Packages
php-cli.x86_64                                  7.2.24-1.module_el8.2.0+313+b04d0a66                                   @appstream
[root@inittest ~]# yum list installed mod_fcgid
Installed Packages
mod_fcgid.x86_64                                             2.3.9-17.el8                                              @appstream
[root@inittest ~]# yum list installed httpd
Installed Packages
httpd.x86_64                                     2.4.37-64.module_el8+965+1ad5c49d                                     @appstream
```
2.2. 
