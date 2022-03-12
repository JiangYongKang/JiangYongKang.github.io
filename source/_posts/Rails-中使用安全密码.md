---
title: Rails 中使用安全密码
date: 2018-07-14 10:28:56
tags: Rails
categories: Ruby
---


但愿你没有在数据库里直接存储明文密码！为了防治密码被窃取，数据库中存储的始终应该是某种形式的哈希值，而非明文密码。但是，`Rails 3.1` 之后可以利用 `has_secure_password` 来实现。这个 `Rails` 内部的机制充分考虑了密码验证和加密。它需要 `model` 中有一个 `password_digest` 字段，用来存储加密后的密码。

<!-- more -->

```Ruby
$ rails g model user email:string password_digest:string
class User < ApplicationRecord
  has_secure_password
end
```

`has_secure_password` 这会为 `model` 加上 `password` 和 `password_confirmation` 两个属性。这两个属性是 `model` 的一部分，但是并不存在于数据库中，因为我们并不需要存储明文密码。

```Ruby
test 'fails because no password' do
  user = User.new(email: 'vincent@email.com')
  assert_not user.save
end

test 'fails because passwrod to short' do
  user = User.new(email: 'vincent@email.com', password: '123')
  assert_not user.save
end

test 'succeeds because password is long enough' do
  user = User.new(email: 'vincent@email.com', password: '123456')
  assert user.save
end
```

- 不提供密码，用户创建失败
- 密码过短，用户创建失败
- 密码合法，用户创建成功

用 `rails test` 跑这些用例时，第二个用例会失败。因为默认 `Rails` 不会加入密码长度的验证。现在我们在 `model` 中自己加上。

```Ruby
class User < ApplicationRecord
  has_secure_password
  validates :password, length: { minimum: 6 }
end
```
再跑一次测试，这次用例都通过了。检查一下数据库会发现 `users` 表的 `password_digest` 列存储着经过加密的数值，这正是我们想要的！现在来做验证的部分。假设一个新的注册用户现在想要登录。你可以这样进行验证：

```ruby
User.find_by(email: "EMAIL").authenticate("CLEAR_TEXT_PASSWORD")
```

如果从 `HTTP` 请求中传来的用户名和明文密码都正确，将从数据库中返回一个有效用户。否则返回 `nil` `has_secure_password` 只在对象创建时对密码进行验证。之后的操作便不再验证（例如 `update` ）。这个可以接受，因为这意味着你可以从数据库加载用户，在不知道密码的情况下修改和保存用户。`has_secure_password` 还为 `model` 提供了 `password_confirmation` 属性。只有当该属性为非 `nil` 值时才触发验证。如果该属性非 `nil`，则其值必须与 `password` 属性相等。让我们针对该属性再加两个测试用例。

```Ruby
test "fails because password confirmation doesnt match" do
  user = User.new(email: 'vincent@email.com', password: '123456', password_confirmation: '654321')
  assert_not user.save
end

test "succeeds because password & confirmation match" do
  user = User.new(email: 'vincent@email.com', password: '123456', password_confirmation: '123456')
  assert user.save
end
```

为了让测试通过，在 `model` 中再加入一行代码。

```Ruby
class User < ApplicationRecord
  has_secure_password
  validates :password, length: { minimum: 6 }
  validates_confirmation_of :password
end
```

`validates_confirmation_of :password` 将会检查密码的一致性。`Rails` 不强制使用 `password_confirmation`，但是你需要的话可以像这么来用。我确实很喜欢这个功能，因为它节省了很多代码和开发时间。而且对大部分应用来说够用了。`has_secure_password` 会自动加入以下验证规则：

- 在创建时密码必须存在
- 密码长度小于等于72字符
- 确认密码，使用 password_confirmation 属性

在实际项目中，通过第三方应用登录的用户是不需要密码的，这时可以通过传递 `validations: false` 参数移除验证规则。


```Ruby
has_secure_password validations: false
```

模块验证规则查看

```Ruby
User.validators
```

模块中某属性验证规则查看

```Ruby
User.validators_on(:password)
```
