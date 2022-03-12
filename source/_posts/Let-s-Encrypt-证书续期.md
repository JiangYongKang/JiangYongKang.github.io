---
title: 'Let''s Encrypt 证书续期 '
date: 2019-03-21 15:49:33
tags:
  - Https
  - 域名
categories: 服务端运维
---
### 查看证书有效期

Let's Encrypt 默认情况下只提供三个月的有效期，在有效期剩余半个月的时候，Let's Encrypt 会发送邮件给你，提醒你需要做证书的续期操作。或者我们也可以通过以下命令查看证书的剩余有效期限：

```shell
$ ./certbot-auto certificates
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Found the following certs:
  Certificate Name: example.com
    Domains: *.example.com example.com
    Expiry Date: 2019-06-19 09:37:01+00:00 (VALID: 89 days)
    Certificate Path: /etc/letsencrypt/live/example.com/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/example.com/privkey.pem
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

<!-- more -->

### 手动续期

如果你使用的是通配符域名，那么很不幸，你讲无法使用 `certbot-auto renew` 方法进行续期。我们可以尝试手动去续期，命令如下：

```shell
$ ./certbot-auto --server https://acme-v02.api.letsencrypt.org/directory \
-d "*.example.com" -d "example.com" \
--manual --preferred-challenges dns-01 certonly
```

已下部分是执行的日志

```shell
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Cert is due for renewal, auto-renewing...
Renewing an existing certificate
Performing the following challenges:
dns-01 challenge for example.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
NOTE: The IP of this machine will be publicly logged as having requested this
certificate. If you're running certbot in manual mode on a machine that is not
your server, please ensure you're okay with that.

Are you OK with your IP being logged?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name
_acme-challenge.example.com with the following value:

mnDglnRF3P0VCEW6xoIDYblcswOJySkc3CPAQIwFm-c

Before continuing, verify the record is deployed.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
```

执行到这里时，我们就要开始添加记录了。打开你的域名提供商，添加一条 TXT 类型的记录。主机记录是上面打印出来的 `_acme-challenge.example.com` 记录值为 `mnDglnRF3P0VCEW6xoIDYblcswOJySkc3CPAQIwFm-c`。填写完毕之后，稍等一分钟，回车继续，不出意外的话，续期成功了。当然，你仍然需要重启 `nginx`。

```shell
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/example.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/example.com/privkey.pem
   Your cert will expire on 2019-06-19. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot-auto
   again. To non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```
