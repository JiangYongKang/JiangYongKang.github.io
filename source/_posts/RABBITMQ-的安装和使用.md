---
title: RabbitMQ 的安装和使用
date: 2022-11-02 21:32:47
tags: RabbitMQ
categories: 中间件
---

**RabbitMQ** 是实现了高级消息队列协议（AMQP）的开源消息代理软件（亦称面向消息的中间件）。RabbitMQ 服务器是用 **Erlang** 语言编写的，而集群和故障转移是构建在开放电信平台框架上的。所有主要的编程语言均有与代理接口通讯的客户端库。

<!-- more -->

### 安装 Erlang

由于 RabbitMQ 是基于 Erlang（面向高并发的语言）语言开发，所以在安装 RabbitMQ 之前，需要先安装 Erlang。

```SHELL
$ apt-get install erlang-nox
```

查看 Erlang 的版本信息

```SHELL
$ erl -version
Erlang (SMP,ASYNC_THREADS) (BEAM) emulator version 10.6.4
```

### 安装 RabbitMQ

Erlang 安装完成之后，就可以安装 RabbitMQ 了

```SHELL
$ apt-get install rabbitmq-server
```

安装完成后可以通过 service 来查看 RabbitMQ 的服务运行状态。可以看到 RabbitMQ 已经运行起来了，绑定的端口是 `5672`。

```SHELL
$ service rabbitmq-server status
● rabbitmq-server.service - RabbitMQ Messaging Server
     Loaded: loaded (/lib/systemd/system/rabbitmq-server.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-11-02 19:36:49 CST; 2h 11min ago
   Main PID: 1231564 (beam.smp)
     Status: "Initialized"
      Tasks: 87 (limit: 4654)
     Memory: 84.9M
     CGroup: /system.slice/rabbitmq-server.service
             ├─1231547 /bin/sh /usr/sbin/rabbitmq-server
             ├─1231564 /usr/lib/erlang/erts-10.6.4/bin/beam.smp -W w -A 64 -MBas ageffcbf -MHas ageffcbf -MBlmbcs 512 -MHlmbcs 512 -MMmcs 30 -P 1048576 -t 5000000 -stbt db -zdbbl 128000 -K true -- -root /usr/lib/erlan>
             ├─1231832 erl_child_setup 65536
             ├─1231857 inet_gethost 4
             └─1231858 inet_gethost 4

Nov 02 19:36:44 vincent-ecs systemd[1]: Starting RabbitMQ Messaging Server...
Nov 02 19:36:49 vincent-ecs systemd[1]: rabbitmq-server.service: Supervising process 1231564 which is not our child. We'll most likely not notice when it exits.
Nov 02 19:36:49 vincent-ecs systemd[1]: Started RabbitMQ Messaging Server.
```

### 开启 Web 端插件

安装插件之前需要使用到 **rabbitmq-plugins** 命令，rabbitmq-plugins 是管理 RabbitMQ 插件的命令行工具。用于启用、禁用和浏览插件。这些操作必须要由具有对 RabbitMQ 配置目录可写权限的用户执行。一些插件依赖于其他的插件才能正常工作，rabbitmq-plugins 会自动遍历这些依赖关系并且启用所有必需的插件。在 rabbitmq-plugins 命令行中列出的插件被标记为显式启用。依赖插件被标记为隐式启用。隐式启用的插件，在他们不需要的时候，在不再需要时会自动禁用。

查看插件列表：

```SHELL
vincent-ecs# rabbitmq-plugins list
Listing plugins with pattern ".*" ...
 Configured: E = explicitly enabled; e = implicitly enabled
 | Status: * = running on rabbit@vincent-ecs
 |/
...
[E*] rabbitmq_management               3.8.2
[e*] rabbitmq_management_agent         3.8.2
...
```

开启 **rabbitmq_management** 插件，用于 Web 端访问

```SHELL
$ rabbitmq-plugins enable rabbitmq_management
```

Web 控制台插件绑定的默认端口是 `15672`，默认的用户名和密码是 `guest` `guest`，但是这个账户仅能在本机访问，如果是在服务器上部署的还需要创建一个新的用户。

### RabbitMQ 用户管理

**rabbitmqctl** 命令可以用来管理 RabbitMQ 节点，同时也包含了用户的管理，用户管理相关的命令如下：
添加一个用户：

```SHELL
$ rabbitmqctl add_user username password
```

删除一个用户

```SHELL
$ rabbitmqctl delete_user username
```

修改用户的密码

```SHELL
$ rabbitmqctl change_password username new_password
```

查看当前用户列表

```SHELL
$ rabbitmqctl list_users
```

设置用户权限

```SHELL
$ rabbitmqctl set_user_tags username tags(administrator，monitoring，policymaker，management)
```
