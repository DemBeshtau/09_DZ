# Инициализация системы. Systemd. #
1. Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого<br/>
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
1. Подготовлен файл со значениями переменных сервиса в директории /etc/sysconfig
