<p align="center" background="black"><img src="../img/unode-logo.svg" width="450"></p>

<p align="center">
</p>

Привет, комьюнити! Мы хотим поделиться с вами инструкцией, как развернуть ноду Minter на Centos. 

В инструкции описаны только базовые вещи, это усредненный рабочий вариант без дополнительных скриптов защиты и подключения дополнительных инструментов. О них кратко — в рекомендациях.

В инструкции все конфиги и команды приведены для оригинального образа Centos 7. Инструкция при желании легко адаптируется под другие ОС семейства Linux.

* [Требования к железу](#требования-к-железу)
* [Подготовка](#подготовка)
* [Конфигурация для каждой ноды](#конфигурация-для-каждой-ноды)
  * [Sentry 1](#sentry-1)
  * [Sentry 2](#sentry-2)
  * [Validator](#validator)
* [Запуск](#запуск)
  * [Сеть](#сеть)
  * [Сервис](#сервис)
* [Рекомендации](#рекомендации)  

# Требования к железу

Сентри - один на сервер. U-node использует виртуализацию и ставит по 2 сентри на сервер, но если вы не знаете как это делать грамотно - не советуем. При использовании виртуализации, сентри имеют высокие резервы по частотам; используется CPU affinity, - это необходимо, чтобы работать без сбоев и не пропускать такты. 

Валидатор — один на сервер.

При высоких нагрузках и увеличении пропусков, вторые сентри можно выключить, чтобы вся производительность и канал достались одной сентри ноде.

**Валидатор:**

* Не менее 24 GB RAM 
* SSD, а лучше NVMe 256/512 GB
* Процессор с Turbo Boost не менее 4,5 GHz (лучше 5, если сможете найти). Желательна поддержка ECC, это увеличивает аптайм на какие-то доли процента
* Не менее 4 ядер, лучше 6

**Сентри:**

* Аналогично валидатору; желательно наличие и прямых аплинков с сентри
* Процессоры с теми же требованиями, можно эконом класса, ECC не обязателен, лучше 6 ядер
* В будущем, когда загрузки в сети достигнут предела текущей инфраструктуры, мы планируем собирать сентри на заказ на Intel Scalable

 # Подготовка

Вам необходимо установить операционную систему, скачать последнюю сборку Minter ноды на Github: https://github.com/MinterTeam/minter-go-node/releases. В примерах ниже, сервис запускаем под пользователем `minter`. Последняя версия Minter ноды на момент написания статьи `1.0.4`, и все ссылки в примерах на нее. LVM на ваше усмотрение, - вам должно быть виднее. Таблицу разделов предлагаем следующую:

```
/boot 320MB
swap 2GB
/ 100% XFS или BTRFS
```

**Создайте пользователя**

```
# useradd minter
```

**Установите пакеты и сгенерируйте ключи**

```
# yum -y install epel-release 
# yum -y install unzip wget nload leveldb-devel
# su - minter
$ wget https://github.com/MinterTeam/minter-go-node/releases/download/v1.0.4/minter_1.0.4_linux_amd64.zip
$ unzip minter_1.0.4_linux_amd64.zip
$ ./minter node #запускаем ноду, чтобы создать конфигурацию и ключи
$ CTRL + C #прерываем работу ноды сразу после запуска
```

**Запишите параметры и сохраните себе куда-нибудь:**

```
$ ./minter show_node_id
$ ip addr
```

У вас должен получиться список из 3 нод:

**Sentry#1**

```
ip_ext 8.8.8.8
ip_int 172.16.0.15
id 25448847baf143149af07e01f4dba128cbfb9cb2
```

**Sentry#2**

```
ip_ext 8.8.8.6
id 49cc44a10f68a0a8dcf6926d3b54f3848c55dea8
```

**Validator**

```
ip_ext 8.8.8.34
ip_int 172.16.0.5
id 437170c01777d2d4066317c400d4c9ee1c23f0a5
```

Прочтите конфигурацию, чтобы проверить новые умолчания, заложенные в конфиг, на любой из машин:

```
$ grep ^[^#] .minter/config/config.toml
```

Получите примерно следующее:

```
moniker = "название сервера"
gui_listen_addr = ":3000"
api_listen_addr = "tcp://0.0.0.0:8841"
validator_mode = false
keep_state_history = false
api_simultaneous_requests = 100
fast_sync = true
db_backend = "cleveldb"
db_path = "tmdata"
log_level = "consensus:info,main:info,blockchain:info,state:info,*:error"
log_format = "plain"
log_path = "stdout"
priv_validator_key_file = "config/priv_validator.json"
priv_validator_state_file = "config/priv_validator_state.json"
node_key_file = "config/node_key.json"
prof_laddr = ""
[rpc]
laddr = "tcp://127.0.0.1:26657"
grpc_laddr = ""
grpc_max_open_connections = 900
unsafe = false
max_open_connections = 900
[p2p]
laddr = "tcp://0.0.0.0:26656"
external_address = ""
seeds = "25104d4b173d1047e9d1a70cdefde9e30707beb1@84.201.143.192:26656,1e1c6149451d2a7c1072523e49cab658080d9bd2@minter-nodes-1.mainnet.btcsecure.io:26656,667b26ffa9f844719a9cd73f96a49252f8bfd7df@node-1.minterdex.com:26656,c098df48319b81a7535b9784873d0f143f8b72f5@minter-node-1.rundax.com:26656"
persistent_peers = ""
upnp = false
addr_book_strict = true
flush_throttle_timeout = "10ms"
max_num_inbound_peers = 40
max_num_outbound_peers = 10
max_packet_msg_payload_size = 1024
send_rate = 15360000
recv_rate = 15360000
pex = true
seed_mode = false
private_peer_ids = ""
[mempool]
broadcast = true
wal_dir = ""
size = 10000
cache_size = 100000
[instrumentation]
prometheus = false
prometheus_listen_addr = ":26660"
max_open_connections = 3
namespace = "minter"
```
Очистите конфиг, он вам больше не понадобится, и затем создайте новый:

```
$ > .minter/config/config.toml
```

# Конфигурация для каждой ноды

Рассмотрим следующий пример:

**Sentry #1** и **Validator** объединены физическим LAN линком и видят друг-друга по сети `172.16.0.0/24`. По этой причине заставим эти две ноды общаться друг с другом по приватной сети. Также решим периодически возникающую ошибку, связанную с потерей связи между сентри и валидатором, прописав пиры друг на друга у сентри нод и валидатора. Валидатор будет закрыт от внешнего мира и будет недоступен для подключений с любых IP к порту `26656`.

----------------------------------

## Sentry 1

С небольшими коментариями

```
##### Sentry#1 #####

moniker = "пишем сюда желаемое название"
gui_listen_addr = "127.0.0.1:3000"
api_listen_addr = "tcp://127.0.0.1:8841"
validator_mode = true # --- обязательно ---
keep_state_history = false # --- обязательно ---
api_simultaneous_requests = 100
api_per_ip_limit = 1000
api_per_ip_limit_window = "1m0s"
fast_sync = true
db_path = "tmdata"
db_backend = "cleveldb"
log_level = "blockchain:info,state:info,*:error" # --- упрощенный вариант лога; можете использовать полный, если хотите, из конфига выше ---
log_path = "stdout" # --- можно писать в общий системный лог, чтобы кореллировать ошибки операционной системы с ошибками минтер, если такие возникнут ---
priv_validator_file = "config/priv_validator.json"
node_key_file = "config/node_key.json"
prof_laddr = ""

##### RPC ##### Секция и ее параметры не важны, мы "глушим" RPC, вешая его на loopback.
[rpc]
laddr = "tcp://127.0.0.1:26657"
grpc_laddr = ""
grpc_max_open_connections = 900
unsafe = false
max_open_connections = 900

##### P2P #####
[p2p]
laddr = "tcp://0.0.0.0:26656"
external_address = "8.8.8.8:26656"
upnp = false
seeds = "25104d4b173d1047e9d1a70cdefde9e30707beb1@84.201.143.192:26656,1e1c6149451d2a7c1072523e49cab658080d9bd2@minter-nodes-1.mainnet.btcsecure.io:26656,667b26ffa9f844719a9cd73f96a49252f8bfd7df@node-1.minterdex.com:26656,c098df48319b81a7535b9784873d0f143f8b72f5@minter-node-1.rundax.com:26656"
persistent_peers = "437170c01777d2d4066317c400d4c9ee1c23f0a5@172.16.0.5:26656" # --- соедниение с валидатором по приватной ссылке ---
addr_book_file = "config/addrbook-main.json"
addr_book_strict = true # --- в адресной книге нельзя сохранять приватные айпи ---
flush_throttle_timeout = "10ms"
max_num_inbound_peers = 25 # ---- между 25 и 35 при текущей нагрузке на сеть для 1Gb/s интерфейса ---
max_num_outbound_peers = 25
max_packet_msg_payload_size = 1024 # --- это параметр сети, его нельзя менять! ---
send_rate = 15360000
recv_rate = 15360000
pex = true
seed_mode = false
private_peer_ids = "437170c01777d2d4066317c400d4c9ee1c23f0a5" # --- ID валидатора, если вы забудете его сюда включить - его айпи утечет в сеть и появится в адресныx книгах, вам придется все начинать сначала, брать другой реальный IP адрес и т.д. ---

##### MEMPOOL ##### Не трогать, это оптимальные настройки
[mempool]
recheck = true
broadcast = true
wal_dir = "wal"
size = 10000
cache_size = 100000

##### PROMETEUS ##### Если есть желание его включать - пошаманьте. 
[instrumentation]
prometheus = false
prometheus_listen_addr = ":26660"
max_open_connections = 3
namespace = "minter"
```

## Sentry 2

```
##### Sentry#2 #####

moniker = "GOD_SAVE_THE_QUEEN"
gui_listen_addr = "127.0.0.1:3000"
api_listen_addr = "tcp://127.0.0.1:8841"
validator_mode = true
keep_state_history = false
api_simultaneous_requests = 100
api_per_ip_limit = 1000
api_per_ip_limit_window = "1m0s"
fast_sync = true
db_path = "tmdata"
db_backend = "cleveldb"
log_level = "blockchain:info,state:info,*:error" 
log_path = "stdout"
priv_validator_file = "config/priv_validator.json"
node_key_file = "config/node_key.json"
prof_laddr = ""

##### RPC #####
[rpc]
laddr = "tcp://127.0.0.1:26657"
grpc_laddr = ""
grpc_max_open_connections = 900
unsafe = false
max_open_connections = 900

##### P2P #####
[p2p]
laddr = "tcp://0.0.0.0:26656"
external_address = "8.8.8.6:26656"
upnp = false
seeds = "25104d4b173d1047e9d1a70cdefde9e30707beb1@84.201.143.192:26656,1e1c6149451d2a7c1072523e49cab658080d9bd2@minter-nodes-1.mainnet.btcsecure.io:26656,667b26ffa9f844719a9cd73f96a49252f8bfd7df@node-1.minterdex.com:26656,c098df48319b81a7535b9784873d0f143f8b72f5@minter-node-1.rundax.com:26656"
persistent_peers = "437170c01777d2d4066317c400d4c9ee1c23f0a5@8.8.8.34:26656"
addr_book_file = "config/addrbook-main.json"
addr_book_strict = true
flush_throttle_timeout = "10ms"
max_num_inbound_peers = 25
max_num_outbound_peers = 25
max_packet_msg_payload_size = 1024
send_rate = 15360000
recv_rate = 15360000
pex = true
seed_mode = false
private_peer_ids = "437170c01777d2d4066317c400d4c9ee1c23f0a5"

##### MEMPOOL #####
[mempool]
recheck = true
broadcast = true
wal_dir = "wal"
size = 10000
cache_size = 100000

##### PROMETEUS ##### 
[instrumentation]
prometheus = false
prometheus_listen_addr = ":26660"
max_open_connections = 3
namespace = "minter"
```

## Validator

Разница с сентри выделена комментариями, обратите внимание

```
##### Validator #####

moniker = "PRAY"
gui_listen_addr = "127.0.0.1:3000"
api_listen_addr = "tcp://127.0.0.1:8841"
validator_mode = true
keep_state_history = false
api_simultaneous_requests = 100
api_per_ip_limit = 1000
api_per_ip_limit_window = "1m0s"
fast_sync = true
db_path = "tmdata"
db_backend = "cleveldb"
log_level = "blockchain:info,state:info,*:error" 
log_path = "stdout"
priv_validator_file = "config/priv_validator.json"
node_key_file = "config/node_key.json"
prof_laddr = ""

##### RPC #####
[rpc]
laddr = "tcp://127.0.0.1:26657"
grpc_laddr = ""
grpc_max_open_connections = 900
unsafe = false
max_open_connections = 900

##### P2P #####
[p2p]
laddr = "tcp://0.0.0.0:26656"
external_address = "8.8.8.34:26656"
upnp = false
seeds = "" # --- Обратите внимание ---
persistent_peers = "25448847baf143149af07e01f4dba128cbfb9cb2@172.16.0.15:26656,49cc44a10f68a0a8dcf6926d3b54f3848c55dea8@8.8.8.6:26656"
addr_book_file = "config/addrbook-main.json"
addr_book_strict = false # --- Обратите внимание ---
flush_throttle_timeout = "10ms"
max_num_inbound_peers = 25
max_num_outbound_peers = 25
max_packet_msg_payload_size = 1024
send_rate = 15360000
recv_rate = 15360000
pex = false # --- Обратите внимание ---
seed_mode = false
private_peer_ids = "" # --- Обратите внимание ---

##### MEMPOOL #####
[mempool]
recheck = true
broadcast = true
wal_dir = "wal"
size = 10000
cache_size = 100000

##### PROMETEUS ##### 
[instrumentation]
prometheus = false
prometheus_listen_addr = ":26660"
max_open_connections = 3
namespace = "minter"
```

# Запуск

## Сеть

**Базовая защита**

Обратите внимание, если у вас SSH на нестандартном порту, вы потеряете связь с сервером.

Для начала запустите файрволл:

```
# service firewalld start
# chkconfig firewalld on
```

Убедитесь, что все интерфейсы находятся в зоне `public`:

```
# firewall-cmd --info-zone=public
```

**Добавьте правила**

**Sentry#1 и Sentry#2**

```
# firewall-cmd --zone=public --permanent --add-port=26656/tcp
# firewall-cmd --reload
```

**Validator**

```
# firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="172.16.0.15/32" port port="26656" protocol="tcp" accept' # --- для Sentry#1 ---
# firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="8.8.8.6/32" port port="26656" protocol="tcp" accept' # --- для Sentry#2 ---
```

## Сервис

**Установите лимиты системы**

```
# vi /etc/security/minter.conf
minter         hard    nofile      500000
minter         soft    nofile      500000
minter         hard    nproc       500000
minter         soft    nproc       500000
```

**Создайте сервис файл**

```
# vi /lib/systemd/system/minter.service
[Unit]
Description=Minter Validator
After=network.target auditd.service

[Service]
ExecStart=/home/minter/minter node --home-dir=/home/minter/.minter
Type=simple
KillMode=process
Restart=always
RestartSec=3
User=minter

LimitNOFILE=500000
LimitNPROC=500000

[Install]
WantedBy=multi-user.target
Alias=minter.service
```

**Установите минтер на автозапуск и перезагрузите систему**

```
# chkconfig minter on
# reboot
```
# Рекомендации

Как посчитать предел коннектов? Проверьте реальную скорость каналов на сервере заранее:

```
wget https://raw.githubusercontent.com/sivel/speedtest-cli/master/speedtest.py
chmod +x speedtest.py
./speedtest.py
```

Снимайте метрики раза 4 за сутки. На основе полученных данных вы сможете рассчитать количество соединений. К примеру, у вас среднее значение такое:

```
Testing download speed................................................................................
Download: 702.08 Mbit/s
Testing upload speed................................................................................................
Upload: 759.45 Mbit/s
```

Следовательно, предел коннектов при идеальных условиях:

```
Download (inbound_peers) 702 / 15 (округлим 15360000b/s в большую сторону) = 46
Upload (outbound_peers) 759 / 15 = 50
```

От полученных данных отсеките пятую часть, т.к. это интернет и не стоит доверять даже среднему значению. Следовательно, предельные значения для нод могут быть такими:

```
max_num_inbound_peers = 36
max_num_outbound_peers = 40
```

Если у вас два сервера на одной машине, то значения следует поделить пополам, с округлением в меньшую сторону.

## Minter-guard

Даже админы спят. Бывают накладки и в датацентрах — везде работают люди и что-то всегда может пойти не так.

Чтобы минимизировать риски отказа ноды, мы советуем использовать сервис **Minter Guard**. Он доступен на Github с подробной инструкцией: https://github.com/U-node/minter-guard/

По его применению пишите мне в Telegram `@ustinovpro`, я проконсультирую как правильно им пользоваться, чтобы не было проблем.

## Общие советы по железу в Hetzner:

* EX-52/62-NVME для сентри
* PX-62-NVME для валидатора

## Конфигурирование нод:

Все в ваших руках — как успех, так и провал. Поэтому внимательно проверяйте адреса в `persistent_peers`, тестируйте серверы неделю после запуска, как они друг-друга видят, не отваливаются ли сентри от валидатора и потом в бой. 

Удачи!
