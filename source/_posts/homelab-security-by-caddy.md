---
title: 一站式保护家庭服务器：Caddy综合应用(SSO、WAF和其他)
date: 2024-07-03 11:05:20
tags: [2FA,sso,WAF,caddy]
category: [计算机技术]
lede: 轻量Web服务安全解决方案
thumbnail: /img/homelab-security-by-caddy/121154966.jpg
---
## 前言
尽管VPN隧道技术是目前最安全的内网穿透方案，但根据个人需求不同，有时也需要为智能家居、私有化IM、家庭影院串流等服务提供广域网可达的持久化访问地址。这篇文章我们围绕网络应用出口的第一道关卡——Web Server(狭义上也可以叫做反代服务)，探讨家庭网络服务公网暴露的可行性与安全保障。  
### 妥善利用家庭宽带
当您决定购买人生中的第一台NAS的时候，就已经会思考如何在家宽出口做互联网“打洞”了。哪怕ISP吝啬静态IPv4资源，通过[免费DDNS服务](https://openwrt.org/docs/guide-user/services/ddns/client#requirements)(其中一些能为同一家宽IP下的数个网络服务附赠不同域名加以区分)解析动态IPv4或者独立IPv6地址，您都可以对外提供标准BS架构的Web服务(因此本文不会涉及FRP、ZeroTier等奇技淫巧)。  
与商业互联网服务有别，家庭网络服务面向特定少数对象(您自己和家人、朋友)，亦缺乏可靠的(企业级)安防条件，因此保持隐匿比正面防御更优先和重要，体现在数据传输各环节上可简单总结为如下事项：   
* 加密数据包避免敏感信息(账密、操作意图、个资等)泄漏  
* 隐藏源站(家)IP地址避免直接遭受网络攻击、社会工程学打击  
* 网络服务采取登录鉴权方式拒绝非法请求，且应尽可能消除能够绕过它的软件漏洞  

幸运的是我们有丰富的开源软件生态和网络公益项目来效仿企业的解决方案！通过给这些域名申请[免费SSL证书](https://letsencrypt.org/)，就可以将访问数据再加密的https协议上传输；而[某些](https://blog.cloudflare.com/waf-for-everyone/)DDNS服务商还提供免费的CDN服务，那么源站的真实IP将被隐藏在网站加速节点背后并获得云防火墙的保护。  
听起来好像哪怕是免费获取它们也需要做不少功课？但只要使用合适的Web服务工具就可以大幅简化部署过程，我们说的是[Caddy Server](https://caddyserver.com/)。  

### Caddy：用单一组件解决最多的web应用(安全)需求
如果您是从本站的其他内容来到这里，应该了解到我们在项目中广泛采用Caddy，请允许我再简单介绍一下它。继Apache、Nginx这些伟大先驱之后，Caddy的理念是以模块化的方式整合Web反代服务、API网关、证书管理器、应用防火墙等诸多功能，并且尽可能提供自然人易读的配置形式与缺省自动化的功能实现，以上文描述的家庭服务器应用场景来说，我们从[官方插件库](https://caddyserver.com/download?package=github.com%2Fcaddy-dns%2Fcloudflare&package=github.com%2Fcaddyserver%2Ftransform-encoder&package=github.com%2Fgreenpau%2Fcaddy-security&package=github.com%2Fcorazawaf%2Fcoraza-caddy&package=github.com%2Fmholt%2Fcaddy-dynamicdns)挑选了这些插件：    

| 模块 | 用途 |  
| --- | --- |  
| [cloudflare](https://github.com/caddy-dns/cloudflare)<br/> | 自动签发证书(DNS质询模式) |  
| [transform-encoder](https://github.com/caddyserver/transform-encoder)<br/> | 日志格式化工具(增加易读性) |  
| [caddy-security](https://github.com/greenpau/caddy-security)<br/> | 为所有网络应用和服务提供单点登录 |   
| [caddy-ddns](https://github.com/mholt/caddy-dynamicdns)<br/> | 解析家宽IP |  
| [coraza-caddy](https://github.com/corazawaf/coraza-caddy)<br/> | 应用防火墙 |   

需要为新手做出的解释如下：  
* 尽管Caddy默认情况下也可以自动签发证书，但您当地(如中国大陆地区)的ISP可能出于政策封锁常见低位端口(≤1024)，这也阻止通过http和tls-alpn验证方式获取证书。DNS质询模式不仅可以绕开这一限制，还可以为您申请泛域名证书令多个子域通用一套证书。  
* 我们无法保证您暴露到公网的所有服务都能提供足够强的密码策略，而您肯定也不想记这么多密码。通过为它们提供一个前置的统一认证登录平台，实现一套账密即可访问所有应用。配合多因素验证令牌，您不必担心暴力破解和密码泄漏。  
* CDN提供的高防和家庭网络边界路由的防火墙只能基于网络流量的IP地址、端口和协议信息过滤和拦截网络层和传输层的攻击(如端口扫描、DDoS攻击等)，但无法防御利用应用层发起的攻击(SQL注入、跨站请求伪造等)，应用防火墙(WAF)可用于缓解此类软件漏洞被攻击者利用。  
* Caddy的日志记录了每一次访问请求和WAF拦截事件，是十分重要的事后安全审计材料。使用插件格式化日志将提高易读性。  

作为对比，如果采用Nginx作为Web服务器要想满足同样的需求，哪怕从简也还需要旁挂DDNS解析服务、证书服务acme、统一认证平台authelia、应用防火墙Modsecurity并且各自独立配置运行，增加了很多潜在故障点。相较之下集成功能插件的Caddy在整个网络拓扑上只新增了单点服务即可接管其他的家庭服务器，且模块与Caddy自身的各项运行参数采用相同语法配置在同一个文件中，随着需求的不断增加，其学习成本和结构复杂度低的优势会愈发体现。  

## 场景预设和效果预期 
要完成本文的操作实践，需要复现以下前提条件：  
* 家庭宽带具备可供解析的(动态)IPv4或IPv6  
* 用户持有域名`mydomain.com`，家庭服务器上部署若干需要暴露在公网的内网服务`service1`与`service2`  
* 考虑到由于ISP政策限制，使用高位端口`8443`替代`443`作为https端口  

按照下文引导完成部署后，将达成实现以下效果或者功能：  
* `service1`与`service2`分配到子域名地址`sub1.mydomain.com:8443`和`sub2.mydomain.com:8443`，复用Caddy签发的通配符证书(本文中由科赋锐提供DNS质询服务)并启用全局https传输加密  
* 作为配置对照组，`service1`接入单点登录(即访问该服务将被重定向至`auth.mydomain.com:8443`鉴权)，`service2`模拟无限制公共服务不做接入。但二者均受到WAF保护  

## 实操步骤
### 定制安装Caddy  
按需集成插件的Caddy与官方预编译的运行文件和各发行版提供的安装包有所不同，我们需要自助生成定制版本，方法由易到难分别是：  

#### 官方一键云编译
访问上文中提及的官方插件库，勾选需求表中的插件，在线编译下载适合自己系统架构的可执行文件  

#### 定制docker镜像部署
使用k8s、docker或者podman等工具部署第三方定制docker[镜像](https://ghcr.io/di-gigen/caddy)。您很容易在开源社区找到安全透明、文档友好的仓库，直接使用或者拷贝一份私有化部署都可以。容器化部署能带来更安全的数据和权限隔离，但这也要求您对容器技术有所了解。例如您需要额外设置持久化存储Caddy容器的配置文件、证书等重要文件的路径；并且根据管理工具的差异，容器运行进程守护的实现方式也有所不同。  

#### 手动编译
根据[官方文档](https://caddyserver.com/docs/build)自行编译。首先确保您的系统安装了官方编译工具xcaddy，不同的Linux发行版在操作上略有区别。配置好开发环境，对照需求表在编译指令中追加集成模块：  
```shell
xcaddy build --with github.com/caddy-dns/cloudflare --with github.com/greenpau/caddy-security --with github.com/mholt/caddy-dynamicdns --with github.com/caddyserver/transform-encoder --with github.com/corazawaf/coraza-caddy/v2
```

### 参数配置  
下方提供了一个几乎是最小化的配置模板，补充实际网络资产参数即可直接套用，您可以根据注释大致了解各配置块存在的意义  
<details>
<summary><font color="#E02222">/etc/caddy/Caddyfile</font></summary>

```
{
    order coraza_waf first
    order authenticate before respond
    order authorize before basicauth
    https_port 8443
    admin off
    acme_dns cloudflare MY_API_TOKEN
    email my@email.com
    log {
        level error
        output file /log/path/sys.log
    }
    dynamic_dns { 
		provider cloudflare MY_API_TOKEN
		dynamic_domains # Scan through the configured domains
	}
    security {
        local identity store localdb {
            realm local
            path /etc/caddy/users.json
        }
        authentication portal myportal {
            crypto default token lifetime 3600
            crypto key sign-verify {env.JWT_SHARED_KEY}
            enable identity store localdb
            cookie domain mydomain.com
            ui { # 统一登录门户收录地址
                links {
                    "Service1" https://sub1.mydomain.com:8443 icon "las la-server"
                    "Service2" https://sub2.mydomain.com:8443 icon "las la-server"
                    "Portal Settings" /settings icon "las la-cog"
                }
            }
            transform user {
                match realm local
                require mfa
                action add role authp/user
            }
        }
        authorization policy users_policy { # 定义跳转到SSO的安全策略
            set auth url https://auth.mydomain.com:8443/
            allow roles authp/admin authp/user
            crypto key verify {env.JWT_SHARED_KEY}
            acl rule {
                comment allow users
                match role authp/user
                allow stop log info
            }
            acl rule {
                comment default deny
                match any
                deny log warn
            }
        }
    }
}
*.mydomain.com {
    encode gzip zstd
    log {
        output file /etc/caddy/access.log 
        format console
    }
    coraza_waf { # WAF使用OWASP规则集
        load_owasp_crs
        directives `
            Include @coraza.conf-recommended
            Include @crs-setup.conf.example
            Include @owasp_crs/*.conf
            SecRuleEngine On
        `
    }
    @auth host auth.mydomain.com # SSO门户网站
    handle @auth {
        authenticate with myportal
    }
    @service1 host sub1.mydomain.com # 内网服务1
    handle @sub1 {
        route {
            authorize with users_policy # 接入SSO鉴权
            reverse_proxy IP:PORT
        }
    }
    @service2 host sub2.mydomain.com # 内网服务2
        handle @sub2 {
            reverse_proxy IP:PORT
        }
    handle {
        abort # Fallback for otherwise unhandled domains
    }
}
```
</details>

该配置文件默认应存放在`/etc/caddy`路径下被Caddy读取  
