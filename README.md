
### Use OpenVPN to connect to school environment at home

#### Requirement
1. A VPS with Public IP 
2. A computer at school

##### Initialization OpenVpn on the computer at school. 
```
export OVPN_DATA="ovpn-data-anyname"
docker volume create --name $OVPN_DATA
docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://yourvpsip # (yourvpsip can be ip or host of your vps)
docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
```

Note that, you need inputting CA key for several times. Please ensure they are consistent. 
```
docker run -v $OVPN_DATA:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn # run
docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full user1 # Generate client certificate
docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient user1 > user1.ovpn # Export client certificate to local directory
```
At this point, the client certificate has been generated, but the default traffic has gone through the proxy, without diversion.

##### diversion (Proxy and Direct)
First, comment out ``redirect-gateway def1'' in the client certificate. 
Second, edit the config file using the following command: 
```
docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn vi /etc/openvpn/openvpn.conf # 
```

My config file
```
### Push Configurations Below
# push "block-outside-dns"
push "dhcp-option DNS XXX.45.240.99" # XXX.45.240.99 is the DNS server of my school. For Intranet domain name resolution. 
push "dhcp-option DNS 8.8.8.8" # public 
push "dhcp-option DNS 8.8.4.4"
# the following address will be visited through Proxy. 
push "route 172.16.200.0 255.255.255.0"
push "route XXX.XXX.0.0 255.255.240.0" # Hefei University of Technology
push "route XXX.XXX.16.0 255.255.248.0" # Hefei University of Technology
push "route XXX.XXX.240.0 255.255.240.0" # Hefei University of Technology
push "route XXX.XXX.240.0 255.255.255.0" # Hefei University of Technology
push "route XXX.XXX.192.0 255.255.192.0" # Hefei University of Technology
push "route 121.248.0.0 255.252.0.0" # China Education and Research Network
push "route 121.250.0.0 255.254.0.0" # China Education and Research Network
push "route 210.45.0.0  255.255.0.0" # China Education and Research Network
push "comp-lzo no"
```
Hefei University of Technology should be replaced with yours 

Third, Self-start
```
docker update --restart=always container_id_of_openvpn
```

How to add new users for OpenVPN
```
export OVPN_DATA="ovpn-data-anyname"
docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full user2
```
Now, openVpn has been successfully set. 

##### Visit Openvpn through the Public IP. 
I use frp to realize the Port mapping.
1. A VPS with Public IP (Server of frp) frps
2. A computer at school (Client of frp) frpc

Use frp to map 1194 port (or any other OpenVpn port) of the school 
frpc-0.37,
Edit frpc.ini 
```
[common]
# A literal address or host name for IPv6 must be enclosed
# in square brackets, as in "[::1]:80", "[ipv6-host]:http" or "[ipv6-host%zone]:80"
server_addr = your_vps_ip
server_port = 7117
# for authentication
token = ***********************************************
# connections will be established in advance, default value is zero
pool_count = 5

# if tcp stream multiplexing is used, default is true, it must be same with frps
tcp_mux = true

# decide if exit program when first login failed, otherwise continuous relogin to frps
# default is true
login_fail_exit = true

# communication protocol used to connect to server
# now it supports tcp and kcp and websocket, default is tcp
protocol = tcp

# proxy names you want to start seperated by ','
# default is empty, means all proxies
# start = ssh,dns

# heartbeat configure, it's not recommended to modify the default value
# the default value of heartbeat_interval is 10 and heartbeat_timeout is 90
# heartbeat_interval = 30
# heartbeat_timeout = 90

log_file = /tmp/frpc.log
log_level = warn
log_max_days = 7

[openvpn]
type = udp
local_ip = 127.0.0.1
local_port = 1194
remote_port = 1194# your open port of vps
use_encryption = false
use_compression = false

```
Edit frps.ini on the Server 
install frps-0.37 (better make sure the same version) on the vps,token in frpc and frps must be the same, allow_ports must include 1194
```
[common]
bind_addr = 0.0.0.0
bind_port = 7117
bind_udp_port = 7118
protocol = tcp
# set dashboard_addr and dashboard_port to view dashboard of frps
# dashboard_addr's default value is same with bind_addr
# dashboard is available only if dashboard_port is set
dashboard_addr = 0.0.0.0
dashboard_port = 7116
dashboard_user = ******
dashboard_pwd = ************
# auth token
token = ***********************************************

# heartbeat configure, it's not recommended to modify the default value
# the default value of heartbeat_timeout is 90
heartbeat_timeout = 90

# only allow frpc to bind ports you list, if you set nothing, there won't be any limit
allowports = 1194
max_pool_count = 20
max_ports_per_client = 0
tcp_mux = true
log_file = /tmp/frps.log
log_level = warn
log_max_days = 7

```
Remember to open port 1194 of the firewall of the public server (or any port you set).
Finally, remember to look at the remote xxx.xxx.xxx.xxx yyyy udp in the client certificate, Is xxx. xxx. xxx. xxx yyyy your server IP and port? If not, modify it manually.

Refererences
[1] https://github.com/kylemanna/docker-openvpn

[2] https://github.com/fatedier/frp/releases/download/v0.37.0/frp_0.37.0_linux_amd64.tar.gz
