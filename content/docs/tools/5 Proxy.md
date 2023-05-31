---
title: "Proxy"
weight: 5
---

2017.7 - 2019.6: `Shadowsocks`  
2019.6 - 2023.4: `Vmess + TLS + WS + Nginx`  
2023.4 - now: `NavieProxy + Caddy`

以下操作基于`CentOS 7`操作系统，只记录关键步骤

## Shadowsocks

### 服务端

安装`pip`
```bash
yum install python-pip
```

检查`pip`版本
```bash
pip -V
```

安装`Shadowsocks`，目前`PyPI`中的最新版本停留在了`Released: Aug 10, 2015`的`2.8.2`版本
```bash
pip install shadowsocks
```

创建和编辑`/etc/shadowsocks.json`，填写如下内容
```json
{
  "server": "0.0.0.0",
  "server_port": 4444,
  "password": "xxxx",
  "timeout": 600,
  "method": "aes-256-cfb",
  "fast_open": "true"
}
```

启动服务并设置开机自起，将如下命令写入到/etc/rc.local（exit 0之前）
```bash
ssserver -c /etc/shadowsocks.json -d start
```

### 客户端

`GUI`客户端
- [shadowsocks-android](https://github.com/shadowsocks/shadowsocks-android): Android client.
- [shadowsocks-windows](https://github.com/shadowsocks/shadowsocks-csharp): Windows client.
- [shadowsocksX-NG](https://github.com/shadowsocks/ShadowsocksX-NG): MacOS client.
- [shadowsocks-qt5](https://github.com/shadowsocks/shadowsocks-qt5): Cross-platform client for Windows/MacOS/Linux.

客户端配置暂略

References
> https://shadowsocks.org/guide/getting-started.html
> https://pypi.org/project/shadowsocks/#history

## Vmess + TLS + WS + Nginx

### 依赖

需要具备域名，以v2.example.com为例，并配置了A记录解析到了VPS的IP，如`45.76.190.133`

### 服务端

下载`v2ray`
```bash
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```

编辑`v2ray`配置文件`vi /usr/local/etc/v2ray/config.json`写入以下内容，并启动`v2ray`
```json
{
  "inbounds": [
    {
      "port": 18967,
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "54bb75fe-e973-4fa1-8390-a4fc95f96ec2",
            "level": 1,
            "alterId": 64
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
          "path": "/videos/"
        }
      }
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
    "rules": [
      {
        "type": "field",
        "ip": [
          "geoip:private"
        ],
        "outboundTag": "blocked"
      }
    ]
  }
}
```
```bash
systemctl enable v2ray
systemctl start v2ray
```

安装并启动`nginx`
```bash
yum install epel-release
yum install nginx
systemctl enable nginx
systemctl start nginx
```

编辑nginx`v2.example.com`站点配置文件`vi /etc/nginx/conf.d/v2.example.com.conf`，用于转发`/vidoes/`路径请求至`v2ray`
```conf
server {
    listen 80;
    root /usr/share/nginx/html;
    server_name v2.example.com;

    location / {
            root   html;
            index  index.html index.htm;
    }

    location /videos/ {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:18967;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
    }
}
```

重启`nginx`
```bash
systemctl restart nginx
```

安装`certbot-nginx`Nginx Plugin
```bash
yum install certbot-nginx
```

证书申请及配置，同时确认关闭防火墙
```bash
systemctl stop firewalld
systemctl disable firewalld
certbot --nginx -d v2.example.name
```

验证，浏览器访问`https://v2.example.com/videos/`返回`Bad Request`即说明服务器端安装成功

### 客户端

`GUI`客户端
- [V2RayN](https://github.com/2dust/v2rayN) 是一个基于 V2Ray 内核的 Windows 客户端
- [V2RayX](https://github.com/Cenmrev/V2RayX) 是一个基于 V2Ray 内核的 Mac OS X 客户端
- [Shadowrocket](https://itunes.apple.com/us/app/shadowrocket/id932747118?mt=8) 是一个通用的 iOS VPN 应用，它支持众多协议，如 Shadowsocks、VMess、SSR 等
- [V2RayNG](https://github.com/2dust/v2rayNG) 是一个基于 V2Ray 内核的 Android 应用


References
> https://www.v2ray.com/awesome/tools.html  
> https://github.com/v2fly/v2ray-core


## NavieProxy + Caddy

### 依赖

需要具备域名，以`v2.example.com`为例，并配置了A记录解析到了VPS的IP，如`45.76.190.133`

需要安装Go

### 服务端

生成证书，输入`v2.example.com`
```bash
certbot certonly
```

安装最新版本Go环境
```bash
wget "https://go.dev/dl/$(curl https://go.dev/VERSION?m=text).linux-amd64.tar.gz"
tar -xf go*.linux-amd64.tar.gz -C /usr/local/

echo 'export GOROOT=/usr/local/go' >> /etc/profile
echo 'export PATH=/root/go/bin:$GOROOT/bin:$PATH' >> /etc/profile
source /etc/profile

go
```

`Caddy`安装
```bash
yum install yum-plugin-copr
yum copr enable @caddy/caddy
yum install caddy
```

使用`XCaddy`编译`Caddy NavieProxy`插件，并替换原有`caddy`文件
```bash
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
xcaddy build --with github.com/caddyserver/forwardproxy@caddy2=github.com/klzgrad/forwardproxy@naive

./caddy list-modules # 显示 http.handlers.forward_proxy，Non-standard modules: 1

mv caddy /usr/bin/caddy
```

编辑JSON格式配置文件`vi /etc/caddy/server.json`，使用了`6443`自定义端口
```json
{
   "admin":{
      "disabled":true
   },
   "apps":{
      "http":{
         "servers":{
            "srv0":{
               "listen":[
                  ":6443"
               ],
               "routes":[
                  {
                     "handle":[
                        {
                           "handler":"subroute",
                           "routes":[
                              {
                                 "handle":[
                                    {
                                       "auth_user_deprecated":"zqq",
                                       "auth_pass_deprecated":"123456",
                                       "handler":"forward_proxy",
                                       "hide_ip":true,
                                       "hide_via":true,
                                       "probe_resistance":{
                                          
                                       }
                                    }
                                 ]
                              },
                              {
                                 "match":[
                                    {
                                       "host":[
                                          "v2.example.com"
                                       ]
                                    }
                                 ],
                                 "handle":[
                                    {
                                       "handler":"file_server",
                                       "root":"/usr/share/caddy",
                                       "index_names":[
                                          "index.html"
                                       ]
                                    }
                                 ],
                                 "terminal":true
                              }
                           ]
                        }
                     ]
                  }
               ],
               "tls_connection_policies":[
                  {
                     "match":{
                        "sni":[
                           "v2.example.com"
                        ]
                     }
                  }
               ],
               "automatic_https":{
                  "disable":true
               }
            }
         }
      },
      "tls":{
         "certificates":{
            "load_files":[
               {
                  "certificate":"/etc/letsencrypt/live/v2.example.com/fullchain.pem",
                  "key":"/etc/letsencrypt/live/v2.example.com/privkey.pem"
               }
            ]
         }
      }
   }
}
```
格式化配置文件
```bash
caddy fmt --overwrite /etc/caddy/server.json
```


修改`systemd caddy`配置`vi /usr/lib/systemd/system/caddy.service`
```ini
User=root
Group=root
ExecStart=/usr/bin/caddy run --environ --config /etc/caddy/server.json
ExecReload=/usr/bin/caddy reload --config /etc/caddy/server.json --force
```

启动`Caddy`
```bash
systemctl daemon-reload
systemctl enable caddy
systemctl start caddy
```

### 客户端

使用[Qv2ray](https://github.com/Qv2ray/Qv2ray)搭配[NavieProxy插件](https://github.com/Qv2ray/QvPlugin-NaiveProxy)

Qv2ray需要指定[v2ray-core](https://github.com/v2fly/v2ray-core/releases/tag/v4.45.2)目录位置，NavieProxy插件放在Qv2ray插件目录中，NavieProxy插件需要指定[navie](https://github.com/klzgrad/naiveproxy/releases/tag/v112.0.5615.49-1)客户端执行文件位置