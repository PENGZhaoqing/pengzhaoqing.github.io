---
date: 2016-11-06 
title: Rails Web App Learning in action (3) -the basic version of students selective courses Contents
categories: Rails教程
tags: [rails]
---

## 1.Route

Rails route can identify the URL and distribute it to the controller for processing. Besides, Rails route can also generate the path and URL, without directly hardcoding the character string in the view.

Let's open the file named `config/route.rb` and add a line: 

``` ruby
Rails.application.routes.draw do

  resources :courses
  
end
```

Run it in the terminal: rake routes will check the router address and the following output can be obtained:

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

Taking the first line as the example, we give more details:

 - `/courses` denotes the URL, which has specified the certain address under this domain name. For example：`http://whatever.com/courses`
 - `GET` represents the HTTP GET method employed in the process of access, and you can find more details [here](http://www.w3school.com.cn/tags/html_ref_httpmethods.asp)
 - `courses#index` means that Rails will deliver this request to the `index` method in `courses_controller` for processing when users try to access the address of `/courses`.
 - `course` corresponding to the prefix field represents the representation of the URL address in Rails. In other words, Rails will return a character string “url：`/courses`” when the method named `course_path` has been called.

Each line behind can be interpreted according to the above four points, while the empty part represents that this line is the same as the previous line. For instance, the prefix field in the second row is empty means that its prefix is the same as the first line, which is to say the prefix field in the second row is `courses`.

> `resources :courses`can lead Rails to generate 8 standard URL address automatically, including the CURD (Create, Update, Retrieve and Delete) operation.
> 
> <font color=#DC143C>Here is just a brief introduction and you can refer to [Rails Routes](http://guides.ruby-china.org/routing.html) for better understanding of Rails route.</font>
> 


----------


## 2.Course controller

Controller embodies the logic implementation of Rails and is the core of the entire web application. It deals with every URL request while models (database) are used in this process. Finally, the result will be delivered to view by controller and the view will return the result to the user's browser after rendering it.

Let’s run `rails generate controller courses`to create course controller:

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

Open `app/controllers/courses_controller`,you can just created and add the following code:

``` ruby
class CoursesController < ApplicationController

  def new
    @course=Course.new
  end

  def create
    @course = Course.new(course_params)
    if @course.save
      redirect_to courses_path, flash: {success: "new course has been successfully registered"}
    else
      flash[:warning] = "wrong info, try it again"
      render 'new'
    end
  end

  def edit
    @course=Course.find_by(id: params[:id])
  end

  def update
    @course = Course.find_by_id(params[:id])
    if @course.update_attributes(course_params)
      flash={:info => "update successfully"}
    else
      flash={:warning => "update failed"}
    end
    redirect_to courses_path, flash: flash
  end

  def destroy
    @course=Course.find_by_id(params[:id])
    @course.destroy
    flash={:success => "successfully delete course: #{@course.name}"}
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

Six methods have been defined in this controller:

 1. **new**：`new` is responsible for creating a new course. Its logic is very simple: Declare a new object of the Course class, and then deliver it to the view
 2. **create**： `create` is responsible for receiving the data of the new courses and storing them. Its logic: Call the method named`course_params`private to get the data of the course, and deliver it to the newly created course object, then save this object. It is worth mentioning that the data will be checked when the object is stored. If the verification can pass (the data exists and conforms to the specification), that is, @ course.Save returns true, then the page will call redirect_to method to redirect to course_path. If the verification fails or there is an error in the saving process, that is, the return value of `@course.save` is `false`, then the `render` method will be called to render the view of the `new` method again, besides, users should edit and submit again.
 If the inspection (verified by the data and meet the specification, namely `@course.save`) returns `true`, then the page will be redirected to the `course_path` call `redirect_to` URL; if the inspection did not pass validation or error occurred during storage, `@course.save` returns `false`, then invokes the `render` method, `new` method, re render the view that users submit re editing
 3. **index**：`index` is responsible for demonstrating all objects. Its logic: Call all method in Course class to return `all` course objects, and then deliver them to the view.
 4.  **edit**：`edit` is responsible for editing an existing course. Its logic is very simple: Find the course object that needs to be edited by `Course.find_by(id: params[:id])`. params[:id] denotes the id of the course that needs to be edited, which is read from the URL by Rails. For example, the 17 in `http://whatever.com/course/17`.
 5. **update**： `update` is responsible for receiving the data of the course which has been edited. Its logic: Through `Course.find_by(id: params[:id])`, find the course object that has been edited already, call `update_attributes` to receive and update the new parameter, then redirect to `courses_path` whether the update is successful or failed. 
 6. **destory**：`destroy` is responsible for deleting the existing courses. Its logic: Find the course object, delete it, and redirect to  `courses_path`.

In fact, there is another method:**show**. This method is responsible for displaying the details of a course, and its logic is the same as edit method.

> **These seven methods have constituted standard four functions (CURD) of the Rails application: Create corresponds to `new` and `create`; Update corresponds to `destroy`; Retrieve corresponds to `edit` and `update`; Delete corresponds to `show`.**

In addition, method called `params` can return all the data involved in a URL request. The flash, as a Hash table, plays a part as prompt after the redirection, which can let users know the result of their operation (success, failure, or need to log in again, etc.).


----------


## 3.Course view

After being processed by a method in the controller, user's request will be handed over to the corresponding view. The view generally consists of JS, CSS and HTML files.

> <font color=#DC143C>If you have no idea about HTML, you can find more knowledge about it [here](http://www.w3school.com.cn/html/html_attributes.asp).
 <font>

Here we use the front-end libraries of Bootstrap. Boostrap is designed for the back-end programmers and can help a programmer who is not good at the front-end to write a relatively good front-end. It is actually a CSS, JS library.

> <font color=#DC143C> [Here](http://www.w3schools.com/bootstrap/) is more information about bootstrap<font>

In the future, we will use the standard JS library called jQuery, which has encapsulated all kinds of methods of JS.

> <font color=#DC143C> [Here](http://www.w3school.com.cn/jquery/) is more information about JQuery.<font>


----------

###<font color=#FF7F50> Please make sure that you have a certain knowledge of HTML and Bootstrap before looking at the following contents:<font>

----------


1.In order to use the external libraries, we should open the Gemfile under the root directory and fill in following information: 


``` ruby
# for bootstrap
gem 'bootstrap-sass', '~> 3.3.7'
# Use ActiveModel has_secure_password
gem 'bcrypt', '~> 3.1.11'
```

We use `bundle install` to install this external library.

2.在Delete `application.css` in `assets/stylesheets`, then add file `application.scss` and the following code to tell Rails to import CSS file related to boostrap.

``` ruby 
@import 'bootstrap-sprockets';
@import 'bootstrap';
@import 'bootstrap/theme';
```


3.Add `//= require bootstrap` in `assets/javascripts/application.js` to tell Rails to import js file related to boostrap. 

4.Create folder named `views/courses` to store view files of the various methods of `course_controller`. 

> **Rails’ default type of view file is `filename.html.erb` instead of simple HTML file, due to that Rails need to make computations in this file. But it will be compiled into HTML file in the end.**

5.Create file`views/courses/new.html.erb`and then fill in following information:

``` ruby
<div class="container-fluid">
  <div class="row">
    <div class="col-md-offset-3 col-sm-6">
      <div class="panel panel-success">
        <div class="panel-heading">
          <h3 class="text-center">new course</h3>
        </div>
        <div class="panel-body">
          <%= render "courses/form" %>
        </div>
      </div>
    </div>
  </div>
</div>
```

This is a simple view files. `div` is the tag of HTML tags and the `class` attribute of div controls the rendering mode of each label through boostrap.

> **In above code, statements in parentheses and the percent signs will be executed by ruby. For example, `< % = the code will be dynamically executed by ruby % >`.**

 - `<%= render "courses/form" %>`means that the content in partial view file named `courses/_form.html.erb`will be rendered. In other words, the content`courses/_form.html.erb`will be replaced to here. **Because some codes of view will be used several times, for improving the reusability of the code, Rails has extracted these codes to set up a file, which can be called in other places. This kind of view is called partial view, and an underline should be added when a partial is being named, such as `_form.html.erb`**.

6.Create`view/courses/_form.html.erb`and fill in the following code:

``` ruby 
<%= form_for @course, html: {class: 'form-horizontal', role: 'form'} do |f| %>

    <div class="form-group">
      <%= f.label "course name", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :name, class: 'form-control', placeholder: "input course name" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "course kind", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :course_type, class: 'form-control', placeholder: "input course kind" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "teaching way", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :teaching_type, class: 'form-control', placeholder: "input teaching way" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "exam type", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :exam_type, class: 'form-control', placeholder: "input exam type" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "limit num", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :limit_num, class: 'form-control', placeholder: "input limit num" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "credit", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :credit, class: 'form-control', placeholder: "input credit" %>
        </div>
      </div>
    </div>


    <div class="form-group">
      <%= f.label "classroom", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :class_room, class: 'form-control', placeholder: "classroom" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "course weeks", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :course_week, class: 'form-control', placeholder: "course weeks" %>
        </div>
      </div>
    </div>

    <div class="form-group">
      <%= f.label "course time", class: 'col-sm-3 control-label' %>
      <div class="col-sm-9">
        <div class="input-group">
          <div class="input-group-addon"><span class="glyphicon glyphicon-arrow-right"></span></div>
          <%= f.text_field :course_time, class: 'form-control', placeholder: "input credit" %>
        </div>
      </div>
    </div>

    <%= f.submit 'submit', class: "btn btn-success btn-block" %>
    <%= link_to 'cancel', courses_path, :class => "btn btn-default btn-block" %>

<% end %>
```

In addition to adding div tags for looking pretty, we should extract the main content:

``` ruby
<%= form_for @course do |f| %>

  <%= f.label "course name" %>
  <%= f.text_field :name %>
  
  .......
   
  <%= f.submit 'submit' %>
  <%= link_to 'cancel', courses_path %>

<% end %>
```

form_for method has constructed a form and the submission object of the form is `@course`.`f.text_field` specifies the field submissions and ` f.submit` states the submit button. `<%= link_to ' cancel ', courses_path %>` adds a link for the table, which means that the table has linked to the URL of `courses_path`, and "cancel" is displayed in this link.

><font color=#DC143C>  You can find more details about the way the form is constructed in [helper](http://guides.ruby-china.org/form_helpers.html)<font>


7.Create the view of edit method named `views/courses/index.html.erb`. Both edit view and new view apply the partial view of `courses/_form.html.erb`:

``` ruby
<div class="container-fluid">
  <div class="row">
    <div class="col-md-offset-3 col-sm-6">
      <div class="panel panel-success">
        <div class="panel-heading">
			<h3 class="text-center">modify the course: <%= @course.name %></h3>
        </div>
        <div class="panel-body">
          <%= render "courses/form" %>
        </div>
      </div>
    </div>
  </div>
</div>
```

8.Finally, we create the view of index method, `views/courses/index.html.erb`:

``` ruby
<div class="container-fluid">
  <div class="row">
  
     <div class="col-md-offset-2 col-sm-8">

      <div class="panel panel-primary">
          <div class="panel-heading">
	      <h3 class="panel-title">course list</h3>
	      </div>
      </div>

        <div class="panel-body">
          <table class="table table-responsive table-condensed table-hover">
            <thead>
            <tr>
              <th>course code</th>
              <th>course name</th>
              <th>credit</th>
              <th>exam way</th>
            </tr>

            <tbody>
            <% @course.each do |course| %>
                <tr>
                  <td><%= course.course_code %></td>
                  <td><%= course.name %></td>
                  <td><%= course.credit %></td>
                  <td><%= course.exam_type %></td>
                  <td><%= link_to "edit", edit_course_path(course), class: 'btn btn-xs btn-info' %></td>
		          <td><%= link_to "delete", course_path(course), :method => "delete", class: 'btn btn-xs btn-danger', :data => {confirm: 'Are you sure?'} %></td>
                </tr>
            <% end %>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</div>
```

 - The function of index view is to list all courses in table form, and add edit and delete button behind each course.

## 4.Run

1.Click run project and input`https://your C9 user name and project name/courses/new` in browser, such as:`https://course-select-dev-pengzhaoqing.c9users.io/courses/new`, then you can see the returned view of the new method under the control of courses:

![这里写图片描述](http://img.blog.csdn.net/20160927201044232)

2.Click “submit” and you will jump to the page of index method（`https://course-select-dev-pengzhaoqing.c9users.io/courses`）, which lists all of the course data.

![这里写图片描述](http://img.blog.csdn.net/20160927201249563)

3.Click `edit` behind every course will jump to the editing page(`https://course-select-dev-pengzhaoqing.c9users.io/courses/2/edit`),Click `submit` again, then you will jump to the page of index method. 

![这里写图片描述](http://img.blog.csdn.net/20160927201531973)

4.Click `delete` behind every course will remove this course, and jump to the page of index method.

> <font color=#DC143C>  The logic of above four steps has been reflected in the various methods of courses_controller. If you're interested, you can try to find how the skip has been realized in the controller, and understand how the view cooperates with the controller and another view.<font>
> 
> What’s more, the new page and the edit page have used the same partial view, so it seems that there is no difference. In fact, that one template can be applied to implement two functions (new and edit) reuses the code, so that can save developers from repeating the same code over and over.


----------


## 5.End

At this point, I have introduced the basic knowledge of the Rails framework, which is just about the simple functions such as adding or deleting an objective. And we will deal with the user login function and access control function later. For the further functions are based on this basic version, please make sure you have fully understand this tutorial before you go to the next.

> <font color=#DC143C>If you have trouble understanding the above content, read the [official guidelines](http://guides.ruby-china.org/getting_started.html) and then come back to this tutorial.<font>

If you have any queries, please leave a message to me on https://github.com/PENGZhaoqing/CourseSelect
