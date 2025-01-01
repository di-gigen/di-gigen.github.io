---
title: 与Caddy复用端口的Hysteria2代理方案
date: 2024-11-30 19:58:00
tags: [科学上网,hysteria2,sing-box,caddy,VPS,UDP,HTTP/3,建站]
category: [计算机技术]
lede: 科学上网系列(その三)
thumbnail: /img/caddy-with-hysteria2/tcp&udp-visualized.jpg
---
## 前言
对于部分侧重发包而非传输可靠性的使用场景如流媒体、直播、游戏联机的场景，如果您的VPS线路足够优质，那么自建基于UDP协议的Hysteria2节点能帮您省下游戏加速器的花费。本文将为使用Caddy建站的服主们提供Hysteria2的端口复用伪装部署方式。  

## 准备 
* 本系列将继续沿用在[前篇](https://di-gigen.github.io/2024/11/21/caddy-sni-with-reality/)中详述过的**Caddy负责入站分流、sing-box为多协议代理后端**的方案架构，故本次仅针对换用Hysteria2协议而产生的些微改动做重点说明。如果您是初学者，请结合系列其他文章拼凑出完整的配置文件。  
* Hysteria2通过模拟Web Server的行为实现伪装，对象是同服务器上部署的真实网站`blog.mydomain.com`，我们假设您已经在服务器上建站并为其申请了SSL证书，在之后的实现中会调用它。  

## 安装与配置
### Caddy
Caddy的[安装](https://caddyserver.com/docs/install)过程不再赘述。原本Web Server会接管TCP和UDP传输层响应不同http协议版本的网站请求，这通常发生在同一个接口上，为保证伪装效果代理后端需要和Web Server复用监听接口。对Caddy一侧来说意味着在配置文件中禁用对HTTP/3支持以避免占用`UDP 8443`端口：  
<details>
<summary><font color="#E02222">/etc/caddy/Caddyfile(节选)</font></summary>

```
{
    # Global Options
    servers :8443 {
        protocols h1 h2 h2c # 监听协议不包含UDP上的HTTP/3
    }
    ...
}
*.mydomain.com {
    ...
}
```
</details>

### sing-box
无论您使用什么后端实现Hysteria2服务，它都需要代替Caddy接管服务器https端口的UDP流量，对其中访问服务器上网站`blog.mydomain.com`的常规流量和Hysteria2代理协议流量分流处理。  
在sing-box配置模板中我们使用了相对保守的BBR流控算法。要使用它，除了需要设定代理用户的`MY_USERNAME`和`MY_PASSWORD`，还需要按实际情况修改网站域名等信息、确保证书路径能够被sing-box正确索引，否则你需要让代理后端自签证书提供给Hysteria2。  
 
<details>
<summary><font color="#E02222">/etc/singbox/config.json(节选)</font></summary>

```json
{
    ...
    "inbounds": [{
        "tag": "hy2-in",
        "type": "hysteria2",
        "listen": "::",
        "listen_port": 8443,
        "sniff": true,
        "ignore_client_bandwidth": true,
        "masquerade": "http://localhost:8080",
        "users": [{
            "name": "MY_USERNAME",
            "password": "MY_PASSWORD"
        }],
        "tls": {
            "enabled": true,
            "server_name": "blog.mydomain.com",
            "key_path": "/path/to/wildcard_.mydomain.com.key",
            "certificate_path": "/path/to/wildcard_.mydomain.com.crt"
        }
    }],
    "outbounds": [...],
    "route": {...}
}
```
</details>

### 性能调优
调增UDP缓冲区(此处为16M)  
```shell
echo "net.core.rmem_max=16777216" >> /etc/sysctl.conf && echo "net.core.wmem_max=16777216" >> /etc/sysctl.conf
```

### 客户端配置
使用剪贴板将节点分享链接导入您的客户端。其中`MY_PASSWORD`需要和sing-box配置文件保持一致。    
```
hysteria2://MY_PASSWORD@blog.mydomain.com:8443?insecure=0
```

## 完成！
启动Caddy与sing-box服务。此时可通过`sudo ss -lnp | grep :8443`命令查询接口占用情况。若配置无误，可以看到Caddy和sing-box分别监听8443端口的TCP和UDP。  
若有余力，还可以模拟网络審查官员使用`curl --http3-only https://blog.mydomain.com:8443`对端口进行扫描，会返回服务器上网站的真实网页内容以达到伪装目的。  
对于已经在使用xray协议的用户请注意，Hysteria2[不支持](https://v2.hysteria.network/zh/docs/misc/CDN/)CDN中转流量。  
