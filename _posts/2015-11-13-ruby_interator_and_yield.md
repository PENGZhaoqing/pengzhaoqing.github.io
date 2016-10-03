---
date: 2015-11-13
title: Ruby－自定义迭代器与yield方法
categories: Study_notes
tags: [ruby]
---

大家对于迭代器each_with_index应该不是很陌生：

```ruby
%w(dog cat parrot).each_with_index do |animal,index|
  puts ">> #{animal} is number #{index}"
end
#>> dog is number 0
#>> cat is number 1
#>> parrot is number 2
```

那么如果要自定义一个迭代器 each_with_custom_index, 要求对index进行管理，即自定义index的起点和步距（如下图），该怎么做呢？

```ruby
%w(dog cat parrot).each_with_custom_index(3,2) do |animal,index|
  puts ">> #{animal} is number #{index}"
end
#>> dog is number 3
#>> cat is number 5
#>> parrot is number 7
```

首先要知道%w这个返回的是一个数组：

```ruby
irb(main):001:0> %w(dog cat parrot).class
=> Array
```

然后我们找出Array的继承类和模块：

```ruby
irb(main):002:0> Array.ancestors
=> [Array, Enumerable, Object, Kernel, BasicObject]
irb(main):003:0> Array.included_modules
=> [Enumerable, Kernel]
```

对于each_with_index，有很大可能是在Enumerable模块中定义的，Enumerable被include在Array类里，所以Array的实例对象才能直接调用each_with_index，我们可以验证一下我们的猜想：

```ruby
irb(main):005:0> Enumerable.instance_methods.respond_to? :each_with_index
=> true
```

当然，完全没有必要去这样做，因为可以直接Google出each_with_index这个方法，找ruby对应的API，就可以知道在哪个模块实现的，但作为新手，这样只是帮助梳理一下思路 ：）

那么我们就在Enumerable中定义新方法each_with_custom_index：

```ruby
module Enumerable
  def each_with_custom_index(start,offset)
    self.each_with_index do |start,index|
      yield name, start+index*offset
    end
  end
end

%w(dog cat parrot).each_with_custom_index(3,2) do |animal,index|
  puts ">> #{animal} is number #{index}"
end
#>> dog is number 3
#>> cat is number 5
#>> parrot is number 7
```
最终还是利用了each_with_index方法，只不过把这种实现封装了Enumerable模块中，以后只要是mix-in了这个模块的实例对象都可以实现这个功能。

**yield方法**

yield这个函数能回调参数到与方法名相同的代码块中执行,也就是说，它能连接同名的块与方法，举个例子：

```ruby
def Haha
  a=1; b=2
  yield a,b
end

Haha do |a,b| 
  puts "a=#{a},b=#{b}"
end 
#＝>a=1,b=2

Haha {|a,b| puts "a=#{a},b=#{b}"}
#＝>a=1,b=2
```

上述代码块Haha用了两种写法，a和b都是在Haha方法内定义，然后我们看一下执行顺序：

```ruby
def Haha
  puts "方法内" 
  a=1; b=2
  yield a,b
  puts "方法内" 
end

Haha do |a,b|
  puts "代码块内" 
  puts "a=#{a},b=#{b}"
  puts "代码块内"
end 
#=> 方法内
#=> 代码块内
#=> a=1,b=2
#=> 代码块内
#=> 方法内
```

首先是在方法内执行，当执行到yield方法后转到同名的Haha代码块内，执行完代码块后继续执行方法内的代码。

关于块，更多戳这里：<a href="http://www.runoob.com/ruby/ruby-block.html"></a>


