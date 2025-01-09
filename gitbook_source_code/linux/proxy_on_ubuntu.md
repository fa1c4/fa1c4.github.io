# Proxy on Ubuntu [Hidden Blog 0x1]

在 Ubuntu 服务器上配置代理

## v2rayA

github 仓库链接: [https://github.com/v2rayA/v2rayA-installer](https://github.com/v2rayA/v2rayA-installer)

与 v2ray core 一起安装 (v2ray 客户端需要 core 支持)

```shell
sudo sh -c "$(wget -qO- https://github.com/v2rayA/v2rayA-installer/raw/main/installer.sh)" @ --with-v2ray
# or use curl to install 
# sudo sh -c "$(curl -Ls https://github.com/v2rayA/v2rayA-installer/raw/main/installer.sh)" @ --with-v2ray

# install successfully output:
# ...
Start v2rayA service now:
systemctl start v2raya
Start v2rayA service at system boot:
systemctl enable v2raya
--------------------------------------------------------------------------------
1. v2rayA has been installed to your system, the configuration directory is
   /usr/local/etc/v2raya.
2. v2rayA will not start automatically, you can start it by yourself.
3. If you want to uninstall v2rayA, please run uninstaller.sh.
4. If you want to update v2rayA, please run installer.sh again.
5. Official website: https://v2raya.org.
6. If you forget your password, run "v2raya-reset-password" to reset it.
--------------------------------------------------------------------------------
```

如果要卸载则执行

```shell
sudo sh -c "$(wget -qO- https://github.com/v2rayA/v2rayA-installer/raw/main/uninstaller.sh)"
```



启动 v2rayA 

```shell
sudo systemctl start v2raya
sudo systemctl status v2raya
# ● v2raya.service - A web GUI client of Project V which supports VMess, VLESS, SS, SSR, Trojan, Tuic and Juicity protocols
#     Loaded: loaded (/etc/systemd/system/v2raya.service; disabled; vendor preset: enabled)
#     Active: active (running) since Thu 2025-01-09 10:57:25 CST; 5s ago
#       Docs: https://v2raya.org
#   Main PID: 929035 (v2raya)
#      Tasks: 11 (limit: 328153)
#     Memory: 23.0M
#     CGroup: /system.slice/v2raya.service
#            └─929035 /usr/local/bin/v2raya

# then ssh redirect remote server port 2017 to localhost 2017 to access configure website of v2raya
ssh -L2017:localhost:2017 user@127.0.0.1 -p 19999
# open localhost:2017 on web browser and import subscription url ...
# update the proxy configuration set the proxy port as specific port like 2444 for http and 2333 for socks5 
# run the proxy v2raya

# enable boot startup v2raya
sudo systemctl daemon-reload
sudo systemctl enable --now v2raya
```



## proxychains

```shell
sudo apt install proxychains
sudo vim /etc/proxychains.conf
# add configuration at last of .conf file
# socks4        127.0.0.1 9050
socks5         127.0.0.1 2333
http           127.0.0.1 2444

# set environment variables
export ALL_PROXY=socks5://127.0.0.1:2333
export HTTP_PROXY=http://127.0.0.1:2444
export HTTPS_PROXY=http://127.0.0.1:2444

# proxychains usage examples
proxychains wget google.com
proxychains wget youtube.com
# ...
# Saving to: ‘index.html.2’
# ...
```



#### 报错解决

1. 如果 proxychains 走不通代理, 先测试代理服务是否有效

```shell
wget -e use_proxy=yes -e http_proxy=127.0.0.1:2444 http://www.google.com
# --2025-01-09 11:22:47--  http://www.google.com/
# Connecting to 127.0.0.1:2444... connected.
# Proxy request sent, awaiting response... 200 OK
# Length: unspecified [text/html]
# Saving to: ‘index.html’
# ...
```

如果有效, 则修改 DNS 解析配置文件

```shell
sudo vim /etc/resolv.conf
# add two namesevers
nameserver 8.8.8.8
nameserver 8.8.4.4
```



2. 如果添加 nameserver 依然不能走通 DNS

修改 `/etc/proxychains.conf`, 取消 http 协议的代理设置

```shell
# proxychains.conf  VER 3.1
#
#        HTTP, SOCKS4, SOCKS5 tunneling proxifier with DNS.
#

# The option below identifies how the ProxyList is treated.
# only one option should be uncommented at time,
# otherwise the last appearing option will be accepted
#
# dynamic_chain
#
# Dynamic - Each connection will be done via chained proxies
# all proxies chained in the order as they appear in the list
# at least one proxy must be online to play in chain
# (dead proxies are skipped)
# otherwise EINTR is returned to the app
#
strict_chain
#
# Strict - Each connection will be done via chained proxies
# all proxies chained in the order as they appear in the list
# all proxies must be online to play in chain
# otherwise EINTR is returned to the app
#
#random_chain
#
# Random - Each connection will be done via random proxy
# (or proxy chain, see  chain_len) from the list.
# this option is good to test your IDS :)

# Make sense only if random_chain
#chain_len = 2

# Quiet mode (no output from library)
#quiet_mode

# Proxy DNS requests - no leak for DNS data
proxy_dns 

# Some timeouts in milliseconds
tcp_read_time_out 15000
tcp_connect_time_out 8000

# ProxyList format
#       type  host  port [user pass]
#       (values separated by 'tab' or 'blank')
#
#
#        Examples:
#
#               socks5  192.168.67.78   1080    lamer   secret
#               http    192.168.89.3    8080    justu   hidden
#               socks4  192.168.1.49    1080
#               http    192.168.39.93   8080
#
#
#       proxy types: http, socks4, socks5
#        ( auth types supported: "basic"-http  "user/pass"-socks )
#
[ProxyList]
# add proxy here ...
# meanwile
# defaults set to "tor"
# socks4        127.0.0.1 9050
socks5         127.0.0.1 2333
# http           127.0.0.1 2444
# https          127.0.0.1 2444
```



