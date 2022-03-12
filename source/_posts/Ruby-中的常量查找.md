---
title: Ruby 中的常量查找
date: 2019-11-13 16:11:53
tags: Ruby
categories: Ruby
---

在 Ruby 中，当你需要访问一个常量的时候，很简单直接使用这个常量的名字就行。但是你知道 Ruby 是如何查找一个常量的吗？简单的讲 Ruby 会按照 `Module.nesting` => `Module.nesting.first.ancestors` => `Object.ancestors` 的顺序去查找常量。

<!-- more -->

### Module.nesting
`Module.nesting` 这个方法会返回在调用点的 `Module` 列表，例如：
```Ruby
module A
  module B
    module C
      puts "Module.nesting => #{Module.nesting}" # Module.nesting => [A::B::C, A::B, A]
    end
  end
end
```
从输出的结果来看，你应该能很直观的了解到什么是 `Module.nesting`，如果你使用的是简写的 `Module` 定义形式，`Module.nesting` 会跳过命名空间。例如：
```Ruby
module A
  module B
  end
end
module A::C
  puts "Module.nesting => #{Module.nesting}" # Module.nesting => [A::C]
  puts B # uninitialized constant error，跳过的命名空间的常量不可用。这是因为外部命名空间未添加到 Module.nesting
end
```

### Module.ancestors
`Module.ancestors` 会返回 Module 中 `prepended/included` 的列表。同时也包含了调用者本身。如果是通过 `Class.ancestors` 调用，则返回类的祖先列表。
```Ruby
class A
  module B
  end
end
class C < A
  # 在子类中，可以访问父类中定义的 module。但是反过来在父类中是无法访问子类的 module
  puts "B = #{B}, B.ancestors = #{B.ancestors}" # B = A::B, B.ancestors = [A::B]
end
```
可以发现，在 `Module.nesting` 无法查找到常量时，会通过 `ancestors` 方法去查找常量，这也是我们比较常见的一种查找方式

### Object.ancestors
当 `Module.nesting == []` 的时候，说明你正处于一个顶级对象中，也就是说你正处于打开的 Object 中。根据文章开始列出的三个查找顺序，第三个 `Object.ancestors` 满足了搜寻条件，所以 `Ruby` 会去 `Object.ancestors` 中寻找。 关于 `Object` 这里有个重要的假设，如果你当前打开的是一个 Module，那么 Ruby 会自动将 `Object.ancestors` 加入到你搜寻的常量列表中。例如：
```Ruby
module A
end
module B
  puts "#{A == Object::A}" # true
end
```
直接定义的 module 直接被附加到 Object 中，根据输出可以看出来，`module A` 是在顶级对象中的。所以这里也可以直接用 `Object::A` 访问。根据这个理论，我们可以得到一个结论，**任何顶级对象下的常量始终都是可以访问的**。

### Class.class_eval
一般情况下，`Class.class_eval` 以及它的代码块并不会对常量查找产生什么影响。`Ruby` 还是会从代码块定义的地方开始去搜寻。例如：
```Ruby
class A
  module B
  end
end
class C
  module B
  end
  # 如果传入一个快块进入 class_eval 或 module_eval 或 instance_eval 或 define_method，这不会改变常量查找的位置
  A.class_eval do
    puts "#{B == C::B}" # true
  end
end
```

### 单例类
单例类的 `ancestors` 并不包含类本身，一个类的单例类的祖先不包括类本身，它们从Class类开始。如：
```Ruby
class A
  module B
  end
end
class << A
  puts "ancestors = #{ancestors}" # ancestors = [#<Class:A>, #<Class:Object>, #<Class:BasicObject>, Class, Module, Object, Kernel, BasicObject]
end
```
```Ruby
class A
  module B
  end
end
# 在一个类的单例类中，你不能访问类本身定义的常量
class << A
  B # uninitialized constant #<Class:A>::B (NameError)
end
```

### 参考资料
原文：[https://cirw.in/blog/constant-lookup](https://cirw.in/blog/constant-lookup)
翻译：[https://ruby-china.org/topics/26890](https://ruby-china.org/topics/26890)
