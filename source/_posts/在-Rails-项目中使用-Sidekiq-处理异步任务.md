---
title: 在 Rails 项目中使用 Sidekiq 处理异步任务
date: 2019-11-21 22:00:52
tags: [Rails, Sidekiq]
category: Ruby
---

Sidekiq 是一个用于后台作业的处理框架，它提供了执行 **定时** 或 **异步** 任务的后台任务处理系统。可以集成在 Rails 项目中使用，也可以单独的使用。通常用来处理耗时的任务，如上传图像，发送邮件等。它可以配置多个队列，将不同类型的任务分配到各自的队列中。Sidekiq 会将一些运行时的数据存放在 Redis 中，所以请先确保你安装了 Redis。在本篇文章中，我们将对如何在 Rails 中使用 Sidekiq 进行介绍和分析。

<!-- more -->

### 添加依赖
目前 Sidekiq 已经更新到 6.0 版本了，这里我们使用最新的版本
```Shell
$ gem install sidekiq
```
```Ruby
gem 'sidekiq', '~> 6.0', '>= 6.0.3'
```

### 修改配置
添加 `config/sidekiq.yml` 文件
```YML
# 最大并发数
concurrency: 5

# 进程和日志的输出位置
pidfile: tmp/pids/sidekiq.pid
logfile: log/sidekiq.log

# 任务队列
queues:
  - default
  - schedule

development:
  concurrency: 5

staging:
  concurrency: 10

production:
  concurrency: 20
```
添加 `config/initializers/sidekiq.rb` 文件
```Ruby
# 设置 sidekiq cron 每秒钟检查一次任务，默认是 15 秒钟检查一次任务
Sidekiq.options[:poll_interval]                   = 1
Sidekiq.options[:poll_interval_average]           = 1

# 设置 sidekiq 服务端
Sidekiq.configure_server do |config|

  # 配置 redis 连接
  config.redis = { url: "redis://localhost:6379/0" }

  # 设置 sidekiq 每秒钟都会检查一次作业（默认每5秒检查一次）
  config.average_scheduled_poll_interval = 1

  # 从配置文件中读取定时任务
  schedule_file = 'config/sidekiq_schedule.yml'
  if File.exists?(schedule_file)
    Sidekiq::Cron::Job.load_from_hash YAML.load_file(schedule_file)
  end
end

Sidekiq.configure_client do |config|
  config.redis = { url: "redis://localhost:6379/0" }
end
```
在 `config/application.rb` 中设置配置项：
```Ruby
config.active_job.queue_adapter = :sidekiq # 使用 sidekiq 作为异步任务的适配器
```

### 异步任务
使用生成器来创建一个发送消息的任务：
```Shell
$ rails g job SendMessage
```
上面的命令将会创建 `app/jobs/send_message_job.rb`
```Ruby
class SendMessageJob < ActiveJob::Base
  queue_as :default
  def perform(*args)
    Rails.logger.info "send message..."
  end
end
```
如何使用：
```Ruby
SendMessageJob.perform_async(xxx, xxx, xxx) # 创建异步作业
SendMessageJob.perform_in(1.minutes, xxx, xxx) # 创建延时异步作业
SendMessageJob.perform_at(1.minutes.from_now, xxx, xxx) # 指定时间创建异步作业
```

### 定时任务
添加 `app/worker/send_message_worker.rb` 文件
```Ruby
class SendMessageWorker
  include Sidekiq::Worker
  sidekiq_options queue: :schedule, backtrace: true, retry: false
  # 无需显示调用，sidekiq 运行后会自动执行
  # 传入参数和执行周期在 config/sidekiq_schedule.yml 中配置
  def perform(*args)
      Rails.logger.info 'every second execution...'
  end
end
```
添加 `config/sidekiq_schedule.yml` 配置文件
```YML
SendMessageWorker:
  cron: '* * * * * *'
  class: SendMessageWorker
```
通过 `sidekiq -C config/sidekiq` 命令启动，定时任务将会周期的执行

### 配置监控
sidekiq 在 4.2 以上版本已经内置了一套监控页面，在 4.2 以前的版本需要添加额外的依赖：
```Ruby
gem 'sinatra', '~> 2.0', '>= 2.0.7'
```
在 `config/router.rb` 文件中挂载 WebUI 页面的路由。
```Ruby
require 'sidekiq/web'
Rails.application.routes.draw do
  mount Sidekiq::Web => '/sidekiq'
end
```
`rails s` 启动项目 `sidekiq -C config/sidekiq.yml` 启动 sidekiq，访问 [http://localhost:3000/sidekiq](http://localhost:3000/sidekiq)

### 配置访问监控的权限
在生产环境中，需要对监控页面添加权限的验证。这里提供几种简单的基于 HTTP Basic 的验证方式。
```Ruby
require 'sidekiq/web'
Rails.application.routes.draw do
    Sidekiq::Web.use Rack::Auth::Basic do |username, password|
        ActiveSupport::SecurityUtils.secure_compare(Digest::SHA256.hexdigest(username), Digest::SHA256.hexdigest('admin')) &
        ActiveSupport::SecurityUtils.secure_compare(Digest::SHA256.hexdigest(password), Digest::SHA256.hexdigest('123456'))
    end
    mount Sidekiq::Web, at: "/sidekiq"
end
```

### 参考文献
* [https://github.com/mperham/sidekiq](https://github.com/mperham/sidekiq)
* [https://draveness.me/sidekiq](https://draveness.me/sidekiq)
* [https://ruby-china.org/topics/36771](https://ruby-china.org/topics/36771)
