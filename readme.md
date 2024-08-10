## Аппаратный прокси на базе Orange Pi Zero 2 и Debian 12

![Orange Pi Zero 2](http://www.orangepi.org/img/orange-pi-zero2-banner-img.png)

Для работы нужны следующие пакеты
> hostapd
> dnsmasq
> redsocks
> ciadpi

Ethernet подключение выполняется к оператору связи, к  Wi-Fi Хотспоту подключаются клиенты.

###  Установка и настройка hostapd

```no-highlight
sudo apt-get install hostapd
```
Редактируем файл /etc/default/hostapd.conf. 

```no-highlight
sudo nano /etc/default/hostapd.conf
```

Раскомментировать строку

```no-highlight
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```
Остановим сервис hostapd

```no-highlight
service hostapd stop
```
Конфигурация Wi-Fi Хотспота hostapd

```no-highlight
interface=wlan0
driver=nl80211
country_code=RU
ssid=INTERNETNODPI          #Wi-Fi AP SSID
hw_mode=a
channel=36
ieee80211n=1
ieee80211ac=1
vht_capab=[VHT80][SHORT-GI-80]
wpa=2
wpa_passphrase=NoDPI12345  #PASSWORD 
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
auth_algs=1
macaddr_acl=0

```
В данной конфигурации реализована Wi-Fi точка доступа 5ГГц, стандарт 802.11n, 802.11ac, частотный канал 36,
wlan0 - беспроводной сетевой интерфейс, определяется с помощью команды:

```no-highlight
ip a
```
Конфигурация сетевого интерфейса 

```no-highlight
sudo nano /etc/network/interfaces
```

Содержимое конфигурационного файла

```no-highlight
source /etc/network/interfaces.d/*
# Network is managed by Network manager
auto lo
iface lo inet loopback
auto wlan0
iface wlan0 inet static
address 192.168.50.1
netmask 255.255.255.0
gateway 192.168.50.1
```
Беспроводному интерфейсу назначен IP адрес 192.168.50.1

Настройка DHCP сервера для Wi-Fi Хотспота, установим dnsmasq

установка dnsmasq

```no-highlight
sudo apt-get install dnsmasq
```
Остановим сервис dnsmasq

```no-highlight
service dnsmasq stop
```

Редактируем файл конфигурации 

```no-highlight
sudo nano /etc/dnsmasq.conf
```

Содержимое файла конфигурации 

```no-highlight
interface=wlan0                                 # Use interface wlan0
listen-address=192.168.50.1                     # Explicitly specify the address to>
bind-interfaces                                 # Bind to the interface to make sur>
server=8.8.8.8                                  # Forward DNS requests to Google DNS
domain-needed                                   # Don't forward short names
bogus-priv                                      # Never forward addresses
dhcp-range=192.168.50.50,192.168.50.150,12h     # IP range DHCP
```

Установка и настройка redsocks

```no-highlight
sudo apt-get install redsocks
```

Редактируем конфигурационный файл

```no-highlight
sudo nano /etc/redsocks.conf
```
Содержание файла конфигурации

```no-highlight
base {
    log_debug = off;
    log_info = on;
    log = "file:/var/log/redsocks.log";
    daemon = on;
    redirector = iptables;
}

redsocks {
    local_ip = 0.0.0.0;
    local_port = 12345;
    ip = 127.0.0.1;   
    port = 1080;    
    type = socks5;
}
```

Настройка iptables для перенаправления трафика

```no-highlight
sudo iptables -t nat -N REDSOCKS
sudo iptables -t nat -A REDSOCKS -d 192.168.50.0/24 -j RETURN        # Исключаем локальную сеть
sudo iptables -t nat -A REDSOCKS -p tcp -j REDIRECT --to-ports 12345
sudo iptables -t nat -A PREROUTING -i wlan0 -p tcp -j REDSOCKS
```

Добавляем правила iptables для автозагрузки при запуске системы

Сохранение текущих правил iptables в файл iptables.rules. Создаем файл и ограничиваем к нему доступ
```no-highlight
sudo touch /etc/iptables.rules
sudo chmod 640 /etc/iptables.rules
```

Сохраняем текущие правила iptables в файл

```no-highlight
sudo iptables-save | sudo tee /etc/iptables.rules
```
Автозагрузка сохраненных правил

В /etc/network/if-pre-up.d/ создаём файл iptables, со следующим содержимым

```no-highlight
#!/bin/sh
iptables-restore < /etc/iptables.rules
exit 0
```

Делаем созданный сценарий исполнимым

```no-highlight
sudo chmod +x  /etc/network/if-pre-up.d/iptables
```
Создаем сервис ciadpi (DPI bypass) под не root пользователем, например orangepi

Клонируем репозитарий 

```no-highlight
git clone https://github.com/VGCH/byedpi_orange_pi
```
Переходим в папку с репозиторием и собираем 

```no-highlight
сd byedpi_orange_pi
make
```

Создаем скрипт сервиса byedpi_orange_pi для автозапуска

```no-highlight
nano /etc/systemd/system/byedpi_orange_pi.service
```
И сохраняем следующее содержимое 

```no-highlight
[Unit]
Description=dpi port 1080
After=network.target

[Service]
User=orangepi
Group=orangepi
ExecStart=/home/orangepi/byedpi_orange_pi/ciadpi --disorder 1 --auto=torst --tlsrec 1+s

[Install]
WantedBy=multi-user.target
```
Добавляем скрипт в автозагрузку 

```no-highlight
systemctl enable byedpi_orange_pi
```
Перезагружаем систему и проверяем работу

```no-highlight
sudo shutdown -r now
```



## Ниже описание аргументов от автора ByeDPI

Implementation of some DPI bypass methods.
The program is a local SOCKS proxy server.

Usage example:
ciadpi --disorder 1 --auto=torst --tlsrec 1+s
ciadpi --fake -1 --ttl 8

------
### Описание аргументов

-i, --ip <ip>
    Прослушиваемый IP, по умолчанию 0.0.0.0

-p, --port <num>
    Прослушиваемый порт, по умолчанию 1080

-c, --max-conn <count>
    Максимальное количество клиентских подключений, по умолчанию 512

-I  --conn-ip <ip>
    Адрес, к которому будут привязаны исходящие соединения, по умолчанию ::
    При указании IPv4 адреса запросы на IPv6 будут отклоняться

-b, --buf-size <size>
    Максимальный размер данных, получаемых и отправляемых за один вызов recv/send
    Размер указывается в байтах, по умолчанию равен 16384

-g, --def-ttl <num>
    Значение TTL для всех исходящий соединений
    Может быть полезен для обхода обнаружения нестандартного/уменьшенного TTL

-N, --no-domain
    Отбрасывать запросы, если в качестве адреса указан домен
    Т.к. резолвинг выполняется синхронно, то он может замедлить или даже заморозить работу

-U, --no-udp
    Не проксировать UDP
    
-F, --tfo
    Включает TCP Fast Open
    Если сервер его поддерживает, то первый пакет будет отправлен сразу вместе с SYN
    Поддерживается только в Linux (4.11+)
    
-A, --auto[=t,r,c,s,a,n]
    Автоматический режим
    Если произошло событие, похожее на блокировку или поломку,
    то будут применены параметры обхода, следующие за данной опцией
    Возможные события:
        torst   : Вышло время ожидания или сервер сбросил подключение после первого запроса
        redirect: HTTP Redirect с Location, домен которого не совпадает с исходящим
        cl_err  : HTTP ответ, код которого равен 40x, но не 429
        sid_inv : session_id в TLS ServerHello и ClientHello не совпадают
        alert   : TLS Error Alert в ответе
        none    : Предыдущая группа пропущена, например из-за ограничения по доменам или протоколам
    
-u, --cache-ttl <sec>
    Время жизни значения в кеше, по умолчанию 100800 (28 часов)
    
-T, --timeout <sec>
    Таймаут ожидания первого ответа от сервера в секундах
    В Linux переводится в миллисекунды, поэтому можно указать дробное число
    
-K, --proto <t,h,u>
    Белый список протоколов: tls,http,udp
    
-H, --hosts <file|:string>
    Ограничить область действия параметров списком доменов
    Домены должны быть разделены новой строкой или пробелом
    
-V, --pf <port[-portr]>
    Ограничитель по портам
    
-s, --split <n[+s]>
    Разбить запрос по указанному смещению
    После числа можно добавить флаг:
        +s: добавить смещение SNI
        +h: добавить смещение Host
    Можно указывать несколько раз, чтобы разбить запрос по нескольким позициям
    При указании отрицательного значения к нему прибавляется размер пакета
    
-d, --disorder <n[+s]>
    Подобен --split, но части отправляются в обратном порядке
    ! Поведение в Windows отлично: сначала отправляется лишь часть, но затем целый запрос
    
-o, --oob <n[+s]>
    Подобен --split, но после части отсылается один или несколько байт OOB данных
    
-f, --fake <n[+s]>
    Подобен --disorder, только перед отправкой первого куска отправляется часть поддельного
    Количество байт отправляемого из фейка равно рамеру разбиваемой части
 
-t, --ttl <num>
    TTL для поддельного пакета, по умолчанию 8
    Необходимо подобрать такое значение, чтобы пакет не дошел до сервера, но был обработан DPI

-k, --ip-opt[=file|:str]
    Установить опции для фейкового IP пакета
    Существенно снизит вероятность, что пакет дойдет до сервера
    Стоит учесть, что до DPI он также может не дойти
    В Windows поддержка может быть отключена
    
-S, --md5sig
    Установить опцию TCP MD5 Signature для фейкового пакета
    Большинство серверов (в основном на Linux) отбрасывают пакеты с данной опцией
    Поддерживается только в Linux, может быть выключен в некоторых сборках ядра (< 3.9, Android)
    
-l, --fake-data <file|:str>
    Указать свои поддельные пакеты, вместо дефолтных

-e, --oob-data <file|:str>
    Данные, отсылаемые вне основного потока, по умолчанию один байт 'a'
    ! При размере более одного байта может работать нестабильно
    
-n, --tls-sni <str>
    Изменить SNI в fake пакете на указанный

-M, --mod-http <h[,d,r]>
    Всякие манипуляции с HTTP пакетом, можно комбинировать
    hcsmix:
        "Host: name" -> "hOsT: name"
    dcsmix:
        "Host: name" -> "Host: NaMe"
    rmspace:
        "Host: name" -> "Host:name\t"

-r, --tlsrec <n[+s]>
    Разделить ClientHello на отдельные записи по указанному смещению
    Можно указывать несколько раз  

-a, --udp-fake <count>
    Количество фейковых UDP пакетов

### Подробнее

------
--split  
Разбивает запрос на части. Пример на запросе в 30 байт:
- Параметры: --split 3 --split 7
- Порядок отправки: 1-3, 3-7, 7-30  

------
--disorder  
Часть, попадающая под disorder, будет отправлена с TTL=1, т.е. фактически не будет никуда доставлена.
ОС узнает об этом лишь после отсылки последующей части, когда сервер сообщит о потере с помощью SACK.
Системе придется отослать предыдущий пакет заново, тем самым нарушив обычный порядок.
- Параметры: --disorder 7
- Порядок отправки: 7-30, 1-7  

Вышесказанное распространяется только на Linux.
В Windows выполняется полная ретрансмиссия:
- Параметры: --disorder 7
- Порядок отправки: 7-30, 1-30

Поэтому желательно использовать ещё и split:  
- Параметры: --split 7 --disorder 23
- Порядок отправки: 1-7, 23-30, 7-30

На практике оптимально использовать:  
Linux: --disorder 1  
Windows: --split 1+s --disorder 3+s  

------
--fake  
- Параметры: --fake 7
- Порядок отправки: 1-7 фейк, 7-30 оригинал, 1-7 оригинал

Данные в первой части запроса заменяются на поддельные.  
Эта часть должна пройти через DPI, но не дойти до сервера.
А раз часть не дойдет, то ОС отправит ее снова, тем самым изменив порядок подобно disorder.
Для того, чтобы фейк не дошел до сервера, есть опции ttl, ip-opt и md5sig.  

TTL необходимо подбирать такой, чтобы пакет прошел через все DPI, но не дошел до сервера.  
Для Linux есть md5sig. Он устанавливает опцию TCP MD5 Signature, что не дает пакету быть принятым многими серверами.
К сожалению, md5sig работает не во всех сборках.

Для Windows есть еще один способ избежать обработки фейка сервером.
Это комбинирование fake с disorder:
- Параметры: --disorder 1 --fake 7
- Порядок отправки: 2-7 фейк, 7-30 оригинал, 1-30 оригинал

Если поддельный пакет и дойдет до сервера, то он будет перезаписан из-за полной ретрансмисси.

На практике оптимально использовать:  
Linux: --fake -1 --md5sig  
Windows: --disorder 1 --fake -1  

------
--oob  
TCP может отсылать данные вне основного потока, используя флаг URG, однако лишь 1 байт в пакете.
Все данные в таком пакете будут доставлены приложению, кроме последнего байта, который и является внеканальным:
- Параметры: --oob 3
- Отправка: 1-4 с флагом URG (1-3 данные запроса + 4-й байт, который будет усечен), 3-30

Этот байт желательно помещать в SNI: --oob 3+s  

------
--tlsrec  
Одну TLS запись можно разбить на несколько, немного переделав заголовок.
На месте разбиения вставляется новый заголовок, увеличивая размер запроса на 5 байт.  
Этот заголовок можно поместить в середину SNI, не давая возможность DPI правильно его прочитать: 
--tlsrec 3+s

Хоть tlsrec и oob запутывают DPI, они также могут запутать всякие мидлбоксы,
которые не поддерживают полноценный стек TCP/TLS.  
Из-за этого их следует использовать вместе с --auto:  
--auto=torst --timeout 3 --tlsrec 3+s  
В примере tlsrec будет применяться лишь в случаях, когда сброшено подключение или вышел таймаут, т.е. когда, скорее всего, произошла блокировка.  
Можно наоборот - отменять tlsrec, если сервер сбрасывает подключение или откидывает пакет:  
--tlsrec 3+s --auto=torst --timeout 3  

------
--auto, --hosts  
Параметр auto делит опции на группы.
Для каждого запроса они обходятся слева на право.
Сначала проверяется триггер, указанный в auto, затем proto и hosts.  
Можно указывать несколько групп опций, раделяя их данным параметром.
Параметры, которые можно вынести в отдельную группу:  
proto, hosts, pf, split, disorder, oob, fake, ttl, ip-opt, md5sig, fake-data, mod-http, tlsrec, udp-fake  

Примеры:  
--fake -1 --ttl 10 --auto=alert,sid_inv --fake -1 --ttl 5  
По умолчанию использовать fake с ttl=10, в случае ошибки использовать fake с ttl=5

--hosts list.txt --disorder 3 --auto=none  
Применять запутывание только для доменов из list.txt

--hosts list.txt --auto=none --disorder 3  
Не применять запутывание для доменов из list.txt

--auto=torst --hosts list.txt --disorder 3  
По умолчанию ничего не делать, использовать disorder при условии, что произошла блокировка и домен входит в list.txt.

--proto=http,tls --disorder 3 --auto=none  
Запутывать только HTTP и TLS

--proto=http --fake -1 --fake-data=':GET /...' --auto=none --fake -1  
Переопределить фейковый пакет для HTTP

------
### Сборка
Для сборки понадобится: 
make, gcc/clang для Linux, mingw для Windows  

Linux: make  
Windows: make windows CC=x86_64-w64-mingw32-gcc

------
### Дополнительная информация о DPI, источники идей  
https://github.com/bol-van/zapret/blob/master/docs/readme.txt  
https://geneva.cs.umd.edu/papers/geneva_ccs19.pdf  
https://habr.com/ru/post/335436  
