sudo apt install bind9
sudo mkdir /var/cache/bind/log
sudo chown bind:bind /var/cache/bind/log


touch /etc/bind/named.conf.log :
```
logging {
        channel update_debug {
                file "log/update_debug.log" versions 3 size 100k;
                severity debug;
                print-severity  yes;
                print-time      yes;
        };
        channel security_info {
                file "log/security_info.log" versions 1 size 100k;
                severity info;
                print-severity  yes;
                print-time      yes;
        };
        channel bind_log {
                file "log/bind.log" versions 3 size 1m;
                severity info;
                print-category  yes;
                print-severity  yes;
                print-time      yes;
        };
        category default { bind_log; };
        category lame-servers { null; };
        category update { update_debug; };
        category update-security { update_debug; };
        category security { security_info; };
};
```

/etc/bind/named.conf | add
```
include "/etc/bind/named.conf.log";
```

/etc/default/named | add
```
OPTIONS="-u bind -t /var/lib/bind"
```

sudo systemctl reenable named
sudo mkdir -p /var/lib/bind/{etc,dev,run/named,var/cache/bind/log}
sudo mkdir -p /var/lib/bind/usr/share
sudo cp -r /usr/share/dns/ /var/lib/bind/usr/share
sudo mknod /var/lib/bind/dev/null c 1 3
sudo mknod /var/lib/bind/dev/random c 1 8
sudo mknod /var/lib/bind/dev/urandom c 1 9
sudo chmod 660 /var/lib/bind/dev/{null,random,urandom}
sudo mv /etc/bind /var/lib/bind/etc/
sudo ln -s /var/lib/bind/etc/bind/ /etc/bind
sudo timedatectl set-timezone Europe/Moscow
sudo chown bind:bind /var/lib/bind/etc/bind/rndc.key
sudo chown bind:bind /var/lib/bind/run/named/
sudo chmod 755 /var/lib/bind/{var/cache/bind,run/named}
sudo chgrp bind /var/lib/bind/{var/cache/bind,run/named}
sudo chown -R bind:bind /var/lib/bind/var/cache/bind/log/

touch /etc/rsyslog.d/bind-chroot.conf
```
$AddUnixListenSocket  /var/lib/bind/var/cache/bind/log
```

Чекалка конфига
sudo named-checkconf


### Создание зоны своего домена

Создаем файл описания зоны
/etc/bind/named.conf.cherniak.ai-zone
```
$TTL    3h
chernyak.ay. SOA ns01 root.server 2212202400 1d 12 h 1w 3h
                NS      ns01
ns01            A       10.0.0.11
test-prom       A       10.0.0.9
```
Добавить в файл /etc/bind/named.conf.local
```
zone "cherniak.ai" {
        type master;
        file "/etc/bind/named.conf.cherniak.ai-zone";
};
```
проверить конфигурацию всех зон
`named-checkconf -z`
`sudo rndc reload`
проверка конфигурации зоны
`nmaed-checkzone chernyak.ay /etc/bind/named.conf.cherniak.ai-zone`

Добавление slave сервер для зоны
на основном  сервере в файл описание зоны добавить второй сервер
```
NS      ns02
```
На slave сервере в файл /etc/bind/named.conf.local
добавляем блок
```
zone "cherniak.ai" {
        type slave;
        file "/var/cache/bind/cherniak.ai";
        masters {
                10.0.0.11;
        };
};
```

```
sudo named-checkconf -z
sudo rndc reload
```

Команда для чтения файла зоны на slave
`sudo named-compilezone -f raw -F text -o - cherniak.ai /var/cache/bind/cherniak.ai`

### Пример конфигурации зоны обратного просмотра
В конф файл /etc/bind/named.conf.local добавит блок
```
zone "0.0.10.IN-ADDR.ARPA" {
        type master;
        file "/etc/bind/cherniak.rev";
};
```

В файл /etc/bind/cherniak.rev добавить конфигурацию
```
$TTL 3h
@ SOA ns01.cherniak.ai. root.server.cherniak.ai. 2212202401 1d 12h 1w 3h
        NS      ns01.cherniak.ai.

9       PTR     test-prom.cherniak.ai.
10      PTR     client.cherniak.ai.
11      PTR     ns01.cherniak.ai.
12      PTR     ns02.cherniak.ai.
```



Еще пример конфигурации зоны
/etc/bind/named.dns.lab
```
$ORIGIN .
$TTL 360        ; 1 hour

dns.lab IN SOA ns01.dns.lab. root.dns.lab. (
                             2212202401 ; serial
                             3600 ; refresh (1 hour)
                             600 ; retry (10 minutes)
                             86400 ;expire (1 day)
                             600 ; minimum (10 minutes)
                             )
        NS      ns01.dns.lab.
        NS      ns02.dns.lab.

$ORIGIN dns.lab.
$TTL 600
client  A       192.168.50.15
client1 A       192.168.50.15
$TTL 3600
client2 A       192.168.50.16
ns01    A       10.0.0.11
ns02    A       10.0.0.12
web1    CNAME   client
web2    CNAME   client2
```

refresh - время спустя которое slave будет пытаться получить информацию с master
retry - если слейву не удастся получить инфо с мастера, то будет повторять запрос каждые 10 минут 
expire - если слейв в течении дня не получит инфо с мастера, то перестанет обслуживать запросы для данной зоны
minimum - время жизни записи (в кеше )

Доп. настройки
/etc/bind/named.conf
```
key "rndc-key" {
        algorithm hmac-sha256;
        secret "S6TliiZ55tBpJRFRNps4vZY8X3YdI3zxboxycGnlaII=";
};

controls {
        inet 10.0.0.11  allow { 10.0.0.9; } keys { "rndc-key"; };
        inet 127.0.0.1  allow { 127.0.0.1; } keys { "rndc-key"; };
};
```
! Если указанные настройки указать в /etc/bind/named.conf.options будет ошибка
Для управления сервером с клиента, нужно создать на клиенте конф. файл  rndc.conf
```
key "rndc-key" {
        algorithm hmac-sha256;
        secret "S6TliiZ55tBpJRFRNps4vZY8X3YdI3zxboxycGnlaII=";
};
options {
        default-key "rndc-key";
        default-server 10.0.0.11;
};
```
Управление с помощью rndc удаленно 
`sudo rndc -c rndc.conf reload`

Полезная утиль rndc-confgen, пример вывода
```
# Start of rndc.conf
key "rndc-key" {
        algorithm hmac-sha256;
        secret "1vgUXstWeH7SpYQbTIXeab7bKam/pBZl2e4rKpPSCz4=";
};

options {
        default-key "rndc-key";
        default-server 127.0.0.1;
        default-port 953;
};
# End of rndc.conf

# Use with the following in named.conf, adjusting the allow list as needed:
# key "rndc-key" {
#       algorithm hmac-sha256;
#       secret "1vgUXstWeH7SpYQbTIXeab7bKam/pBZl2e4rKpPSCz4=";
# };
#
# controls {
#       inet 127.0.0.1 port 953
#               allow { 127.0.0.1; } keys { "rndc-key"; };
# };
# End of named.conf
```
