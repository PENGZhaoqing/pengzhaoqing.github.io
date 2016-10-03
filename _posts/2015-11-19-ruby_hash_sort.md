---
date: 2015-11-19
title: Ruby－Hash表的排序高级用法
categories: Study_notes
tags: [ruby] 
---

**经常会遇到hash表中元素比较的问题，mark一下**

在ruby中,如果我要比较这个Hash表中年龄的大小：

```ruby
people = {
  :me => 23,
  :brother => 18,
  :sister => 25
}
```

直接用sort方法：

```ruby
irb(main):025:0> people.sort
=> [[:brother, 18], [:me, 23], [:sister, 25]]
```


ruby比较智能，可能会知道我们要比较的Fixnum对象的大小。

而且sort方法对字符串类型的也能处理：

```ruby
irb(main):037:0> ["3","1","2"].sort
=> ["1", "2", "3"]
```

但是，如果我们想指定Hash表中比较的元素，就要用到sort_by方法，比如要处理下面的复杂点的内嵌的hash表：

```ruby
people = {
  :me => { :name => "hen", :age => 23 },
  :brother => { :name => "abb", :age => 18 },
  :sister => { :name => "ndd", :age => 25 }
}
```

我们可以这样写：

```ruby
people.sort_by do |k,v|
  v[:age]
end
#=> [[:brother, {:name=>"abb", :age=>18}], [:me, {:name=>"hen", :age=>23}], [:sister, {:name=>"ndd", :age=>25}]]
```

解释: sort_by提取出了两个元素k和v，k是第一层hash表的键（symbol对象），而v是第一层hash表的值（内嵌hash对象），这样v[:age]就代表着年龄值（Fixnum），块内需指明了要比较的对象，在这里是指的是Fixnum对象，还有其他String也可以，所以sort_by能正常工作。

简单验证下上面的说法：

```ruby
people.sort_by do |k,v|
  print "#{k.class} #{v.class}\n"
end
# => Symbol Hash
# => Symbol Hash
# => Symbol Hash
```

回到之前的people对象，我们可以用sort_by这样写：

```ruby
people = {
  :me => 23,
  :brother => 18,
  :sister => 25

people.sort_by do |k|
  k[1]
end
# => [[:brother, "18"], [:me, "23"], [:sister, "25"]]
```

在这里，sort_by提取出来的k是数组，所以`k[1]`就代表着年龄值。

而且,可以看出来，`:me => 23` 在hash内部是一个数组，数组第一个值是键，第二个是值。


