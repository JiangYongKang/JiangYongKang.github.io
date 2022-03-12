---
title: 'Rails 中 ActiveSupport::Concern 的使用'
date: 2017-10-25 18:22:30
tags: Rails
categories: Ruby
---

先观察一下下面两个类
```ruby
# app/models/accout.rb
class Account < ActiveRecord::Base
  def message(content)
    "Message: #{content}"
  end
end

# app/models/user.rb
class User < ActiveRecord::Base
  def message(content)
    "Message: #{content}"
  end
end
```

<!-- more -->

两个类存在同样的方法 `message`，这种情况下我们可以使用，`ActiveSupport::Concern` 来将这一段代码提出来，简化我们的类。
```ruby
# app/modles/concerns/common.rb
module Common extend ActiveSupport::Concern # 这里的继承一定不要忘记了
  # 可以将重复的代码都提取到这里
  def message(content)
    "Message: #{content}"
  end
end
```

重构原本的两个类，将 `Common` 包含进去。
```ruby
# app/models/accout.rb
class Account < ActiveRecord::Base
  include Common
end

# app/models/user.rb
class User < ActiveRecord::Base
  include Common
end
```

测试一下代码，是正常运行的。把相同逻辑的代码抽取到 `ActiveSupport::Concern` 中，可以很大程度的减少 `ActiveRecord Model` 的复杂度。也符合代码的高内聚，低耦合。
```ruby
$ rails c
Running via Spring preloader in process 11485
Loading development environment (Rails 4.2.0)
irb(main):001:0> Account.new.message "hello world!"
=> "hello world!"
irb(main):001:0> User.new.message "hello world!"
=> "hello world!"
```
