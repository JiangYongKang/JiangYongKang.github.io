---
title: Ruby Web 服务器的配置与使用
date: 2019-11-14 21:22:33
tags: Ruby
categories: Ruby
---

### Puma
Puma 是 Rails 的默认 Web Server，在创建 Rails 项目时已经自动添加了 Puma 的依赖。Puma 是用了多进程加多线程模型，它可以同时在 fork 出来的多个 worker 中创建多个线程来处理请求；不仅如此 Puma 还实现了用于提高并发速度的 Reactor 模块和线程池能够在提升吞吐量的同时，降低内存的消耗。

<!-- more -->

```Ruby
gem 'puma', '~> 3.7'
```
创建 `config/puma.rb` 配置文件，下面是文件内容：
```Ruby
# Puma can serve each request in a thread from an internal thread pool.
# The `threads` method setting takes two numbers: a minimum and maximum.
# Any libraries that use thread pools should be configured to match
# the maximum value specified for Puma. Default is set to 5 threads for minimum
# and maximum; this matches the default thread size of Active Record.
#
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count

# Specifies the `port` that Puma will listen on to receive requests; default is 3000.
#
port        ENV.fetch("PORT") { 3000 }

# Specifies the `environment` that Puma will run in.
#
environment ENV.fetch("RAILS_ENV") { "development" }

# Specifies the number of `workers` to boot in clustered mode.
# Workers are forked webserver processes. If using threads and workers together
# the concurrency of the application would be max `threads` * `workers`.
# Workers do not work on JRuby or Windows (both of which do not support
# processes).
#
# workers ENV.fetch("WEB_CONCURRENCY") { 2 }

# Use the `preload_app!` method when specifying a `workers` number.
# This directive tells Puma to first boot the application and load code
# before forking the application. This takes advantage of Copy On Write
# process behavior so workers use less memory. If you use this option
# you need to make sure to reconnect any threads in the `on_worker_boot`
# block.
#
# preload_app!

# If you are preloading your application and using Active Record, it's
# recommended that you close any connections to the database before workers
# are forked to prevent connection leakage.
#
# before_fork do
#   ActiveRecord::Base.connection_pool.disconnect! if defined?(ActiveRecord)
# end

# The code in the `on_worker_boot` will be called if you are using
# clustered mode by specifying a number of `workers`. After each worker
# process is booted, this block will be run. If you are using the `preload_app!`
# option, you will want to use this block to reconnect to any threads
# or connections that may have been created at application boot, as Ruby
# cannot share connections between processes.
#
# on_worker_boot do
#   ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
# end
#

# Allow puma to be restarted by `rails restart` command.
plugin :tmp_restart

```

### Unicorn
与 Thin 同年发布的 Unicorn 虽然也是 Mongrel 项目的一个 fork，但是使用了完全不同的并发模型，每Unicorn 内部通过多次 fork 创建多个 worker 进程，所有的 worker 进程也都由一个 master 进程管理和控制。由于 master 进程的存在，当 worker 进程被意外杀掉后会被 master 进程重启，能够保证持续对外界提供服务，多个进程的 worker 也能够充分的利用多核 CPU 的性能，尽可能地提高请求的处理速度。
```Ruby
gem 'unicorn', '~> 5.5', '>= 5.5.1'
```
创建 `config/unicorn.rb` 配置文件，下面是文件内容。
```Ruby
# 启动 unicorn
# $ unicorn
# 带配置文件的方式启动
# $ unicorn -c config/unicorn.rb
# 启动生产环境
# $ unicorn -c config/unicorn.rb --env production
# 后台运行
# $ unicorn -c config/unicorn.rb --env production --daemonize
# 也可以通过 unicorn_rails 启动. unicorn_rails 命令更适用于 rails 项目，unicorn 命令适用于非 rails 项目。
# $ unicorn_rails -c config/unicorn.rb

module Rails
  class <<self
    def root
      File.expand_path(__FILE__).split('/')[0..-3].join('/')
    end
  end
end

# Sample verbose configuration file for Unicorn (not Rack)
#
# This configuration file documents many features of Unicorn
# that may not be needed for some applications. See
# https://bogomips.org/unicorn/examples/unicorn.conf.minimal.rb
# for a much simpler configuration file.
#
# See https://bogomips.org/unicorn/Unicorn/Configurator.html for complete
# documentation.

# 设置 worker_processes 数量
# Use at least one worker per core if you're on a dedicated server,
# more will usually help for _short_ waits on databases/caches.
worker_processes 1

# 设置运行 worker processes 的用户和组
# Since Unicorn is never exposed to outside clients, it does not need to
# run on the standard HTTP port (80), there is no reason to start Unicorn
# as root unless it's from system init scripts.
# If running the master process as root and the workers as an unprivileged
# user, do this to switch euid/egid in the workers (also chowns logs):
# user "unprivileged_user", "unprivileged_group"

# 设置 unicorn 工作目录，SIGUSR2 将启动一个新的 unicorn 实例在这个目录。
# Help ensure your application will always spawn in the symlinked
# "current" directory that Capistrano sets up.
working_directory Rails.root # available in 0.94.0+

# 监听地址，可以使是tcp地址，也可以是UNIX domain sockets
# listen on both a Unix domain socket and a TCP port,
# we use a shorter backlog for quicker failover when busy
# TODO: listen to port 3000 on the IPv6 loopback interface
# listen "[::1]:3000"
# TODO: listen to port 3000 on the loopback interface
# listen "127.0.0.1:3000"
# TODO: listen on the given Unix domain socket
listen "#{Rails.root}/.unicorn.sock", :backlog => 64
# TODO: listen to port 3000 on all TCP interfaces
listen 3000, :tcp_nopush => true

# 设置 worker processes 的超时时间
# nuke workers after 30 seconds instead of 60 seconds (the default)
timeout 30

# 设置pid文件的位置
# feel free to point this anywhere accessible on the filesystem
pid "#{Rails.root}/tmp/pids/unicorn.pid"

# 设置 Logger 输出的位置
# By default, the Unicorn logger will write to stderr.
# Additionally, ome applications/frameworks log to stderr or stdout,
# so prevent them from going to /dev/null when daemonized here:
# 重定向 stderr 到指定文件
# stderr_path "#{Rails.root}/log/unicorn.stderr.log"
# 重定向 stdout 到指定文件
# stdout_path "#{Rails.root}/log/unicorn.stdout.log"

# 在 forking worker processes 之前预加载程序
# combine Ruby 2.0.0+ with "preload_app true" for memory savings
preload_app true

# 禁止 rewindability，可以提高上传的性能，降低 io 和内存使用
rewindable_input false

# 检查客户端链接是否断开，防止断开的请求调用。
# Enable this flag to have unicorn test client connections by writing the
# beginning of the HTTP headers before calling the application.  This
# prevents calling the application for connections that have disconnected
# while queued.  This is only guaranteed to detect clients on the same
# host unicorn runs on, and unlikely to detect disconnects even on a
# fast LAN.
check_client_connection false

# local variable to guard against running a hook multiple times
run_once = true

# 在 master fork work之前被调用。
before_fork do |server, worker|
  # the following is highly recomended for Rails + "preload_app true"
  # as there's no need for the master process to hold a connection
  defined?(ActiveRecord::Base) and
    ActiveRecord::Base.connection.disconnect!

  # Occasionally, it may be necessary to run non-idempotent code in the
  # master before forking.  Keep in mind the above disconnect! example
  # is idempotent and does not need a guard.
  if run_once
    # do_something_once_here ...
    run_once = false # prevent from firing again
  end

  # The following is only recommended for memory/DB-constrained
  # installations.  It is not needed if your system can house
  # twice as many worker_processes as you have configured.
  #
  # # This allows a new master process to incrementally
  # # phase out the old master process with SIGTTOU to avoid a
  # # thundering herd (especially in the "preload_app false" case)
  # # when doing a transparent upgrade.  The last worker spawned
  # # will then kill off the old master process with a SIGQUIT.
  # old_pid = "#{server.config[:pid]}.oldbin"
  # if old_pid != server.pid
  #   begin
  #     sig = (worker.nr + 1) >= server.worker_processes ? :QUIT : :TTOU
  #     Process.kill(sig, File.read(old_pid).to_i)
  #   rescue Errno::ENOENT, Errno::ESRCH
  #   end
  # end
  #
  # Throttle the master from forking too quickly by sleeping.  Due
  # to the implementation of standard Unix signal handlers, this
  # helps (but does not completely) prevent identical, repeated signals
  # from being lost when the receiving process is busy.
  # sleep 1
end

# TODO: worker fork 之后被调用。
after_fork do |server, worker|
  # per-process listener ports for debugging/admin/migrations
  # addr = "127.0.0.1:#{9293 + worker.nr}"
  # server.listen(addr, :tries => -1, :delay => 5, :tcp_nopush => true)

  # the following is *required* for Rails + "preload_app true",
  defined?(ActiveRecord::Base) and
    ActiveRecord::Base.establish_connection

  # if preload_app is true, then you may also want to check and
  # restart any other shared sockets/descriptors such as Memcached,
  # and Redis.  TokyoCabinet file handles are safe to reuse
  # between any number of forked children (assuming your kernel
  # correctly implements pread()/pwrite() system calls)
end
```

### Thin
Thin 使用 Reactor 模型处理客户端的 HTTP 请求，每一个请求都会交由 EventMachine，通过内部对事件的分发，最终执行相应的回调，这种事件驱动的 IO 模型与 node.js 非常相似，使用单进程单线程的并发模型却能够快速处理 HTTP 请求。
```Runy
gem 'thin', '~> 1.7', '>= 1.7.2'
```
Thin 提供了生成配置文件的功能，在项目根目录输入 `thin config -C config/thin.yml` 可以快速的创建一份配置文件。
```yml
---
chdir: "/Users/vincent/"
environment: development
address: 0.0.0.0
port: 3000
timeout: 30
log: "/Users/vincent/log/thin.log"
pid: tmp/pids/thin.pid
max_conns: 1024
max_persistent_conns: 100
require: []
wait: 30
threadpool_size: 20
daemonize: true
```

### 参考文献

* [http://www.mojidong.com/rails/2013/04/20/rails-nginx-unicorn/](http://www.mojidong.com/rails/2013/04/20/rails-nginx-unicorn/)
* [https://ruby-china.org/topics/4709](https://ruby-china.org/topics/4709)
* [https://bogomips.org/unicorn/](https://bogomips.org/unicorn/)
* [https://read01.com/zh-hk/zm5B.html#.Xc1XC-u-t0t](https://read01.com/zh-hk/zm5B.html#.Xc1XC-u-t0t)
* [https://ruby-china.org/topics/25276](https://ruby-china.org/topics/25276)
* [https://ruby-china.org/topics/10832](https://ruby-china.org/topics/10832)
* [https://stackoverflow.com/questions/4113299/ruby-on-rails-server-options](https://stackoverflow.com/questions/4113299/ruby-on-rails-server-options)
* [https://draveness.me/ruby-webserver](https://draveness.me/ruby-webserver)
