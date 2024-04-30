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
[root@inittest ~]# systemctl start watchlog.timer
[root@inittest ~]# systemctl status watchlog.timer
● watchlog.timer - Run watchlog script every 10 seconds
   Loaded: loaded (/etc/systemd/system/watchlog.timer; disabled; vendor preset: disabled)
   Active: active (waiting) since Tue 2024-04-30 18:24:12 UTC; 8s ago
  Trigger: Tue 2024-04-30 18:24:22 UTC; 1s left

Apr 30 18:24:12 inittest systemd[1]: Started Run watchlog script every 10 seconds.
```
1.7. Проверка работы сервиса:
```shell
[root@inittest ~]# tail -f /var/log/messages 
Apr 30 18:24:12 inittest systemd[1]: Started Run watchlog script every 10 seconds.
Apr 30 18:24:12 inittest systemd[1]: Starting Testing watchlog service...
Apr 30 18:24:12 inittest root[6084]: Tue Apr 30 18:24:12 UTC 2024: I found word!
Apr 30 18:24:12 inittest systemd[1]: watchlog.service: Succeeded.
Apr 30 18:24:12 inittest systemd[1]: Started Testing watchlog service.
Apr 30 18:24:22 inittest systemd[1]: Starting Testing watchlog service...
Apr 30 18:24:22 inittest root[6092]: Tue Apr 30 18:24:22 UTC 2024: I found word!
Apr 30 18:24:22 inittest systemd[1]: watchlog.service: Succeeded.
Apr 30 18:24:22 inittest systemd[1]: Started Testing watchlog service.
Apr 30 18:24:32 inittest systemd[1]: Starting Testing watchlog service...
Apr 30 18:24:32 inittest root[6097]: Tue Apr 30 18:24:32 UTC 2024: I found word!
Apr 30 18:24:32 inittest systemd[1]: watchlog.service: Succeeded.
Apr 30 18:24:32 inittest systemd[1]: Started Testing watchlog service.
Apr 30 18:24:42 inittest systemd[1]: Starting Testing watchlog service...
Apr 30 18:24:42 inittest root[6102]: Tue Apr 30 18:24:42 UTC 2024: I found word!
Apr 30 18:24:42 inittest systemd[1]: watchlog.service: Succeeded.
Apr 30 18:24:42 inittest systemd[1]: Started Testing watchlog service.
Apr 30 18:24:52 inittest systemd[1]: Starting Testing watchlog service...
Apr 30 18:24:52 inittest root[6108]: Tue Apr 30 18:24:52 UTC 2024: I found word!
Apr 30 18:24:52 inittest systemd[1]: watchlog.service: Succeeded.
Apr 30 18:24:52 inittest systemd[1]: Started Testing watchlog service.
Apr 30 18:25:03 inittest systemd[1]: Starting Testing watchlog service...
Apr 30 18:25:03 inittest root[6113]: Tue Apr 30 18:25:03 UTC 2024: I found word!
Apr 30 18:25:03 inittest systemd[1]: watchlog.service: Succeeded.
Apr 30 18:25:03 inittest systemd[1]: Started Testing watchlog service.
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
2.2. Раскомментирование переменных в /etc/sysconfig/spawn-fcgi:
```shell
[root@inittest ~]# nano /etc/sysconfig/spawn-fcgi
...
[root@inittest ~]# cat /etc/sysconfig/spawn-fcgi 
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"
```
2.3. Подготовка юнит файла для сервиса spawn-fcgi:
```shell
[root@inittest ~]# nano /etc/systemd/system/spawn-fcgi.service
...
[root@inittest ~]# cat /etc/systemd/system/spawn-fcgi.service 
[Unit]
Description=spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
```
2.4. Запуск сервиса spawn-fcgi и проверка его работоспособности:
```shell
[root@inittest ~]# systemctl start spawn-fcgi
[root@inittest ~]# systemctl status spawn-fcgi
● spawn-fcgi.service - spawn-fcgi startup service by Otus
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-04-30 17:38:06 UTC; 21s ago
 Main PID: 4192 (php-cgi)
    Tasks: 33 (limit: 2697)
   Memory: 27.7M
   CGroup: /system.slice/spawn-fcgi.service
           ├─4192 /usr/bin/php-cgi
           ├─4193 /usr/bin/php-cgi
           ├─4194 /usr/bin/php-cgi
           ├─4195 /usr/bin/php-cgi
           ├─4196 /usr/bin/php-cgi
           ├─4197 /usr/bin/php-cgi
           ├─4198 /usr/bin/php-cgi
           ...
```
#### 3. Дополнение юнит файла httpd (он же apache2) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами. ####
3.1. Подготовка юнит файла для запуска нескольких экземпляров сервиса с использованием шаблона в <br/>
конфигурации файла окружения (/usr/lib/systemd/system/httpd.service ):
```shell
[root@inittest ~]# nano /etc/systemd/system/httpd.service
...
[root@inittest ~]# cat /etc/systemd/system/httpd.service
[Unit]
Description=The Apache HTTP Server
Wants=httpd-init.service
After=network.target remote-fs.target nss-lookup.target httpd-init.service
Documentation=man:httpd.service(8)

[Service]
Type=notify
Environment=LANG=C
EnvironmentFile=/etc/sysconfig/httpd-%I
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
# Send SIGWINCH for graceful stop
KillSignal=SIGWINCH
KillMode=mixed
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
3.2. Создание двух файлов окружения с опциями запуска веб-сервера с необходимыми конфигурационными<br/>
файлами:
```shell
[root@inittest ~]# cat /etc/sysconfig/httpd-first 
OPTIONS=-f conf/first.conf
[root@inittest ~]# cat /etc/sysconfig/httpd-second 
OPTIONS=-f conf/second.conf
```
3.3. Создание конфигурационных файлов httpd в /etc/httpd/conf:
```shell
[root@inittest ~]# cat /etc/httpd/conf/first.conf 
...
ServerRoot "/etc/httpd"
PidFile /var/run/httpd-first.pid
Listen 80
...
[root@inittest ~]# cat /etc/httpd/conf/second.conf
...
ServerRoot "/etc/httpd"
PidFile /var/run/httpd-second.pid
Listen 8080
... 
```
3.4. Запуск двух инстансов httpd:
```shell
[root@inittest ~]# ss -ntlpu | grep httpd
tcp   LISTEN 0      511          0.0.0.0:8080      0.0.0.0:*    users:(("httpd",pid=5701,fd=3),("httpd",pid=5700,fd=3),("httpd",pid=5699,fd=3),("httpd",pid=5698,fd=3),("httpd",pid=5696,fd=3))
tcp   LISTEN 0      511          0.0.0.0:80        0.0.0.0:*    users:(("httpd",pid=5473,fd=3),("httpd",pid=5472,fd=3),("httpd",pid=5471,fd=3),("httpd",pid=5470,fd=3),("httpd",pid=5468,fd=3))
```
