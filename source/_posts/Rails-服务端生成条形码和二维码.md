---
title: Rails 服务端生成条形码和二维码
date: 2019-07-30 22:51:03
tags: Rails
categories: Ruby
---

通常情况下，图片的生成应当在客户端去实现。但是当我们必须在服务端去生成图片时 [Barby Gem](https://github.com/toretore/barby/) 是一个非常不错的选择。它是一个用来生成各种标准的条形码，以及二维码的库。`Barby` 的代码结构可以大致分为 **生成器** 和 **输出**。输出器的功能非常全面，可以输出 Base64、PNG、PDF 等等，甚至我们可以基于 `Barby` 添加自己的输出器。

<!-- more -->

第一步将 [rqrcode Gem](https://github.com/whomwah/rqrcode/) 的依赖添加到 Gemfile 中
```Ruby
gem 'barby', '~> 0.6.8'
gem 'rqrcode', '~> 0.10.1' # 如果不需要生成二维码，可以不添加
```

第二步，在 `config/application.rb` 中加载相关的依赖库
```ruby
require 'barby/barcode/qr_code'
require 'barby/barcode/code_39'
require 'barby/outputter/png_outputter'
```

第三部，创建 `app/controllers/concerns/barby_generator.rb` 编写用于生成条形码和二维码图片的通用方法
```ruby
module BarbyGenerator
  extend ActiveSupport::Concern

  def make_qr_code(content)
    data = Barby::QrCode.new(content)
    base64_output = Base64.encode64 data.to_png({ margin: 2, xdim: 26, ydim: 26 })
    "data:image/png;base64,#{base64_output}".gsub(/\n/, '')
  end

  def make_barcode(content)
    data = Barby::Code39.new(content)
    base64_output = Base64.encode64 data.to_png({ margin: 2, xdim: 3, ydim: 3 })
    "data:image/png;base64,#{base64_output}".gsub(/\n/, '')
  end
end
```

最后就是控制图片尺寸大小的问题了，`Barby` 可以自定义生成图片的一些属性，具体的用法 [官方 WiKi](https://github.com/toretore/barby/wiki/Outputters#definitions-and-common-options) 中有详细的描述。可以按照自身的需求去设置。
