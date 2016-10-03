---
date: 2015-11-10 
title: Ruby－元编程和自定义访问器attr_accessor
categories: Study_notes
tags: [ruby]
---


这些天在自学《Engineering Software as a Service: An Agile Approach Using Cloud Computing》时候，遇到一个课后习题（project 3.6）:

**Define a method attr_accessor_with_history that provides the same functionality as attr_accessor but also tracks every value the attribute has ever had:**]

大概意思是自己定义一个访问器，不仅具有att_accessor的功能而且还能纪录输入的历史数据。琢磨了一天，终于有了点眉目，拿出来分享下。



# 1.在哪定义attr_accessor？

``` ruby
class Class
  def attr_accessor_custom(attr_name)
    attr_reader attr_name
    attr_writer attr_name
  end
end

class Example
  attr_accessor_custom :foo
end

a = Example.new;
a.foo = 2; 
puts a.foo # => 2
```

首先要知道在哪定义能attr_accessor。

在Ruby中，一切都是对象，包括类，上述的Example虽然是被声明为一个类，但是它实际上是一个Class类的对象，所以我们在Class类里定义 attr_accessor_custom这个方法后，由于Example是Class的一个实例对象，因此在Example里能直接调用这个方法。

然后是attr_reader和attr_writer，我们都知道attr_accessor定义后，既可以get变量又可以set变量，而attr_reader就相当于其中的getter，attr_writer就相当于setter。

下一步我们要拆开这个两个封装好的方法。

# 2.class_eval的引入

```ruby
class Class
  def attr_accessor_custom(attr_name)
    #getter
    self.class_eval(
    
    "def #{attr_name};
      @#{attr_name};
    end"
    
    ) 
    #setter
    self.class_eval(
    
    "def #{attr_name}=(value);
      @#{attr_name}=value;
    end"
    
    ) 
  end
end

class Example
  attr_accessor_custom :foo
end

a = Example.new;
a.foo = 2; 
puts a.foo # => 2
```

我们用class_eval替换了前面的attr_writer和attr_reader。class_eval能动态执行字符串保存的Ruby代码，当获取到字符串后会让解释器执行这段代码，而不是当成字符串处理。从上述代码中可以看出，正常定义def的函数作为字符串被传递给class_eval，因为整个定义在字符串中，所以＃｛attr_name｝就可以代表传入的变量attr_name，也就是这个方法的名字；当传入的变量不一样时，所以生成的方法名字也不一样，体现了动态的特点。

- 除了class_eval，还有instance_eval。这两个函数实际上是对实例对象或者类添加新的实例方法或者类方法，在这里的self.class_eval的self指的是Class这个类，class_eval为这个类添加了getter和setter两个类方法，因此他们能被所有Class类的实例调用。有兴趣深入了解的请戳这里http://web.stanford.edu/~ouster/cgi-bin/cs142-winter15/classEval.php

wow，可能你觉得引号不那么好看，但是我们可以用％Q代替, 放在class_eval的后面：

```ruby
class Class
  def attr_accessor_custom(attr_name)
    self.class_eval %Q(
    
    def #{attr_name};
      @#{attr_name};
    end
    
    ) 
    self.class_eval %Q(
    
    def #{attr_name}=(value);
      @#{attr_name}=value;
    end
    
    ) 
  end
end

class Example
  attr_accessor_custom :foo
end

a = Example.new;
a.foo = 2; 
puts a.foo # => 2

```

# 3.自定义attr_accessor_with_history

好了，前面铺垫了那么多，我们该分析一下正题了。题目要求能纪录输入的数据，其实很简单，多申明一个变量@#{attr_name}_history用来在每次set的时候来纪录输入的数据，然后增加一个类似get_history方法来返回所有纪录的数据：

```ruby
class Class
  def attr_accessor_with_history(attr_name)
    #getter
    attr_reader attr_name
    #history_getter
    attr_reader "#{attr_name}_history"
    #setter
    self.class_eval %Q{
      def #{attr_name}=(new_value)
        @#{attr_name} =new_value
        @#{attr_name}_history ||= [nil] 
        @#{attr_name}_history << new_value
      end
    }
  end
end

class Example
  attr_accessor_with_history :foo
end

a = Example.new; a.foo = 2; a.foo = "test"; 
puts a.foo_history # => [nil, 2, "test"]

```

我们只重新编辑了setter方法，增加了一个@#{attr_name}_history数组变量在每次set的时候纪录输入的数据，new_value被不断加入到这个数组，然后用已有的att_reader自动申明了两个getter，一个getter返回现有的new_value，另外一个history_getter返回@#{attr_name}_history数组中的数据。到此为止，题目就算做完了。

- `a||=1`表示当变量a存在但没有初值时等于 1

如果感觉上述代码跳跃性太大，下面的可能会更容易明白：

```ruby
class Class
  def attr_accessor_with_history(attr_name)
    #getter
    attr_reader attr_name
    
    #history_getter
    class_eval %Q{
      def #{attr_name}_history
        @#{attr_name}_history 
      end
	}
	
	#setter
	class_eval %Q{
      def #{attr_name}=(new_value)
        @#{attr_name} =new_value
        @#{attr_name}_history ||= [nil] 
        @#{attr_name}_history << new_value
      end
    }
  end
end

class Example
  attr_accessor_with_history :foo
end

a = Example.new; a.foo = 2; a.foo = "test"; 
a.foo_history # => [nil, 2, "test"]
```

# 4. 进阶

当然还可以不借助class_eval，用define_methods动态创建方法：

```ruby
class Class
  def attr_accessor_with_history(attr_name)
    attr_name = attr_name.to_s
    attr_reader attr_name

    define_method "#{attr_name}_history" do
      instance_variable_get("@#{attr_name}_history") 
    end

    define_method "#{attr_name}=" do |new_value|
      v = instance_variable_get("@#{attr_name}_history")
      v ||= [nil]
      v << new_value

      instance_variable_set("@#{attr_name}_history", v)
      instance_variable_set("@#{attr_name}", new_value)
    end
  end
end

class Example
  attr_accessor_with_history :foo
end

a = Example.new; a.foo = 2; a.foo = "test"; 
puts a.foo_history # => [nil, 2, "test"]

```

有兴趣可以戳[这里](http://apidock.com/ruby/Module/define_method)

# 5. 总结

其实我也是小白一个，由于刚接触ruby，有些说法不是很地道，有些地方还是自己猜的，并没有严格的去看源代码，哈哈哈，可能会有错误，请各路大神谅解 : )

下面给出参考过的reference：

<a href="http://stackoverflow.com/questions/11604149/initializing-attr-accessor-like-method-with-history">http://stackoverflow.com/questions/11604149/initializing-attr-accessor-like-method-with-history</a>

<a href="http://maxivak.com/ruby-metaprogramming-and-own-attr_accessor/#codesyntax_4">http://maxivak.com/ruby-metaprogramming-and-own-attr_accessor/#codesyntax_4</a>


  


