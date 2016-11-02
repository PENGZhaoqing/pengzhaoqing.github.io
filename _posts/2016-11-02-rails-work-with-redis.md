---
date: 2016-11-02 
title: How Rails work with redis
categories: 学习笔记
tags: [rails redis]
---

Redis是一个非常快速的，原子性的key-value存储。它允许字符串，集合，排序集合，列表和散列存储。 Redis把数据持久化在RAM中，很像Memcached但又不全是Memcached的，因为Redis的周期性写入磁盘来维持它的持久性。

## 1. 数据类型

Redis支持五种数据类型：string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set：有序集合)，详情请看[这里](http://www.runoob.com/redis/redis-data-types.html)

## 2. 启动服务

运行redis-server，启动redis服务器，得到以下

```
PENG-MacBook-Pro:~ PENG-mac$ redis-server
31998:C 02 Nov 13:58:45.177 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
31998:M 02 Nov 13:58:45.178 * Increased maximum number of open files to 10032 (it was originally set to 256).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 3.0.6 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 31998
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

31998:M 02 Nov 13:58:45.179 # Server started, Redis version 3.0.6
31998:M 02 Nov 13:58:45.184 * DB loaded from disk: 0.004 seconds
31998:M 02 Nov 13:58:45.184 * The server is now ready to accept connections on port 6379
```

 以上未指定配置文件，使用默认的配置文件，若要自定义请在后面指定参数`redis-server /path/to/redis.conf`

然后用`redis-cli`本地连接到redis服务器

## Rails and Redis

在Rails中为了避免Sql的反复查询，可以考虑把一部分数据用Redis储存（缓存），例如Web服务器中的Session储存就可以利用redis来缓存，这里用个简单的例子来说明Redis的一些基本方法和如何连接rails和redis

1.创建新的rails应用`rails new redis-test`

2.在Gemfile中加入ruby的redis库
```
gem 'redis'
```

3.运行`bundle install`安装依赖库

4.在Rails中配置与redis的连接：在初始化文件中创建`config/initializers/redis.rb`，并填入一下：
```
$redis = Redis.new(:host => 'localhost', :port => 6379)
```

以上会创建一个新的redis客户端实例，与`localhost:6379`连接（默认），这个实例会被储存在全局变量`$redis`中，能在其他任何地方直接调用

5.运行`rails console`进入Rails的控制台，检查与redis的连接：

```
PENG-MacBook-Pro:redis-test PENG-mac$ rails console
Loading development environment (Rails 4.2.5.2)
2.2.4 :001 > $redis
 => #<Redis client v3.2.2 for redis://localhost:6379/0> 
2.2.4 :002 > $redis.set('chunky', 'bacon')
 => "OK" 
2.2.4 :003 > $redis.get('chunky')
 => "bacon" 
2.2.4 :004 > 
```

6.运行`rails generate model user name:string`建立一个用户模型，并指定字段name的类型为字符串

7.下面考虑用户之间的关注问题：比如，在微博中用户之间可以互相关注（follow），那么就有followers和followings，followings表示我关注的人，followers表示关注我的人。这种功能的实现可以通过传统的关系数据库实现：多对多的关联关系，通过建立额外的关联表，储存双方的id即可，请戳[这里](https://www.railstutorial.org/book/following_users)获得详细的描述

现在我们通过redis来实现这个，编辑`user.rb`:

``` ruby
class User < ActiveRecord::Base
  # follow a user
  def follow!(user)
    $redis.multi do
      $redis.sadd(self.redis_key(:following), user.id)
      $redis.sadd(user.redis_key(:followers), self.id)
    end
  end
  
  # unfollow a user
  def unfollow!(user)
    $redis.multi do
      $redis.srem(self.redis_key(:following), user.id)
      $redis.srem(user.redis_key(:followers), self.id)
    end
  end
  
  # users that self follows
  def followers
    user_ids = $redis.smembers(self.redis_key(:followers))
    User.where(:id => user_ids)
  end

  # users that follow self
  def following
    user_ids = $redis.smembers(self.redis_key(:following))
    User.where(:id => user_ids)
  end

  # users who follow and are being followed by self
  def friends
    user_ids = $redis.sinter(self.redis_key(:following), self.redis_key(:followers))
    User.where(:id => user_ids)
  end

  # does the user follow self
  def followed_by?(user)
    $redis.sismember(self.redis_key(:followers), user.id)
  end
  
  # does self follow user
  def following?(user)
    $redis.sismember(self.redis_key(:following), user.id)
  end

  # number of followers
  def followers_count
    $redis.scard(self.redis_key(:followers))
  end

  # number of users being followed
  def following_count
    $redis.scard(self.redis_key(:following))
  end
  
  # helper method to generate redis keys
  def redis_key(str)
    "user:#{self.id}:#{str}"
  end
end
```

如何使用?

``` ruby
> %w[Alfred Bob].each{|name| User.create(:name => name)}
=> ['Alfred', 'Bob']
> a, b = User.all
=> [#<User id: 1, name: "Alfred">, #<User id: 2, name: "Bob">] 
> a.follow!(b)
=> [1, 1] 
> a.following?(b)
=> true 
> b.followed_by?(a)
=> true 
> a.following
=> [#<User id: 2, name: "Bob">] 
> b.followers
=> [#<User id: 1, name: "Alfred">]
> a.friends
=> [] 
> b.follow!(a)
=> [1, 1] 
> a.friends
=> [#<User id: 2, name: "Bob">] 
> b.friends
=> [#<User id: 1, name: "Alfred">] 
```

我们使用了上述redis的集合（set）数据类型，使用了以下方法：

| 方法 | 说明 | 示例 |
| ------------- |:-------------:| -----:|
| sadd | 向集合中添加新的成员(不能重复) | `SADD myset "Hello"`|
| srem | 移除集合中一个或多个成员 | `SREM myset "Hello"`|
| smembers | 返回集合中的所有成员 | `SMEMBERS myset`|
| sinter | 返回给定所有集合的交集 | `SINTER set1 set2`|
| scard | 获取集合的成员数 | `SCARD myset`|
| multi | 标志着一个事务块的开始（多条语句执行） | |

## 结束语

上述通过一个实际例子，介绍了如何在Rails中使用redis储存，以及redis的集合的一些操作，可以看出来，用key-value的储存方式设计起来要比传统的多对多关系要简单的多。

当然，当Web服务器中的登录人数过多时，仅靠内存来储存session会话信息可能不太够用（<4k），这时就可以考虑用redis来缓存，因为redis可以周期性将数据部分储存在硬盘里。但是如果直接将sessions会话信息直接储存在数据库中，随着时间的增加，这些冗余数据会占据一部分空间，因此需要周期对数据库进行清理。所以相对来说，使用redis缓存会成为一个折中的方案。

reference：http://www.justinweiss.com/articles/how-rails-sessions-work/

http://jimneath.org/2011/03/24/using-redis-with-ruby-on-rails.html



