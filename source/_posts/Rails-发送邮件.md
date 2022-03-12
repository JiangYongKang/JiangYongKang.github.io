---
title: Rails 发送邮件
date: 2019-07-17 22:23:04
tags: Rails
categories: Ruby
---

首先需要在 `config/application.rb` 中配置发送邮件的相关配置

<!-- more -->

```Ruby
config.action_mailer.perform_caching       = false
config.action_mailer.perform_deliveries    = true
config.action_mailer.raise_delivery_errors = true
config.action_mailer.delivery_method       = :smtp
config.action_mailer.smtp_settings         = {
    address:              'smtp.exmail.qq.com',
    port:                 465,
    ssl:                  true,
    user_name:            '<username>',
    password:             '<password>',
    authentication:       'plain',
    enable_starttls_auto: true,
}
```

使用脚手架工具生成邮件视图模板，脚手架工具会为我们自动创建发送邮件的方法、视图模板、测试以及预览组件
```shell
$ rails g mailer welcome
Running via Spring preloader in process 59807
      create  app/mailers/welcome_mailer.rb
      invoke  erb
      create    app/views/welcome_mailer
      invoke  test_unit
      create    test/mailers/welcome_mailer_test.rb
      create    test/mailers/previews/welcome_mailer_preview.rb
```

打开 `app/mailers/application_mailer.rb` 文件，替换 `default from` 后面的参数为发件人邮箱，如：
```ruby
class ApplicationMailer < ActionMailer::Base
  default from: 'xxxxxxxx@qq.com'
  layout 'mailer'
end
```

打开 `app/mailers/welcome_mailer.rb` 文件，编辑收件人和邮件主题，如：
```ruby
class WelcomeMailer < ApplicationMailer
  def welcome_mail(member)
    @member = member
    mail to: @member.email, subject: 'Welcome to My Awesome Site'
  end
end
```

在 `app/views/welcome_mailer` 文件夹中创建 `welcome_mailer.html.erb` 文件，使用 `html` 来编写邮件的视图。如下：
```erb
<h1>Welcome to example.com, <%= @member.username %></h1>
<h3>You have successfully signed up to example.com</h3>
<h3>Thanks for joining and have a great day!</h3>
```

然后，你就可以测试发送邮件了。
```ruby
WelcomeMailer.welcome_mail(@member).deliver_later # 延迟发送邮件
WelcomeMailer.welcome_mail(@member).deliver_now   # 立即发送邮件
```

`Rails` 可以支持预览邮件内容。打开 `test/mailers/previews/welcome_mailer_preview.rb` 文件。添加与发送邮件同名方法，如：
```ruby
class WelcomeMailerPreview < ActionMailer::Preview
  def welcome_mail
    WelcomeMailer.welcome_mail(Member.first)
  end
end
```
然后启动你的 `Rails` 应用，访问 [http://localhost:3000/rails/mailers/welcome_mailer/welcome_mail](http://localhost:3000/rails/mailers/welcome_mailer/welcome_mail) 进行预览。

关于发送邮件更多的资料可以在 [https://ruby-china.github.io/rails-guides/action_mailer_basics.html](https://ruby-china.github.io/rails-guides/action_mailer_basics.html) 中查看。
