---
date: 2016-09-22
title: Rails Web应用开发实战－学生选课系统基础版(三)
categories: Rails教程
tags: [rails]
---

上次主要讲了MVC中的模型，此教程我们开始涉及控制器和视图。

----------

## 1.路由

Rails 路由能识别 URL，将其分发给控制器的动作进行处理，还能生成路径和 URL，无需直接在视图中硬编码字符串。

我们打开`config/route.rb`文件，加入一行：

``` ruby
Rails.application.routes.draw do

  resources :courses

end
```

然后在终端中运行：`rake routes` 查看路由地址，得到下面的输出：

``` ruby
pengzhaoqing:~/workspace $ rake routes
     Prefix Verb   URI Pattern                 Controller#Action
    courses GET    /courses(.:format)          courses#index
            POST   /courses(.:format)          courses#create
 new_course GET    /courses/new(.:format)      courses#new
edit_course GET    /courses/:id/edit(.:format) courses#edit
     course GET    /courses/:id(.:format)      courses#show
            PATCH  /courses/:id(.:format)      courses#update
            PUT    /courses/:id(.:format)      courses#update
            DELETE /courses/:id(.:format)      courses#destroy
```

以第一行为例进行解释：

 - `/courses` 指的是URL路径，指定了用户在此域名下要访问的地址，例如：`http://whatever.com/courses`
 - `GET` 指访问的过程中用的HTTP GET方法，详见[这里](http://www.w3school.com.cn/tags/html_ref_httpmethods.asp)
 - `courses#index` 表示了当用户访问`/courses`地址时，Rails将会把这个请求交给控制器`courses_controller`中的`index`方法处理
 - 而prefix字段对应的`course`代表了，URL地址在Rails中的表示形式，即在Rails中调用`course_path`这个方法时，Rails将会解析返回一个字符串url：`/courses`

后面每一行都可以按照上面的四个小点进行解释，而其中空的部分代表与上一行相等（如第二行中的prefix字段为空，表示它的prefix与上一行相等，即也为`courses` ）

> 一句简单的`resources :courses`，Rails将会自动生成8个标准的URL地址，包括了CURD（创建Create、更新Update、读取Retrieve和删除Delete操作）
>
> 这里只是一个简单的介绍，请参照 [Rails Routes](http://guides.ruby-china.org/routing.html)来理解Rails的路由


----------


## 2.课程控制器

控制器体现了Rails的逻辑实现，是整个web应用的核心，它处理着每个URl请求，而处理过程中调用了模型（数据库），最终将结果交给给视图，然后视图经过一定的渲染返回给用户的浏览器。

我们运行`rails generate controller courses`来创建课程控制器：

``` ruby

pengzhaoqing:~/workspace $ rails generate controller courses
Running via Spring preloader in process 1699
      create  app/controllers/courses_controller.rb
      invoke  erb
      create    app/views/courses
      invoke  test_unit
      create    test/controllers/courses_controller_test.rb
      invoke  helper
      create    app/helpers/courses_helper.rb
      invoke    test_unit
      invoke  assets
      invoke    coffee
      create      app/assets/javascripts/courses.coffee
      invoke    scss
      create      app/assets/stylesheets/courses.scss
```

打开刚才创建的app/controllers/courses_controller，填入以下代码：

``` ruby
class CoursesController < ApplicationController

  def new
    @course=Course.new
  end

  def create
    @course = Course.new(course_params)
    if @course.save
      redirect_to courses_path, flash: {success: "新课程申请成功"}
    else
      flash[:warning] = "信息填写有误,请重试"
      render 'new'
    end
  end

  def edit
    @course=Course.find_by(id: params[:id])
  end

  def update
    @course = Course.find_by_id(params[:id])
    if @course.update_attributes(course_params)
      flash={:info => "更新成功"}
    else
      flash={:warning => "更新失败"}
    end
    redirect_to courses_path, flash: flash
  end

  def destroy
    @course=Course.find_by_id(params[:id])
    @course.destroy
    flash={:success => "成功删除课程: #{@course.name}"}
    redirect_to courses_path, flash: flash
  end

  def index
    @course=Course.all
  end

  private

  def course_params
    params.require(:course).permit(:course_code, :name, :course_type, :teaching_type, :exam_type,
                                   :credit, :limit_num, :class_room, :course_time, :course_week)
  end

end
```

在这个控制器中一共定义了六个方法：

 1. **new**：负责创建一个新的课程，它的逻辑很简单：即申明一个新的Course类的对象，然后交给视图。
 2. **create**： 负责接收新创建课程的数据，然后保存，它的逻辑为：先调用名为`course_params`private类型的方法以获得课程的数据，并传入新创建的课程模型的对象。然后保存这个对象，而在保存这个对象时会进行对数据进行检查，若检查验证通过（各项数据都存在且符合规范），即`@course.save`返回
为`true`，那么页面就会调用`redirect_to`方法重定向至`course_path`这个URL；若检查验证未通过或者保存过程中发生错误，即`@course.save`返回为`false`，那么会调用`render`方法，重新渲染`new`方法的视图，让用户再次编辑提交
 3. **index**：负责对所有的对象进行展示，逻辑为：在Course类中调用`all`方法返回所有课程对象，然后交给视图
 4.  **edit**：负责对一个已有的课程进行编辑修改，它的逻辑也很简单：通过`Course.find_by(id: params[:id])`找到需要编辑的课程对象，其中params[:id]表示Rails从URL里读取的当前需要编辑课程的id，例如`http://whatever.com/course/17`中的17。
 5. **update**：负责接收更新修改后的课程数据，它的逻辑为：通过`Course.find_by(id: params[:id])`找到当前已经修改的课程对象，然后调用`update_attributes`接收新的参数并更新，无论更新成功或者更新失败，都重定向至`courses_path`
 6. **destory**：负责对已有的课程进行删除，逻辑为：找到要删除的课程对象，执行删除，重定向至 `courses_path`

这里其实还有一个方法未写进去：**show**，这个方法负责展示某个课程的详细信息，逻辑与前面的edit方法相同。

> **这一共七个方法构成了Rails应用标准的增删改查（CURD）四个功能：增为new和create方法，删为destroy方法，改为edit和update方法，查为show方法。**

另外，params方法能返回URL请求包含的所有数据，而flash作为一个Hash表（哈希表），起到了重定向后提示语的作用，能在用户操作后，告诉用户的操作的结果（成功、失败或者需要登录等）。

----------

## 3.课程视图

用户的请求经过了控制器里某个方法处理后，会被交给这个请求对应的视图。视图一般由JS、CSS和HTML文件构成。

> 若完全不了解HTML，请移步[这里](http://www.w3school.com.cn/html/html_attributes.asp)

这里我们使用Bootstrap前端库，Boostrap是一个专门为后端程序员设计的，可以让一个不懂前端的程序员写出比较良好的前端效果，其实就是一个CSS、JS库。

> 有关bootstrap请看[这里](http://www.w3schools.com/bootstrap/)

我们在将来会用到JS的标准库JQuery，封装了各种JS的方法。

> 有关JQuery请看[这里](http://www.w3school.com.cn/jquery/)

----------

<h2 style=" color:#FF7F50"> 在看下面的时候请确保对HTML和Bootstrap有一定的了解：</h2>

----------

1.为了使用这种外部库，我们打开根目录下Gemfile文件，在里面加入：


``` ruby
# for bootstrap
gem 'bootstrap-sass', '~> 3.3.7'
# Use ActiveModel has_secure_password
gem 'bcrypt', '~> 3.1.11'
```

使用`bundle install`安装这个外部库

2.在`assets/stylesheets`中把`application.css`删除，增加`application.scss`文件，并加入以下代码来告诉Rails导入boostrap相关的css文件

``` ruby
@import 'bootstrap-sprockets';
@import 'bootstrap';
@import 'bootstrap/theme';
```


3.在`assets/javascripts/application.js`中增加一行：`//= require bootstrap`，告诉Rails导入bootstrap相关的js文件

4.创建`views/courses`文件夹，用来存放`course_controller`的各个方法的视图文件。

> **Rails默认的视图文件类型为`filename.html.erb`，并不是单纯的HTML文件，这是因为Rails需要在这个文件中做一些运算，但最终会编译成HTML文件。**

5.创建`views/courses/new.html.erb`文件，在文件中写入：

``` html
<div class="container-fluid">
  <div class="row">
    <div class="col-md-offset-3 col-sm-6">
      <div class="panel panel-success">
        <div class="panel-heading">
          <h3 class="text-center">新课程</h3>
        </div>
        <div class="panel-body">
          <%= render "courses/form" %>
        </div>
      </div>
    </div>
  </div>
</div>
```

这就是一个简单的视图文件，`div`为HTML的标签，`div`的`class`属性控制了每个标签的渲染方式（通过boostrap）

> **上述代码中，被括号和百分号包裹的语句会被ruby执行，例如`<%= 这里的代码会被ruby动态执行 %>`**

 - `<%= render "courses/form" %>`这一句表示这里将会渲染局部视图`courses/_form.html.erb`中的文件内容，也就是将`courses/_form.html.erb`文件内容替换到这里。**因为有些视图代码将会被使用多次，Rails为了提高代码的复用性，将这些代码提取出来单独成立一个文件，在其他地方调用即可。这种视图被称为partial（局部视图），partial在命名的时候前面要加上下划线，如`_form.html.erb`**。

6.那么我们接着创建`view/courses/_form.html.erb`，填入以下代码：

``` html
<%= form_for @course, html: {class: 'form-horizontal', role: 'form'} do |f| %>

    <div class="form-group">
      <%= f.label "课程名", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :name, class: 'form-control', placeholder: "输入课程名" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "课程属性", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :course_type, class: 'form-control', placeholder: "输入课程属性" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "授课方式", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :teaching_type, class: 'form-control', placeholder: "输入授课方式" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "考试方式", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :exam_type, class: 'form-control', placeholder: "输入考试方式" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "限选人数", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :limit_num, class: 'form-control', placeholder: "输入限选人数" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "课时学分", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :credit, class: 'form-control', placeholder: "输入课时学分" %>
        </div>
      </div>
    </div>


    <div class="form-group">
      <%= f.label "教室", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :class_room, class: 'form-control', placeholder: "输入教室" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "上课周数", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :course_week, class: 'form-control', placeholder: "输入课时学分" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "上课时间", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :course_time, class: 'form-control', placeholder: "输入课时学分" %>
        </div>
      </div>
    </div>

    <%= f.submit '提交', class: "btn btn-success btn-block" %>
    <%= link_to '取消', courses_path, :class => "btn btn-default btn-block" %>

<% end %>
```

除了为美观而加上div标签，提取其主要内容为：

``` html
<%= form_for @course do |f| %>

  <%= f.label "课程名" %>
  <%= f.text_field :name %>

  .......

  <%= f.submit '提交' %>
  <%= link_to '取消', courses_path %>

<% end %>
```

form_for方法构造了一个表单，表单的提交对象是`@course`，`f.text_field`指定了提交的字段，` f.submit`申明了提交的按钮， `<%= link_to '取消', courses_path %>`这一句为表增加了一个链接，表示链接到了`courses_path`这个URL，链接上显示的是“取消”这两个字。

> 更多的关于表单构造方式详见[表单helper](http://guides.ruby-china.org/form_helpers.html)<font>


7.然后我们创建edit方法的视图`views/courses/edit.html.erb`，edit视图与new视图一样，使用了`courses/_form.html.erb`局部视图：

``` html
<div class="container-fluid">
  <div class="row">
    <div class="col-md-offset-3 col-sm-6">
      <div class="panel panel-success">
        <div class="panel-heading">
			<h3 class="text-center">更改课程: <%= @course.name %></h3>
        </div>
        <div class="panel-body">
          <%= render "courses/form" %>
        </div>
      </div>
    </div>
  </div>
</div>
```

8.最后创建index方法的视图`views/courses/index.html.erb`：

``` html
<div class="container-fluid">
  <div class="row">

    <div class="col-md-offset-2 col-sm-8">

      <div class="panel panel-primary">
        <div class="panel-heading">
          <h3 class="panel-title">课程列表 </h3>
        </div>
      </div>

      <div class="panel-body">
        <table class="table table-responsive table-condensed table-hover">
          <thead>
          <tr>
            <th>课程编号</th>
            <th>课程名称</th>
            <th>课时/学分</th>
            <th>考试方式</th>
          </tr>
          </thead>

          <tbody>
          <% @course.each do |course| %>
              <tr>
                <td><%= course.course_code %></td>
                <td><%= course.name %></td>
                <td><%= course.credit %></td>
                <td><%= course.exam_type %></td>
                <td><%= link_to "编辑", edit_course_path(course), class: 'btn btn-xs btn-info' %></td>
                <td><%= link_to "删除", course_path(course), :method => "delete", class: 'btn btn-xs btn-danger',
                                :data => {confirm: '确定要删除此课程?'} %>
                </td>
              </tr>
          <% end %>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</div>
```

 - index视图功能是把所有的课程用表格形式列举出来，并在每个课程后面加上编辑和删除按钮。

## 4.运行

1.点击run project，在浏览器中输入`https://你的c9用户名和项目名/courses/new`，例如：`https://course-select-dev-pengzhaoqing.c9users.io/courses/new`，就能看到courses控制下new方法返回的视图：

![](/assets/img/posts/2016-09-22-course-select-3/1.png)


2.点击提交，若成功会跳转至index方法的页面（`https://course-select-dev-pengzhaoqing.c9users.io/courses`），列举出所有的课程数据：

![](/assets/img/posts/2016-09-22-course-select-3/2.png)


3.在每一个课程后面点击编辑，会跳转至编辑页面（`https://course-select-dev-pengzhaoqing.c9users.io/courses/2/edit`），再次点击提交，若成功还会跳转至index方法页面：

![](/assets/img/posts/2016-09-22-course-select-3/3.png)

4.在每一个课程后面点击删除，会删除此课程，并跳转至index方法页面。

> <text style="color:red">以上4步的逻辑，都是在courses_controller的各方法里体现的，有心的学习者可以回头看看控制器里是怎么实现跳转的，并理解视图与控制器还有视图之间如何配合的</text>

> 而且，这里new和edit页面都使用了相同的局部视图，因此看起来没有什么差别，但却用一个模板实现了两种功能（新建和编辑），复用了代码，省去开发人员重复编写工作。

----------


## 5.结束语

到这里，基本的Rails框架就介绍全了，但是这还仅仅只是个很简单的增删改查功能，后面我们会介绍到用户登录功能，权限控制功能，都是在这个基础上进行的，所以请务必将此教程理解，再进行下个教程。

> 如果理解有困难，请先跟着[官方指南](http://guides.ruby-china.org/getting_started.html)走一遍，再回头看这个

若有疑问请在https://github.com/PENGZhaoqing/CourseSelect上提issue