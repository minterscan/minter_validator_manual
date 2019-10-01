<p align="center" background="black"><img src="../img/unode-logo.svg" width="450"></p>

Hello, community! We'd like to share a tutorial how to deploy the Minter node on Centos. 

The instruction describes only basic things, it is an average working variant without additional protection scripts and additional tools. They are briefly described in the recommendations section.

All configs and commands in the instructions are provided for the original Centos 7 image. The instruction can be easily adapted to other Linux based operating systems.

* [Hardware requirements](#hardware-requirements)
* [Preparing](#preparing)
* [Configuration](#configuration)
  * [Sentry 1](#sentry-1)
  * [Sentry 2](#sentry-2)
  * [Validator](#validator)
* [Running](#running)
  * [Network](#network)
  * [Service](#service)
* [Recommendations](#recommendations)  

# Hardware requirements

Sentry — one per server. U-node uses virtualization and puts 2 sentries on the server, but if you don't know how to do it properly — we don't recommend that you do that. When using virtualization, sentries has high clock limits; the CPU affinity is used, which is necessary in order to work without failures and not to miss the tacts. 

Validator — one per server.

With high loads and increased misses, the second sentry can be turned off to ensure that the entire performance and channel are allocated to one sentry node.

**Validator**

* At least 24 GB RAM 
* SSD, or better NVMe 256/512 GB
* CPU with Turbo Boost and at least 4,5 GHz (better 5, if possible). ECC support desirable, it increases the uptime by some percentage.
* At least 4 cores, better 6

**Sentry**

* Similar to the validator; it is advisable to have direct uplinks with sentry
* CPU with the same requirements, can be economy class, ECC is optional, better 6 cores
* In the future, when the network loads will reach the limit of the current infrastructure, we plan to run sentries on Intel Scalable.

 # Preparing

You need to install the operating system, download the latest release from Github: https://github.com/MinterTeam/minter-go-node/releases. The examples below uses `minter` user and the current build at the time of writing the article `1.0.4`. LVM at your discretion - you should be more aware. For GPT sections we suggest the following:

```
/boot 320MB
swap 2GB
/ 100% XFS или BTRFS
```

**Create user**

The examples below will use `minter` user and current `1.0.4` build 

```
# useradd minter
```


**Install packages & generate keys**

```
# yum -y install epel-release # yum -y install unzip wget nload leveldb-devel
# su - minter
$ wget https://github.com/MinterTeam/minter-go-node/releases/download/v1.0.4/minter_1.0.4_linux_amd64.zip
$ unzip minter_1.0.4_linux_amd64.zip
$ ./minter node #run node to create config and keys
$ CTRL + C #stop right after running
```

**Write config and save somewhere**

```
$ ./minter show_node_id
$ ip addr
```

Let's say we have a list of three nodes:

**Sentry #1**

```
ip_ext 8.8.8.8
ip_int 172.16.0.15
id 25448847baf143149af07e01f4dba128cbfb9cb2
```

**Sentry #2**

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

Read the configuration to check the new default settings on any server:

```
$ grep ^[^#] .minter/config/config.toml
```

We're getting about this:

```
moniker = "server name"
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
Let's get config cleaned up, we won't need it anymore, and we'll create a new one:

```
$ > .minter/config/config.toml
```

# Configuration

**Sentry #1** and **Validator** will be connected by a physical LAN link and will see each other over the network `172.16.0.0/24`. For this reason, we will make these two nodes communicate with each other over a private network. To solve the periodically arising error related with loss of communication between a sentry and the validator we will connect sentries and validator by linking them each other over private peers. The validator will be hidden from outside and will be unavailable for connections from any IP to the port `26656`.

----------------------------------

## Sentry 1

Look at the comments

```
##### Sentry#1 #####

moniker = "put any name here"
gui_listen_addr = "127.0.0.1:3000"
api_listen_addr = "tcp://127.0.0.1:8841"
validator_mode = true # --- required ---
keep_state_history = false # --- required ---
api_simultaneous_requests = 100
api_per_ip_limit = 1000
api_per_ip_limit_window = "1m0s"
fast_sync = true
db_path = "tmdata"
db_backend = "cleveldb"
log_level = "blockchain:info,state:info,*:error" # --- simplified log file, you can use the full one if you want from the config above ---
log_path = "stdout" # --- You can write to the general system log; it allows to correlate system errors with errors of the minter, if such errors occur ---
priv_validator_file = "config/priv_validator.json"
node_key_file = "config/node_key.json"
prof_laddr = ""

##### RPC ##### This section are not important, we turn RPC off, linking it on the loopback.
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
persistent_peers = "437170c01777d2d4066317c400d4c9ee1c23f0a5@172.16.0.5:26656" # --- connect to validator by private link ---
addr_book_file = "config/addrbook-main.json"
addr_book_strict = true # it's not allowed to save private IP in address book
flush_throttle_timeout = "10ms"
max_num_inbound_peers = 25 # --- from 25 to 35 with current network load and in the precense of 1Gb/s interface on server ---
max_num_outbound_peers = 25
max_packet_msg_payload_size = 1024 # --- minter constant, don't change! ---
send_rate = 15360000
recv_rate = 15360000
pex = true
seed_mode = false
private_peer_ids = "437170c01777d2d4066317c400d4c9ee1c23f0a5" # validator ID. If you forget to set it here - it's IP will leak to the network and will appear in the address book and you have to start all over again, take another real IP address, etc.

##### MEMPOOL ##### --- Optimal config, better don't touch this ---
[mempool]
recheck = true
broadcast = true
wal_dir = "wal"
size = 10000
cache_size = 100000

##### PROMETEUS ##### --- Optional ---
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

Difference from Sentry config is highlighted by comments, please note

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
seeds = "" # --- Please note ---
persistent_peers = "25448847baf143149af07e01f4dba128cbfb9cb2@172.16.0.15:26656,49cc44a10f68a0a8dcf6926d3b54f3848c55dea8@8.8.8.6:26656"
addr_book_file = "config/addrbook-main.json"
addr_book_strict = false # --- Please note ---
flush_throttle_timeout = "10ms"
max_num_inbound_peers = 25
max_num_outbound_peers = 25
max_packet_msg_payload_size = 1024
send_rate = 15360000
recv_rate = 15360000
pex = false # --- Please note ---
seed_mode = false
private_peer_ids = "" # --- Please note ---

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

# Running

## Network

**Basic protection**

Note that if you have SSH on a non-standard port, you will lose connect with the server.

First, let's run a firewall:

```
# service firewalld start # chkconfig firewalld on
```

All interfaces must be in the public area, let's check.

```
# firewall-cmd --info-zone=public
```

**Add rules**

**Sentry#1 Sentry#2**

```
# firewall-cmd --zone=public --permanent --add-port=26656/tcp
# firewall-cmd --reload
```

**Validator**

```
# firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="172.16.0.15/32" port port="26656" protocol="tcp" accept' # --- for Sentry#1 ---
# firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="8.8.8.6/32" port port="26656" protocol="tcp" accept' # --- for Sentry#2 ---
```

## Service

**System limits:**

```
# vi /etc/security/minter.conf
minter         hard    nofile      500000
minter         soft    nofile      500000
minter         hard    nproc       500000
minter         soft    nproc       500000
```

**Create a service file**

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

**Enable service autorun and reboot**

```
# chkconfig minter on
# reboot
```
# Recommendations

How to calculate the limit of connections? Check the real speed of the channels on the server first:

```
wget https://raw.githubusercontent.com/sivel/speedtest-cli/master/speedtest.py
chmod +x speedtest.py
./speedtest.py
```

Measure your metrics four times a day. Calculate the number of connections based on the data obtained. For example, you have an average numbers: 

```
Testing download speed................................................................................
Download: 702.08 Mbit/s
Testing upload speed................................................................................................
Upload: 759.45 Mbit/s
```

Therefore, the limit of connectors for ideal conditions:

```
Download (inbound_peers) 702 / 15 (round 15360000b/s up) = 46
Upload (outbound_peers) 759 / 15 = 50
```

From the received data cut off the fifth part, because it is the Internet, you should not trust even the average value. Therefore, the limit values for nodes can be as follows:

```
max_num_inbound_peers = 36
max_num_outbound_peers = 40
```

If you have two nodes on one physical server, the values should be divided in half, with rounding down.

## Minter-guard

Even the admins are asleep. There are overlays in data centers — everything maintained by people and something can always go wrong.

To minimize the risk of node failure, we recommend using **Minter Guard** service. It's available on Github with detailed instructions: https://github.com/U-node/minter-guard/

If you have any questions about Minter Guard, please contact me at Telegram `@ustinovpro`, I will advise you on how to use it correctly so that you don't have any problems.

## General hardware tips for Hetzner:

* EX-52/62-NVME for sentry
* PX-62-NVME for validator

## Configuration

Everything in your hands, both a success and a failure. Therefore attentively check addresses in `persistent_peers `, test servers at least a week before start, check how they see each other, check sentry for falls and then run validator. 

Good luck!
