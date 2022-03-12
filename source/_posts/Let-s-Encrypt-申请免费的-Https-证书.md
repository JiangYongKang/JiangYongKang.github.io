---
title: Let's Encrypt 申请免费的 Https 证书
date: 2019-01-01 15:49:07
tags:
  - Https
  - 域名
categories: 服务端运维
---

Let's Encrypt 证书不仅是免费的，而且支持通配符证书，通配符证书指的是一个可以被多个子域名使用的公钥证书，多个子域名使用起来十分方便。申请和配置的流程都非常简单，虽然每次的有效期为 90 天，但可以通过脚本去更新证书，只要配置好了，几乎可以一劳永逸。而市场上其他的通配符证书都比较昂贵，个人开发者平时做个小东西玩玩，Let's Encrypt 应该是最好的选择了。

<!-- more -->

### Certbot
[certbot](https://github.com/certbot/certbot) 可以通过简单的命令来生成证书，我们需要先将 certbot 克隆到我们的服务器中。
```shell
$ git clone https://github.com/certbot/certbot
```
```shell
$ cd certbot
```

### 申请证书
需要提到的一点是，客户在申请 Let’s Encrypt 证书的时候，需要校验域名的所有权，证明操作者有权利为该域名申请证书，目前支持三种验证方式：
- dns-01：给域名添加一个 DNS TXT 记录。
- http-01：在域名对应的 Web 服务器下放置一个 HTTP well-known URL 资源文件。
- tls-sni-01：在域名对应的 Web 服务器下放置一个 HTTPS well-known URL 资源文件。

而通配符域名只能通过 dns-01 的方式去申请，我是通过阿里云购买的域名，需要登录阿里云在解析设置中添加解析记录，后面会提到如何添加TXT解析记录。使用下面的命令开始生成证书，注意将 `*.example.com` 和 `example.com` 替换成你自己的域名。

```shell
$ certbot-auto certonly --manual \
-d *.example.com \
-d example.com --agree-tos \
--manual-public-ip-logging-ok --preferred-challenges \
dns-01 --server https://acme-v02.api.letsencrypt.org/directory
```

输入完上面的命令之后，会开始下载一大堆依赖库，至于是什么东西，我也不太清楚，耐心等待依赖文件下载完成即可。之后便会提示你输入邮箱：

```shell
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Enter email address (used for urgent renewal and security notices) (Enter 'c' to
cancel): xxxxxxx@email.com
```

当你输入完正确的邮箱之后，需要验证域名的所有权，如下：

```shell
Please deploy a DNS TXT record under the name
_acme-challenge.example.com with the following value:

mhumL1xJOHPIZtFTEm4rotjJnR9TdkBVPuCS9YHvNjs

Before continuing, verify the record is deployed.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
```

此时打开你的域名提供商去添加解析记录，我的域名是阿里云购买的。其他域名提供商应该也是一致的。记录类型选择 **TXT**，主机记录输入上面的 **_acme-challenge.example.com**，记录值输入上面生成的随机字符串 **mhumL1xJOHPIZtFTEm4rotjJnR9TdkBVPuCS9YHvNjs** 。

安装一个工具，用于验证 **TXT** 解析是否生效：
```shell
$ yum install bind-utils
```

```Text
$ dig -t txt _acme-challenge.example.com @8.8.8.8

; <<>> DiG 9.9.4-RedHat-9.9.4-72.el7 <<>> -t txt _acme-challenge.example.com @8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29355
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;_acme-challenge.example.com. IN  TXT

;; ANSWER SECTION:
_acme-challenge.example.com. 599 IN TXT   "1scXnCO43OgpWRkdaVpTb-_vd2NGHwdmJEmQhvRC6AA"

;; Query time: 317 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Tue Jan 01 12:30:15 CST 2019
;; MSG SIZE  rcvd: 118
```


有可能会提示需要再次验证，如下所示：

```Text
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name
_acme-challenge.example.com with the following value:

1scXnCO43OgpWRkdaVpTb-_vd2NGHwdmJEmQhvRC6AA

Before continuing, verify the record is deployed.
(This must be set up in addition to the previous challenges; do not remove,
replace, or undo the previous challenge tasks yet. Note that you might be
asked to create multiple distinct TXT records with the same name. This is
permitted by DNS standards.)

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

不过没关系，依照上面的步骤再做一次即可，如果不出意外，你能看到下面的输出：

```Text
Press Enter to Continue
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/example.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/example.com/privkey.pem
   Your cert will expire on 2019-04-01. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot-auto
   again. To non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

生成的的证书和秘钥以及过期时间都已经打印出来了，妥善保管。

### 配置 Https 访问
如果你使用的是 nginx，那么配置起来很简单：
```nginx

# 设置 http 自动跳转到 https
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;                                      
}

# 监听 443 端口，转发请求到 3000 端口
server {
    listen 443;
    server_name example.com;
    location / {
        proxy_pass http://127.0.0.1:3000;
    }

    # 开启 ssl 并指定证书文件和秘钥的位置
    ssl on;
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;        
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;          
}                                                                                     
```
