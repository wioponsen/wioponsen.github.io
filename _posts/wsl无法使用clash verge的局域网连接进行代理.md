---
layout: post
title: 'wsl无法使用clash verge的局域网连接进行代理'
date: 2027-08-4 10:44:00 +0000
author: wioponsen
categories: [blogs]
tags: [blogs]
math: true
mermaid: true
---

{% raw %}

有时候win10主机开了代理，而想在wsl下共用，但是发现无法连接，可以进行排查。

win端：
1. clash设置里允许局域网连接/Allow LAN
2. 确认端口， 一般是7890， 开启代理
3. 防火墙放行
```
# 方法一：直接给 7890 端口放行（推荐）
New-NetFirewallRule -DisplayName "Clash Verge WSL Proxy" -Direction Inbound -Protocol TCP -LocalPort 7890 -Action Allow

# 方法二：更彻底——对 WSL 虚拟网卡禁用防火墙（很多人用这个立刻就通）
Set-NetFirewallProfile -DisabledInterfaceAliases "vEthernet (WSL)"

```

wsl端，配置：
在 `~/.bashrc` 中追加
```
export HOST_IP=$(ip route | grep default | awk '{print $3}')
PROXY_PORT=7890

function proxy_on() {
    export http_proxy="http://${HOST_IP}:${PROXY_PORT}"
    export https_proxy="http://${HOST_IP}:${PROXY_PORT}"
    export ALL_PROXY="socks5://${HOST_IP}:${PROXY_PORT}"
    echo -e "\033[32m[✔] WSL 代理已开启。目标宿主机 IP: ${HOST_IP}:${PROXY_PORT}\033[0m"
}

function proxy_off() {
    unset http_proxy
    unset https_proxy
    unset ALL_PROXY
    echo -e "\033[31m[✘] WSL 代理已关闭。\033[0m"
}

function proxy_test(){
    curl -s --max-time 5 https://www.google.com -o /dev/null -w "%{http_code}" && echo " - 代理正常" || echo " - 代>理异常"
}
```


然后 `source` 一下
```
source ~/.bashrc
```

测试排查：
1. 测试端口连通， `nc -vz $HOST_IP 7890` , 无法连通就windows开方端口或者禁用防火墙
2. 测试代理， proxy_test


{% endraw %}
