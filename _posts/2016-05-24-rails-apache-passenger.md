---
date: 2016-05-24
title: Rails部署云环境配置（Passenger＋Apache）
categories: Web开发
tags: [rails]
---

**买了个阿里云的虚拟云主机ECS,在上面搭建Rails部署的生产环境(production)...**

## 1.准备 

首先你要配置好你的云主机, 包括ssh连接, git, 创建用户等

配置步骤详见 <a href="https://www.digitalocean.com/community/tutorials/how-to-deploy-a-rails-app-with-passenger-and-apache-on-ubuntu-14-04" style="color:red"> 英文博客 </a>

然后安装Passenger和Apache, rails, ruby, bundle, gem等

安装步骤详见:<a href="https://www.phusionpassenger.com/library/walkthroughs/deploy/ruby/" style="color:red"> Passenger官方指南 </a>

## 2.部署开始

1.将github上的代码克隆到服务器上

```ruby
sudo mkdir -p /var/www/portalgate
cd /var/www/portalgate
git clone git://github.com/username/myapp.git
```

2.安装第三方库

```ruby
bundle install --deployment --without development test
```

3.编译 Rails assets 

```ruby
bundle exec rake assets:precompile RAILS_ENV=production
```

4.运行数据库迁移

```ruby
bundle exec rake db:migrate RAILS_ENV=production
```

5.运行数据库seed

```ruby
bundle exec rake db:seed RAILS_ENV=production
```

6.配置生产模式的secrets.yml

```ruby
bundle exec rake secret
nano config/secrets.yml
```


7.配置apache和passenger

```ruby
sudo nano /etc/apache2/sites-enabled/portalgate.conf
```

并填入下面代码

```ruby
<VirtualHost *:80>
    ServerName yourserver.com

    # Tell Apache and Passenger where your app's 'public' directory is
    DocumentRoot /var/www/myapp/code/public

    PassengerRuby /path-to-ruby

    # Relax Apache security settings
    <Directory /var/www/myapp/code/public>
      Allow from all
      Options -MultiViews
      # Uncomment this if you're on Apache >= 2.4:
      #Require all granted
    </Directory>
</VirtualHost>
```

8.重启apache

```ruby
sudo apache2ctl restart
```

9.测试

```ruby
curl http://yourserver.com/
```


## 3.版本迭代(代码更新)


1.代码更新

```ruby
git pull
```

2.安装第三方库

```ruby
bundle install --deployment --without development test
```

3.编译 Rails assets 

```ruby
bundle exec rake assets:precompile RAILS_ENV=production
```

4.数据库迁移重置（若之前已seed过）
```
bundle exec rake db:migrate:reset RAILS_ENV=production
```

5.数据库seed

```ruby
bundle exec rake db:seed RAILS_ENV=production
```

6.重启应用
```ruby
passenger-config restart-app $(pwd)
```

reference：<a href="https://www.phusionpassenger.com/library/walkthroughs/deploy/ruby/ownserver/apache/oss/trusty/deploy_app.html" style="color:cornflowerblue">reference here</a>

### 更新数据库为postgresql

1.在终端中安装postgresql

```ruby
sudo apt-get update
sudo apt-get install postgresql postgresql-contrib
```

2.转换为postgresql用户，并创建用户

```ruby
sudo -i -u postgres
createuser --interactive
```

3.在上面创建的用户的工作区间下创建数据库

```ruby
createdb dabase_name
```

4.然后其他的跟sqlite3操作一样

reference：<a href="https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-14-04" style="color:cornflowerblue">refrence here</a>
