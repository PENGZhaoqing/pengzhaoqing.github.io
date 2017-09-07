---
date: 2017-09-02
title: Heroku部署Rails应用流程
categories: 学习笔记
tags: [ruby]
---



1.创建Heroku账号以及Heroku app

2.将Heroku app与自己Github下的的项目进行连接

4.下载配置[Heroku CLI](https://devcenter.heroku.com/articles/heroku-command-line)命令行工具

5.在本机终端中使用`heroku login`命令行登陆，会要求heroku的账号密码

6.登陆成功后，可以用`heroku create`在当前目录下创建新的heroku app, 若已经有了heroku app ，请使用`heroku git:remote -a app_name`切换到现在即将要部署那个heroku app里

7.使用git remote -v查看与heroku的远程仓库连接，输出如下：

```
heroku  https://git.heroku.com/house-pricing.git (fetch)
heroku  https://git.heroku.com/house-pricing.git (push)
origin  https://github.com/PENGZhaoqing/HousePricing.git (fetch)
origin  https://github.com/PENGZhaoqing/HousePricing.git (push)
```

8.命令`git push heroku master`将当前目录下的项目push到heroku服务器中，提交前请将所有的修改都commit，输出如下：

```
PENG-MacBook-Pro:HousePricing PENG-mac$ git push heroku master
Counting objects: 3, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 2.32 MiB | 160.00 KiB/s, done.
Total 3 (delta 1), reused 0 (delta 0)
remote: Compressing source files... done.
remote: Building source:
remote: 
remote: -----> Ruby app detected
remote: -----> Compiling Ruby/Rails
remote: -----> Using Ruby version: ruby-2.2.4
remote: -----> Installing dependencies using bundler 1.15.2
remote:        Running: bundle install --without development:test --path vendor/bundle --binstubs vendor/bundle/bin -j4 --deployment
remote:        Fetching gem metadata from https://rubygems.org/..........
remote:        Fetching version metadata from https://rubygems.org/..
remote:        Fetching dependency metadata from https://rubygems.org/.
remote:        Using rake 11.2.2
remote:        Using i18n 0.7.0
remote:        Using json 1.8.3
remote:        Using minitest 5.9.0
remote:        Using thread_safe 0.3.5
remote:        Using builder 3.2.2
remote:        Using erubis 2.7.0
remote:        Using mini_portile2 2.0.0
remote:        Using rack 1.6.4
remote:        Using mime-types-data 3.2016.0521
remote:        Using arel 6.0.3
remote:        Using execjs 2.7.0
remote:        Using htmlentities 4.3.4
remote:        Using rubyzip 1.1.7
remote:        Using bcrypt 3.1.11
remote:        Using sass 3.4.22
remote:        Using will_paginate 3.1.0
remote:        Using bundler 1.15.2
remote:        Using coffee-script-source 1.10.0
remote:        Using thor 0.19.1
remote:        Using concurrent-ruby 1.0.2
remote:        Using multi_json 1.12.1
remote:        Using mimemagic 0.3.2
remote:        Using pg 0.18.4
remote:        Using rails_serve_static_assets 0.0.5
remote:        Using rails_stdout_logging 0.0.5
remote:        Using ruby-ole 1.2.12
remote:        Using tilt 2.0.5
remote:        Using tzinfo 1.2.2
remote:        Using rdoc 4.2.2
remote:        Using nokogiri 1.6.7.2
remote:        Using rack-test 0.6.3
remote:        Using mime-types 3.0
remote:        Using autoprefixer-rails 6.4.0.2
remote:        Using uglifier 3.0.0
remote:        Using bootstrap-will_paginate 0.0.10
remote:        Using coffee-script 2.4.1
remote:        Using sprockets 3.6.0
remote:        Using rails_12factor 0.0.3
remote:        Using spreadsheet 1.1.3
remote:        Using activesupport 4.2.5.2
remote:        Using loofah 2.0.3
remote:        Using axlsx 2.1.0.pre
remote:        Using roo 2.4.0
remote:        Using bootstrap-sass 3.3.7
remote:        Using mail 2.6.4
remote:        Using sdoc 0.4.1
remote:        Using rails-deprecated_sanitizer 1.0.3
remote:        Using globalid 0.3.6
remote:        Using activemodel 4.2.5.2
remote:        Using climate_control 0.0.3
remote:        Using jbuilder 2.4.1
remote:        Using rails-html-sanitizer 1.0.3
remote:        Using roo-xls 1.0.0
remote:        Using activejob 4.2.5.2
remote:        Using activerecord 4.2.5.2
remote:        Using cocaine 0.5.8
remote:        Using rails-dom-testing 1.0.7
remote:        Using paperclip 5.1.0
remote:        Using actionview 4.2.5.2
remote:        Using actionpack 4.2.5.2
remote:        Using actionmailer 4.2.5.2
remote:        Using axlsx_rails 0.5.0
remote:        Using railties 4.2.5.2
remote:        Using sprockets-rails 3.0.4
remote:        Using coffee-rails 4.1.1
remote:        Using jquery-rails 4.1.1
remote:        Using rails 4.2.5.2
remote:        Using sass-rails 5.0.4
remote:        Using turbolinks 2.5.3
remote:        Bundle complete! 23 Gemfile dependencies, 70 gems now installed.
remote:        Gems in the groups development and test were not installed.
remote:        Bundled gems are installed into ./vendor/bundle.
remote:        The latest bundler is 1.15.4, but you are currently running 1.15.2.
remote:        To update, run `gem install bundler`
remote:        Bundle completed (3.21s)
remote:        Cleaning up the bundler cache.
remote:        The latest bundler is 1.15.4, but you are currently running 1.15.2.
remote:        To update, run `gem install bundler`
remote: -----> Installing node-v6.11.1-linux-x64
remote: -----> Detecting rake tasks
remote: -----> Preparing app for Rails asset pipeline
remote:        Running: rake assets:precompile
remote:        Asset precompilation completed (2.14s)
remote:        Cleaning assets
remote:        Running: rake assets:clean
remote: 
remote: ###### WARNING:
remote:        You have not declared a Ruby version in your Gemfile.
remote:        To set your Ruby version add this line to your Gemfile:
remote:        ruby '2.2.4'
remote:        # See https://devcenter.heroku.com/articles/ruby-versions for more information.
remote: 
remote: ###### WARNING:
remote:        No Procfile detected, using the default web server.
remote:        We recommend explicitly declaring how to boot your server process via a Procfile.
remote:        https://devcenter.heroku.com/articles/ruby-default-web-server
remote: 
remote: -----> Discovering process types
remote:        Procfile declares types     -> (none)
remote:        Default types for buildpack -> console, rake, web, worker
remote: 
remote: -----> Compressing...
remote:        Done: 53.3M
remote: -----> Launching...
remote:        Released v15
remote:        https://house-pricing.herokuapp.com/ deployed to Heroku
remote: 
remote: Verifying deploy... done.
To https://git.heroku.com/house-pricing.git
   421c4c2..413eb1f  master -> master
```
9.在服务器中建表，运行`heroku run rake db:migrate`，然后也可以写入种子数据`heroku run rake db:seed`， 但是若要重构整个数据库，`heroku run rake db:migrate:reset`会失败，因为没有权限，解决方法是使用`heroku pg:reset` 来重构


10.数据库建好之后，就可以进行正常访问了，使用`heroku ps`查看运行情况，使用`heroku open`访问网页

### 迁移本地数据库postgres至heroku服务器

1.使用`pg_dump your_databse_name > mydb.dump`，将数据库导出到当前目录的 mydb.dump文件中，若pg_client与pg_server版本不符合，会报以下错误：

```
pg_dump: server version: 9.5.4; pg_dump version: 9.4.5
pg_dump: aborting because of server version mismatch
```
**解决方案**：使用`find / -name pg_dump -type f 2>/dev/null`查找所有的pg_dump安装版本，我的输出如下（Mac OS）：

```
/Applications/Postgres.app/Contents/Versions/9.5/bin/pg_dump
/usr/local/Cellar/postgresql/9.4.5_2/bin/pg_dump
```
显然这里pg_dump环境变量目录是在9.4.5_2下，所以才会提示错误，我们只需要使用上面那个路径下的pg_dump就可以，因此用以下命令解决，：

```
/Applications/Postgres.app/Contents/Versions/9.5/bin/pg_dump -Fc --no-acl --no-owner -h localhost -U postgres housepricing_development > mydb2.dump

```
 也可以创建新的symlink，这样以后每次调用就不用加上目录了：

```
sudo ln -s /Applications/Postgres.app/Contents/Versions/9.5/bin/pg_dump /usr/local/Cellar/postgresql/9.4.5_2/bin/pg_dump --force
```

2.将mydb.dump上传至一个能用HTTP访问位置，heroku官方推荐使用Amazon的S3，登陆Amazon S3控制台，新建bucket，在bucket中上传这个文件，上传后点击Make Public让这个资源能被公共访问，你也可以使用dropbox等分享这个文件。然后使用 `heroku pg:backups:restore dump_url DATABASE_URL`在huroku服务器中恢复本地的数据库，命令如下：

```
heroku pg:backups:restore "https://s3-ap-southeast-1.amazonaws.com/campus-portal/mydb1.dump" DATABASE_URL
```

成功后的输出：

```
 ▸    WARNING: Destructive Action
 ▸    This command will affect the app house-pricing
 ▸    To proceed, type house-pricing or re-run this command with --confirm house-pricing

> house-pricing
Starting restore of https://s3-ap-southeast-1.amazonaws.com/campus-portal/mydb1.dump to postgresql-concave-31360... done

Use Ctrl-C at any time to stop monitoring progress; the backup will continue restoring.
Use heroku pg:backups to check progress.
Stop a running restore with heroku pg:backups:cancel.

Restoring... done
```

#### Reference:
1. https://stackoverflow.com/questions/12836312/postgresql-9-2-pg-dump-version-mismatch

2. https://devcenter.heroku.com/articles/getting-started-with-rails4

3. https://stackoverflow.com/questions/31057998/migrating-database-from-local-development-to-heroku-django-1-8

4. https://devcenter.heroku.com/articles/heroku-postgres-import-export




