+++
author = "Will"
title = "v2ray+nginx+ws+tls's single port with multiple users including traffic statistics"
date = "2020-09-19"
description = "How to depoly v2ray+nginx+ws+tls configuration of a single port but multiple users with traffic statistics "
tags = [
    "vps",
    "bash",
    "v2ray",
]
categories = [
    "vps",
    "linux",
    "v2ray",
]
image="v2ray.jpg"
comments = "true"
+++
<!--more-->
# v2ray+nginx+ws+tls实现单端口多用户流量统计

本文重在v2ray+nginx+ws+tls单端口多用户并且带有流量查询功能的配置，因此对于v2ray+nginx+ws+tls的安装直接使用了wulabing大佬的一键脚本。故之后将基于wulabing一键脚本得到的v2ray配置文件config.json进行说明。

ssh客户端：FinalShell

## 一键脚本搭建v2ray+nginx+ws+tls
首先域名解析到自己的ip上待解析生效后，用一键脚本按照提示安装完成v2ray+nginx+ws+tls。
```bash
bash <(curl -L -s https://raw.githubusercontent.com/wulabing/V2Ray_ws-tls_bash_onekey/master/install.sh) | tee v2ray_ins.log
```
安装完成后,开始v2ray配置文件的修改

## 命令行下生成随机uuid
```bash
cat /proc/sys/kernel/random/uuid 
```

![w5L9vd.png](https://s1.ax1x.com/2020/09/19/w5L9vd.png)

将生成的uuid先复制到其它地方备用

## v2ray原始配置文件备份
使用cat命令，将v2ray的配置文件显示出来，并复制到本地作为备份，其中有有效信息需要更改

```bash
cat /etc/v2ray/config.json
```

将文件里的内容备份到本地
```json
{
  "log": {
        "access": "/var/log/v2ray/access.log",
        "error": "/var/log/v2ray/error.log",
        "loglevel": "warning"
    },
  "inbounds": [
    {
    "port":33145,
      "listen": "127.0.0.1", 
      "tag": "vmess-in", 
      "protocol": "vmess", 
      "settings": {
        "clients": [
          {
          "id":"af7482b7-4ab9-49e6-bd10-ebbf229cc4d1",
          "alterId":2
          }
        ]
      }, 
      "streamSettings": {
        "network": "ws", 
        "wsSettings": {
          "path":"/982546d18747e3/"
        }
      }
    }
  ], 
  "outbounds": [
    {
      "protocol": "freedom", 
      "settings": { }, 
      "tag": "direct"
    }, 
    {
      "protocol": "blackhole", 
      "settings": { }, 
      "tag": "blocked"
    }
  ], 
  "dns": {
    "servers": [
      "https+local://1.1.1.1/dns-query",
          "1.1.1.1",
          "1.0.0.1",
          "8.8.8.8",
          "8.8.4.4",
          "localhost"
    ]
  },
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "type": "field",
        "inboundTag": [
          "vmess-in"
        ],
        "outboundTag": "direct"
      }
    ]
  }

```
**! 注意**，该配置中之后需要应用的新配置中的关键信息包括**inbounds**模块中的**port**值，以及**websocket的路径**

## 单端口多用户带流量统计的配置文件
```json
{
    "stats": {},
    "log": {
        "access": "/var/log/v2ray/access.log",
        "error": "/var/log/v2ray/error.log",
        "loglevel": "debug"
    },
    "api": {
        "tag": "api",
        "services": [
            "HandlerService",
            "LoggerService",
            "StatsService"
        ]
    },
    "policy": {
        "levels": {
            "0": {
                "statsUserUplink": true,
                "statsUserDownlink": true
            },
            "1": {
                "statsUserUplink": true,
                "statsUserDownlink": true
            }
        },
        "system": {
            "statsInboundUplink": true,
            "statsInboundDownlink": true
        }
    }, 
    "inbounds": [
        {
            "tag": "vmess-in",
            "port": 33145,
            "listen": "127.0.0.1",
            "protocol": "vmess",
            "settings": {
                "clients": [
                    {
                        "email": "aaa@google.com",
                        "id": "af7482b7-4ab9-49e6-bd10-ebbf229cc4d1",
                        "level": 1,
                        "alterId": 2
                    },
                    {
                        "email": "bbb@google.com",
                        "id": "66b3973c-fa49-11ea-adc1-0242ac120002",
                        "level": 1,
                        "alterId": 2                   },
                    {
                        "email": "ccc@google.com",
                        "id": "c9eddeb1-912d-4b43-b2a6-cad666a48a1e",
                        "level": 1,
                        "alterId": 2
                    }
                ]
            },
            "streamSettings": {
            "network": "ws", 
            "wsSettings": {
	        "path":"/982546d18747e3/"
        }
      }
        },
        {
            "listen": "127.0.0.1",
            "port": 10085,
            "protocol": "dokodemo-door",
            "settings": {
                "address": "127.0.0.1"
            },
            "tag": 
 "api"
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom",
            "settings": {}
        },
        {
            "protocol": "blackhole",
            "settings": {},
            "tag": "blocked"
 
        }
    ],
    "routing": {
        "settings": {
            "rules": [
                {
                    "inboundTag": [
                        "api"
                    ],
                    "outboundTag": "api",
                    "type": "field"
                }
            ]
        },
        "strategy": "rules"
    }
}
```

这份配置文件可以视作一个标准的配置模板，为了与符合原配置文件和nginx的交流规则，你所需要改动的地方是：
- **inbounds**模块中的**port**值，本例中为33145
- 以及**websocket的路径**，本例中为/982546d18747e3/
- 用户模块，本例中给出了aaa@google.com, bbb@google.com, ccc@google.com三位用户，借以不同的uuid区分他们的不同身份，email的名称将在之后用于查询各用户所使用的流量情况，你也可以自己新增用户。
- **一个需要注意的地方是在新增一个用户时，需要在上一个用户模块的结尾增添逗号**

一个用户模块的示例：
```json
{
                        "email": "ccc@google.com",
                        "id": "c9eddeb1-912d-4b43-b2a6-cad666a48a1e",
                        "level": 1,
                        "alterId": 2
                    }
```

uuid是确立身份的核心，而email则是我们方便的辨识的一个别名。

## 新配置文件覆盖旧配置文件
1. 先删除原文件的所有内容

```bash
vi /etc/v2ray/config.json
```
vi命令 __ggdG__ 会删除文件内的所有内容

2. 修改好的新配置覆盖

复制按照上述说明已经修改好的配置文件，粘贴到v2ray的配置文件里
然后**输入:wq** 保存退出
ps: 推荐小白用finalshell cd到config.json目录下直接在下面打开该文件进行删除粘贴再保存的操作，也可以用nano,界面有详细的操作说明
然后**输入:wq**保存退出

3. 重启v2ray服务
```bash
systemctl restart v2ray
```
这样v2ray已经成功实现了单端口多用户的配置，整个过程不需要对nginx进行处理
导入vmess链接后，可以先克隆该链接，再更改uuid顺便换个名称便于区分
![w5jzdI.png](https://s1.ax1x.com/2020/09/19/w5jzdI.png)
测试三个链接是否可以正常使用：

![wICCYn.png](https://s1.ax1x.com/2020/09/19/wICCYn.png)

测试显示均有速度，说明配置成功。
ps: 不要在意我图中的速度快慢，我只是临时开了一台vultr专门写这个教程

测试网页，三个链接也都可以打开谷歌，说明均是可用状态：
![wIVQET.png](https://s1.ax1x.com/2020/09/19/wIVQET.png)

## 单用户流量统计
查看某用户上传流量(单位字节)：
```bash
v2ctl api --server=127.0.0.1:10085  StatsService.GetStats 'name: "user>>>aaa@google.com>>>traffic>>>uplink" reset: false'
```

查看某用户下载流量(单位字节)：
```bash
v2ctl api --server=127.0.0.1:10085  StatsService.GetStats 'name: "user>>>bbb@google.com>>>traffic>>>downlink" reset: false'
```
解释一下换算关系：
>  字节（Byte)=8bit
>  Byte简写为B 
> bit简写为b
> 1KB=8Kb

> $1GB=10^3 MB= 10^6 KB =10^9 Byte$
按照换算关系计算一下消耗的流量
测试上传：
![wIC1l6.png](https://s1.ax1x.com/2020/09/19/wIC1l6.png)
测试下载：
![wIC1l6.png](https://s1.ax1x.com/2020/09/19/wIC1l6.png)
可看到正常工作
这些基本上够用了，当然如果想要方便的话可以使用v2fly官网提供的代码保存做一个固定的一键脚本
新建traffic.sh 文件，将以下内容粘贴到该文件中：
```bash
#!/bin/bash

_APISERVER=127.0.0.1:10085
_V2CTL=v2ctl

apidata () {
    local ARGS=
    if [[ $1 == "reset" ]]; then
      ARGS="reset: true"
    fi
    $_V2CTL api --server=$_APISERVER StatsService.QueryStats "${ARGS}" \
    | awk '{
        if (match($1, /name:/)) {
            f=1; gsub(/^"|link"$/, "", $2);
            split($2, p,  ">>>");
            printf "%s:%s->%s\t", p[1],p[2],p[4];
        }
        else if (match($1, /value:/) && f){ f = 0; printf "%.0f\n", $2; }
        else if (match($0, /^>$/) && f) { f = 0; print 0; }
    }'
}

print_sum() {
    local DATA="$1"
    local PREFIX="$2"
    local SORTED=$(echo "$DATA" | grep "^${PREFIX}" | sort -r)
    local SUM=$(echo "$SORTED" | awk '
        /->up/{us+=$2}
        /->down/{ds+=$2}
        END{
            printf "SUM->up:\t%.0f\nSUM->down:\t%.0f\nSUM->TOTAL:\t%.0f\n", us, ds, us+ds;
        }')
    echo -e "${SORTED}\n${SUM}" \
    | numfmt --field=2 --suffix=B --to=iec \
    | column -t
}

DATA=$(apidata $1)
echo "------------Inbound----------"
print_sum "$DATA" "inbound"
echo "-----------------------------"
echo
echo "-------------User------------"
print_sum "$DATA" "user"
echo "-----------------------------"
```

新建traffic.sh粘贴到其中保存即可
```bash
vi traffic.sh
```

授予traffic.sh执行权限
```bash
chmod 755 traffic.sh
```

执行：
```bash
./traffic.sh
```

![wIP2DO.png](https://s1.ax1x.com/2020/09/19/wIP2DO.png)

这样可以看到全部的统计信息，下方可以一次性看到用户的流量使用情况。
至此，本次教程的内容已完成，感谢wulabing, v2fly网站，loonlog.com提供的思路。

v2+ws+tls是当下最为安全的科学上网方式之一，然后对于该种配置单端口多用户并且成功实现流量监控网上的例子似乎寥寥无几，故在他人的基础上出此教程希望对小白们所帮助。



【参考资料】

[https://github.com/wulabing/V2Ray_ws-tls_bash_onekey](https://github.com/wulabing/V2Ray_ws-tls_bash_onekey)

[http://loonlog.com/2020/1/3/v2ray-more-id-more-port/](http://loonlog.com/2020/1/3/v2ray-more-id-more-port/)

[http://loonlog.com/2020/8/24/v2ray-traffic-statistics/](http://loonlog.com/2020/8/24/v2ray-traffic-statistics/)

[https://guide.v2fly.org/advanced/traffic.html#%E6%9F%A5%E7%9C%8B%E6%B5%81%E9%87%8F%E4%BF%A1%E6%81%AF](https://guide.v2fly.org/advanced/traffic.html#%E6%9F%A5%E7%9C%8B%E6%B5%81%E9%87%8F%E4%BF%A1%E6%81%AF)

