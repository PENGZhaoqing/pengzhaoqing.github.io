---
date: 2015-11-13
title: Ruby－mixin机制和Java接口
categories: 学习笔记
tags: [ruby]
layout: single-blog
---

在ruby中，支持把模块mixin（混入）进类中，于是在这个类中我们能使用模块中定义的方法和变量。

mixin的存在是为了解决ruby的类无法多继承的缺陷，类似于Java中的接口（interface），但它比interface更要灵活：<text style="color:cornflowerblue">我们知道在Java中，接口被子类继承后要覆写接口中定义的所有方法，这些方法都是public而且abstract或者static的</text>。

但是mixin一个模块就要简单多了，例如: <text style="color:red">如果要继承Enumerbale模块，只需要覆写each方法，然后在类开头include Enumerable即可</text>。

## 如何使用Enumerable模块

举个例子，我们实现一个FibSequence类，这个类在实例化的时候能够产生特定的斐波那契数列，然后能调用reject、collect、map等方法对数列进行处理。

``` ruby

class FibSequence
  include Enumerable
  
  def initialize(arg)
     fibonacci(arg)
  end
  
   def getter
     @array
   end
 
  # overwirte each function to mixin Enumerable moudle
  def each
    @array.each do |value|
      yield value
    end
  end
  
  def func(n)
    if(n==1||n==2)
      return 1
    end 
    return func(n-1)+func(n-2)
  end
    
  def fibonacci(count)
     @array=[]
     i=1
     while i<=count do
       @array<<func(i)
       i=i+1
     end
  end
end
```

结果:

```ruby
f=FibSequence.new(6)
f.each { |s| puts "#{s}" }
# => [1, 1, 2, 3, 5, 8]
f.reject { |s| s.odd? }
# => [2, 8]
f.reject(&:odd?)
# => [2, 8]
f.map{|x|2*x}
# => [2, 2, 4, 6, 10, 16]
```

只要定义了each，其他的collect，map等方法都能顺利在类中被调用，所以其他的collect、map应该都是基于each方法实现的，最终调用的都是each方法。

