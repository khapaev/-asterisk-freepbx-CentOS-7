# Подготовка системы
Прежде чем мы начнем устанавливать asterisk, нам надо выполнить целый ряд подготовительных действий. Первым делом отключаем selinux. Для этого открываем файл:
```
mcedit /etc/sysconfig/selinux
```
и устанавливаем значение:
```
SELINUX=disabled
```
После этого применяем настройку без перезагрузки сервера:
```
setenforce 0
```
Дальше обновляем систему и ставим пакеты Development Tools:
```
yum update
yum groupinstall core base "Development Tools"
```
# Установка Mariadb
Подключаем репозиторий со свежей версией MariaDB. Для этого создаем файл /etc/yum.repos.d/MariaDB.repo следующего содержания:
```
# MariaDB 10.3 CentOS repository list - created 2019-04-01 09:11 UTC
# http://downloads.mariadb.org/mariadb/repositories/
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.3/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```
Устанавливаем MariaDB:
```
yum install MariaDB-server MariaDB-client MariaDB-shared
```
Запускаем mariadb и добавляем в автозагрузку:
```
systemctl start mariadb
systemctl enable mariadb
```
# Настройка Web сервера
Подключаем репозиторий epel:
```
yum install epel-release
```
Подключаем remi репозиторий для centos 7:
```
rpm -Uhv http://rpms.remirepo.net/enterprise/remi-release-7.rpm
```
Ставим пакет yum-utils:
```
yum install yum-utils
```
Активируем репу remi-php71:
```
yum-config-manager --enable remi-php71
```
Устанавливаем необходимые пакеты для работы сервера voip:
```
yum install wget php php-pear php-cgi php-common php-curl php-mbstring php-gd php-mysql php-gettext php-bcmath php-zip php-xml php-imap php-json php-process php-snmp
```
Далее установим httpd:
```
yum install httpd
```
Теперь нам нужно изменить некоторые параметры httpd - запустить его от пользователя asterisk и включить опцию AllowOverride. Это можно сделать руками в файле /etc/httpd/conf/httpd.conf, либо автоматически с помощью sed:
```
sed -i 's/^\(User\|Group\).*/\1 asterisk/' /etc/httpd/conf/httpd.conf
sed -i 's/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf
```
Мы просто выставили следующие параметры:
- [x] User asterisk
- [x] Group asterisk
- [x] AllowOverride All

В /etc/php.ini устанавливаем параметр:
```
upload_max_filesize = 120M
```
Или через sed:
```
sed -i 's/\(^upload_max_filesize = \).*/\120M/' /etc/php.ini
```
# Установка NodeJS
Подключаем репозиторий NodeJS с помощью скрипта автоматизации от разработчика:
```
curl -sL https://rpm.nodesource.com/setup_10.x | bash -
```
Обновляем кэш yum:
```
yum clean all && sudo yum makecache fast
```
Устанавливаем NodeJS и некоторые зависимости:
```
yum install gcc-c++ make nodejs
```
Проверяем на всякий случай версию:
```
node -v
```
Если видите номер версии, значит установка прошла успешно.
# Установка Asterisk
Скачиваем архив последней версии Asterisk с официального сайта:
```
cd ~ && wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-16-current.tar.gz
```
Распаковываем исходники:
```
tar zxvf asterisk-*.tar.gz
```
Переходим в директорию с исходниками:
```
cd asterisk*
```
Выполняем скрипт для установки пакетов с зависимостями для asterisk:
```
contrib/scripts/install_prereq install
```
Запускаем скрипт для скачивания исходников для работы с mp3:
```
contrib/scripts/get_mp3_source.sh
```
Настраиваем конфигурацию:
```
./configure --with-pjproject-bundled --with-jansson-bundled --with-crypto --with-ssl=ssl --with-srtp
```
Можно запускать установку asterisk:
```
make && make install && make config && make samples && ldconfig
```
Настроим запуск астериск от системного пользователя asterisk. Для этого редактируем скрипт запуска /usr/sbin/safe_asterisk, установив параметр:
```
ASTARGS="-U asterisk"
```
Или через sed:
```
sed -i 's/ASTARGS=""/ASTARGS="-U asterisk"/g' /usr/sbin/safe_asterisk
```
Создадим этого пользователя и назначим нужные права на каталоги:
```
useradd -m asterisk
chown asterisk.asterisk /var/run/asterisk
chown -R asterisk.asterisk /etc/asterisk
chown -R asterisk.asterisk /var/{lib,log,spool}/asterisk
chown -R asterisk.asterisk /usr/lib/asterisk
```
Запускаем Asterisk:
```
systemctl start asterisk
```
Проверьте сразу, что он запустился:
```
systemctl status asterisk
```
Если будут ошибки:
```
radcli: rc_read_config: rc_read_config: can't open /etc/radiusclient-ng/radiusclient.conf: No such file or directory
```
То редактируем конфигурационные файлы asterisk, заменив в некоторых строках пути на правильные:
```
sed -i 's";\[radius\]"\[radius\]"g' /etc/asterisk/cdr.conf
sed -i 's";radiuscfg => /usr/local/etc/radiusclient-ng/radiusclient.conf"radiuscfg => /etc/radcli/radiusclient.conf"g' /etc/asterisk/cdr.conf
sed -i 's";radiuscfg => /usr/local/etc/radiusclient-ng/radiusclient.conf"radiuscfg => /etc/radcli/radiusclient.conf"g' /etc/asterisk/cel.conf
```
После этого перезапускаем asterisk, ошибок быть не должно.
# Установка и настройка Freepbx
Скачиваем последнюю версию Freepbx с сайта разработчика:
```
cd ~ && wget http://mirror.freepbx.org/modules/packages/freepbx/freepbx-15.0-latest.tgz
```
Распаковываем исходники:
```
tar xvfz freepbx-*.tgz
```
Переходим в каталог freepbx и запускаем скрипт проверки запуска asterisk:
```
cd freepbx && ./start_asterisk start
```
Если не получили ошибок, то запускаем установку непосредственно FreePBX:
```
./install -n
```
Если получили ошибку php:
```
PHP Fatal error: Uncaught Error: Call to a member function connected() on null in /root/freepbx/amp_conf/htdocs/admin/libraries/BMO/Framework.class.php:180
```
Запускаем установку еще раз:
```
./install -n
```
Она должна пройти без ошибок, но freepbx не будет работать корректно, так как не сможет подключиться к asterisk. После повторной установки надо открыть конфиг:
```
/etc/asterisk/manager.conf
```
И в конце строки:
```
#include manager_additional.conf
#include manager_custom.conf
```
Заменить на:
```
;include manager_additional.conf
;include manager_custom.conf
```
И убедиться, что указан параметр secret с паролем amp111. Если это не так, отредактируйте строку:
```
secret = amp111
```
Если этого параметра вообще нет, то добавить в секцию [admin]. После этого надо еще раз запустить установку freepbx, в третий раз. После этого ошибок быть не должно и freepbx будет корректно работать.

Если все прошло без ошибок, то можно проверять работу Freepbx. Но перед этим отключим Firewall, если он у вас работает:
```
systemctl stop firewalld && systemctl disable firewalld
```
Нам нужно открыть 80-й порт, чтобы мы смогли работать с веб интерфейсом:
```
iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
```
Для полноценной работы астериск, нужно открыть следующие порты:
```
iptables -A INPUT -p udp -m udp --dport 5060 -j ACCEPT
iptables -A INPUT -p udp -m udp --dport 5061 -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 5060 -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 5061 -j ACCEPT
iptables -A INPUT -p udp -m udp --dport 4569 -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 5038 -j ACCEPT
iptables -A INPUT -p udp -m udp --dport 5038 -j ACCEPT
iptables -A INPUT -p udp -m udp --dport 10000:20000 -j ACCEPT
```
Теперь можно запустить httpd:
```
systemctl start httpd
```
Добавляем его в автозагрузку:
```
systemctl enable httpd
```
На этом установка закончена Freepbx. Можно зайти браузером на страницу с ip адресом сервера. Открывается начальная страница freepbx, где нам предлагается создать нового пользователя. Создаем пользователя и заходим в web интерфейс управления астериском.
