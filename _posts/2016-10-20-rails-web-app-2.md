---
date: 2016-10-20
title: Rails Web App Learning in action (1)--the basic version of students selective courses
categories: Translation
tags: [rails]
---

#Rails Web App Learning in action (1)--the basic version of students selective courses

TThis tutorial is based on the postgraduate course, named software development methodology, from University of Chinese Academy of Sciences. This tutorial can help you generate a basic version of the final project. At first, you must set up Rails environment successfully and run the basic framework. Secondly, you can add more functions based on it. We will continue supply guidance for your following project. The code of the basic version of the Demo in this tutorial has been uploaded to Github already. This tutorial is helpful for new developers of Ruby on Rails to add the following new functions on the basic version:

 - Deal with course conflict and control the student number
 - Count credit points, degree course, etc.
 - Add the function of opening and closing the selective courses system.
 - Customize administrator backstage
 - Authorized login based on OAuth
 - Data input in Excel format
 - Bind user’s mailbox (to achieve the function of register activation,forgot password, etc.)
 - Search and retrieval within the site (such as searching courses
   according to the classification)

## Installation environment

Compared with other languages for Web development, Rails requires the developer to possess more relevant professional knowledge, for the reason that the construction of local development environment for Rails is relatively tough, especially under Windows system. For other system, such as Mac OS and Linux, the situation is better. Hence this tutorial is on basis of Linux system or Mac system.

###Local

1.Ubantu 14.04

Install via rbenv--[tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-ruby-on-rails-with-rbenv-on-ubuntu-14-04)
Install via RVM--[tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-ruby-on-rails-with-rvm-on-ubuntu-16-04)

2.Mac OS

Similar to Ubantu --[tutorial](https://gorails.com/setup/osx/10.11-el-capitan)

### Cloud IDE (integrated development environment)

In addition to the construction of local environment, you can also choose cloud IDE (integrated development environment) such as Cloud9 IDE. Using cloud IDE can be exempted from the trouble of setting up local environment. As long as you can surf the Internet, the computer can access cloud IDE through the webpage, and you can edit and run the code online. But this ease of use comes with a drawback: The IDE of cloud editor is not as strong as the download IDE of local. For example, RubyMine of Jetbeans company is the most intelligent IDE of Ruby so far, which can rival Eclipse and Netbeans of J ava. Besides, Jetbeans company also provide PyCharm for Python and PHPStorm for PHP language. They are all very good integrated development environment, which can add breakpoints to debug your applications, correct code automatically and layout automatically, etc. The good news is that students and teachers can use them for free for one year via applying by school email.

## Running instances

1.We take the development of Cloud9 as example. After you register and login, you will see the following interface:

![这里写图片描述](http://img.blog.csdn.net/20160923211215618)


2.Clicking “create a new workspace”, it will jump to the following page:

![这里写图片描述](http://img.blog.csdn.net/20160923211318667)

Note: Please write the branch URL behind fork in the website of Clone from Git. For instance, https://github.com/your account name of github /CourseSelect. Write project name and choose Ruby as the project language, then you can click to create new workspace.

3.Enter the page of editing IDE:

![这里写图片描述](http://img.blog.csdn.net/20160923211359949)

On the left is project directory and on the right is the window for editing files. The terminal at its lower right is for entering the command line.

4.Enter bundle install in terminal and install external libraries required by project, which is called as Gems by Rails. When you see following result, that means all libraries we need are installed completely:

```
pengzhaoqing:~/workspace (master) $ bundle install
Fetching gem metadata from https://gems.ruby-china.org/...........
Fetching version metadata from https://gems.ruby-china.org/..
Using rake 11.2.2
Using i18n 0.7.0
Using json 1.8.3
Using minitest 5.9.0
Using thread_safe 0.3.5
Using builder 3.2.2
Using erubis 2.7.0
Using mini_portile2 2.1.0
Using pkg-config 1.1.7
Using rack 1.6.4
Using mime-types-data 3.2016.0521
Using arel 6.0.3
Using execjs 2.7.0
Installing bcrypt 3.1.11 with native extensions
Using debug_inspector 0.0.2
Using sass 3.4.22
Using byebug 9.0.5
Using coffee-script-source 1.10.0
Using thor 0.19.1
Using concurrent-ruby 1.0.2
Using tilt 2.0.5
Using multi_json 1.12.1
Installing nested_form 0.3.2
Installing pg 0.18.4 with native extensions
Using bundler 1.12.5
Installing rails_serve_static_assets 0.0.5
Installing rails_stdout_logging 0.0.5
Installing remotipart 1.2.1
Installing safe_yaml 1.0.4
Using spring 1.7.2
Using turbolinks-source 5.0.0
Installing faker 1.6.3
Using rdoc 4.2.2
Using tzinfo 1.2.2
Using nokogiri 1.6.8
Using rack-test 0.6.3
Using mime-types 3.1
Installing autoprefixer-rails 6.4.0.2
Installing uglifier 3.0.1
Using binding_of_caller 0.7.2
Using coffee-script 2.4.1
Using sprockets 3.7.0
Installing haml 4.0.7
Installing rails_12factor 0.0.3
Using turbolinks 5.0.1
Using sdoc 0.4.1
Installing activesupport 4.2.5.2
Using loofah 2.0.3
Installing rack-pjax 0.8.0
Using mail 2.6.4
Installing bootstrap-sass 3.3.7
Using rails-deprecated_sanitizer 1.0.3
Using globalid 0.3.7
Installing activemodel 4.2.5.2
Using jbuilder 2.6.0
Using rails-html-sanitizer 1.0.3
Using rails-dom-testing 1.0.7
Installing activejob 4.2.5.2
Installing activerecord 4.2.5.2
Installing actionview 4.2.5.2
Installing actionpack 4.2.5.2
Installing actionmailer 4.2.5.2
Installing railties 4.2.5.2
Installing kaminari 0.16.3
Using sprockets-rails 3.1.1
Using coffee-rails 4.1.1
Installing font-awesome-rails 4.5.0.1
Installing jquery-rails 4.1.1
Installing jquery-ui-rails 5.0.5
Installing rails 4.2.5.2
Using sass-rails 5.0.6
Using web-console 2.3.0
Installing rails_admin 0.8.1
Bundle complete! 17 Gemfile dependencies, 73 gems now installed.
Use `bundle show [gemname]` to see where a bundled gem is installed.
Post-install message from haml:

HEADS UP! Haml 4.0 has many improvements, but also has changes that may break
your application:

* Support for Ruby 1.8.6 dropped
* Support for Rails 2 dropped
* Sass filter now always outputs <style> tags
* Data attributes are now hyphenated, not underscored
* html2haml utility moved to the html2haml gem
* Textile and Maruku filters moved to the haml-contrib gem

For more info see:

http://rubydoc.info/github/haml/haml/file/CHANGELOG.md
```

5.Originally, the default database inside Rails is Sqlite3 and you do not need to install it. However, we use postgresql database in this project, for better matching Heroku in later period. Fortunately, Cloud9 has installed Postgresql database already, so we just need to enter sudo service Postgresql start in the terminal, and then Postgresql database will be started.

At this point if we build the table directly, there will generate coding error: PG: : Error: Error: the new encoding UTF8) is incompatible, according to this. According to there, we will run the following code line by line in terminal to solve above problem:


```
// enter postgresql database
pengzhaoqing:~/workspace (master) $ psql
psql (9.3.14)
Type "help" for help.

ubuntu=# UPDATE pg_database SET datistemplate = FALSE WHERE datname = 'template1';
UPDATE 1
ubuntu=# DROP DATABASE template1;
DROP DATABASE
ubuntu=# CREATE DATABASE template1 WITH TEMPLATE = template0 ENCODING = 'UNICODE';
CREATE DATABASE
ubuntu=# UPDATE pg_database SET datistemplate = TRUE WHERE datname = 'template1';
UPDATE 1
ubuntu=# \c template1
You are now connected to database "template1" as user "ubuntu".
template1=# VACUUM FREEZE;
VACUUM
template1-# \q
```

6.After the database is ready, we establish the table by the following instructions:

```
pengzhaoqing:~/workspace (master) $ rake db:create:all
```

Then migrating data:

```
pengzhaoqing:~/workspace (master) $ rake db:migrate
== 20160818081955 CreateUsers: migrating ======================================
-- create_table(:users)
   -> 0.0328s
-- add_index(:users, :email, {:unique=>true})
   -> 0.0081s
== 20160818081955 CreateUsers: migrated (0.0412s) =============================

== 20160907152104 CreateCourses: migrating ====================================
-- create_table(:courses)
   -> 0.0117s
== 20160907152104 CreateCourses: migrated (0.0118s) ===========================

== 20160909105514 CreateGrades: migrating =====================================
-- create_table(:grades)
   -> 0.0192s
== 20160909105514 CreateGrades: migrated (0.0194s) ============================
```

At last, writing seed data:

```
pengzhaoqing:~/workspace (master) $ rake db:seed
```

7.Click Run Project button in IDE, then the URL of the website can be seen in log. You can click this URL to enter the project presentation page:

![这里写图片描述](http://img.blog.csdn.net/20160923211739243)

![这里写图片描述](http://img.blog.csdn.net/20160923211811306)



##End

Now, this program run successfully. You can find more details about this project [here](https://github.com/PENGZhaoqing/CourseSelect). Next time I will decompose this project and write from very beginning.

If you think this program is good, please click the star for this program in the top right corner of Github page~
