
### 校园代理-OpenVPN代理分流

#### 以学生身份买了个公网服务器打算用作校园代理，但是有流量限制，故分流。设置如下
##### 初运行以及分流设置
```
export OVPN_DATA="ovpn-data-anyname"
docker volume create --name $OVPN_DATA
docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://yourvpsip # (yourvpsip can be ip or host of your vps)
docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
```
CA key要多次输入，请记清楚，不要前后输入不一致
```
docker run -v $OVPN_DATA:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn # 开始运行
docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full user1 # 生成客户端证书
docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient user1 > user1.ovpn # 导出user1的客户端证书到本地目录
```
到这，其实客户端证书已经生成了，不过默认的流量全部走代理了，不分流。
继续处理。
首先，注释掉客户端证书里的最后一句，就是# redirect-gateway def1 这一句
```
docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn vi /etc/openvpn/openvpn.conf # 更新配置文件，配置分流内容
```
以上内容是配置需要的分流内容，把需要走代理的写进route里面，注释掉block-outside-dns
```
### Push Configurations Below
# push "block-outside-dns"
push "dhcp-option DNS XXX.45.240.99" # XXX.45.240.99是我们学校的DNS服务器地址
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
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
Hefei University of Technology部分换成自己学校的部分
##### 添加新用户
```
export OVPN_DATA="ovpn-data-anyname"
docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full user2
```
##### frp将1194端口曝露出去
frp可以把挂载openvpn服务的机器的1194端口曝露出去
在本地机器安装frpc-0.37
frpc.ini证书
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
在公网服务器上安装frps-0.37，证书内容注意token与frpc要相同，allow_ports要包括1194
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
最后记得把公网服务器的防火墙的1194端口打开（或者你设置的任意端口）
最后的最后，记得看下客户端证书里 remote xxx.xxx.xxx.xxx yyyyy udp
 xxx.xxx.xxx.xxx yyyyy是不是你的服务器ip和端口，如果不是，手动修改一下
 
[1] https://github.com/kylemanna/docker-openvpn
[2] https://github.com/fatedier/frp/releases/download/v0.37.0/frp_0.37.0_linux_amd64.tar.gz
