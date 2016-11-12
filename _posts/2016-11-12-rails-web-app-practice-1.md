---
date: 2016-11-12 
title: 校园选课系统－课程开放关闭功能
categories: Rails教程
tags: [ruby rails]
---


这里是国科大研究生课程（高级软件工程）[校园选课系统样本](https://github.com/PENGZhaoqing/CourseSelect) 的项目作业（1），要求同学们完成下面功能：

 - 增加选课的开放、关闭功能

需求如下：

1. 同学们登录后，选择课程里只能看到已经开放的课程
2. 老师可以对课程进行开发和关闭操作


## 功能展示

首先将最后要完成的功能展示出来：

1.当同学登录后：

![这里写图片描述](http://img.blog.csdn.net/20161109172708063)

默认开始所有的课程都是未开放的，所以该同学看不到任何课程


2.某老师开放了自己的课程：

![这里写图片描述](http://img.blog.csdn.net/20161109173000835)

点击开课按钮，页面显示开课成功，并将按钮更新为关闭

![这里写图片描述](http://img.blog.csdn.net/20161109173122134)


3.学生再次登陆后，能看到刚刚被开放的课程

![这里写图片描述](http://img.blog.csdn.net/20161109173229073)


## Hint
 
 1.首先给course表增加一个字段`open`，类型boolean，用来判断该课程是否开放或关闭

>   **Tips:** 运行`rails generate migration add_open_attribute`产生新的数据迁移文件，然后用`add_column`添加`open`字段，用`default`参数制定此字段默认为`false`，请参考[这里](http://guides.rubyonrails.org/active_record_migrations.html)，最后运行`rake db:migrate` 往数据库里写入字段


2.在course_controller文件中的方法list中，加入代码来筛选出已经开放的课程

``` ruby

class CoursesController < ApplicationController
 
 ...
 
 def list
    @course=Course.all
    @course=@course-current_user.courses
    填入你的代码，找到已经开放的课程传给视图
  end
  
  ...
  
end
```

> Tips: 用一个循环语句遍历所有的课程，把所有已经开放的课程放到另外一个数组中，然后把这个数组传到视图中

3.在route.rb文件代码片段中，添加open和close的路由url，方法都为get

``` ruby
resources :courses do
    member do
      get :select
      get :quit
    end
    collection do
      get :list
    end
  end
```

> Tips：修改完后运行`rake routes`查看路由详情

4.在courses_controller.rb中添加open和close两个方法，用来处理当老师点击按钮时，将课程的状态设为开放或者关闭，设置完以后重定向至index方法，（参考update_attribute方法）

``` ruby
class CoursesController < ApplicationController
 ...
 
 def open
    填写你的代码
    redirect_to courses_path, flash: {:success => "已经成功开启该课程:#{ @course.name}"}
  end

  def close
   填写你的代码
    redirect_to courses_path, flash: {:success => "已经成功关闭该课程:#{ @course.name}"}
  end
  
  ....
end

```

5.在视图文件`course/index.html.erb`文件中，填写两种按钮的代码，参考`link_to`方法


``` ruby
  <% @course.each do |course| %>
                <tr>

                  <td><%= course.course_code %></td>
                  <td><%= course.name %></td>
                  <td><%= course.credit %></td>
                  <td><%= course.exam_type %></td>
                  <td><%= course.teacher.name %></td>

                  <% if teacher_logged_in? %>
                      <td><%= link_to "编辑", edit_course_url(course), class: 'btn btn-xs btn-info' %></td>

		              在这里填写相关按钮的代码

                      <td><%= link_to "删除", course_path(course), :method => "delete", class: 'btn btn-xs btn-danger', :data => {confirm: '确定要删除此课程?'} %></td>
                  <% elsif student_logged_in? %>
                      <td><%= link_to "删除", quit_course_path(course), class: 'btn-sm btn-danger' %></td>
                  <% end %>
                </tr>
            <% end %>
```

## 结束语

上面功能开发一共不超过30行代码就能完成，提倡同学在这个基础上提出更多的改进，例如将选择课程的计数改为当前能选择的课程的数量，如图：

![这里写图片描述](http://img.blog.csdn.net/20161109180213580)

左侧选择课程后面的数字应该为1
