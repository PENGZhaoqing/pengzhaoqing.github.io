---
date: 2016-10-25
title: Rails Web App Learning in action (2)--the basic version of students selective courses
categories: Translation
tags: [rails]
---


In last tutorial, the demo code has been run successfully in Cloud9. Now we will create a new Rails application from very beginning.

Let’s have a look at the framework of Rails before writing code:

![Rails框架](http://img.blog.csdn.net/20160920232503121)

The most important feature of the framework of Rails is following the design pattern of MVC (M:model, V:view, C:controller), that is, the pattern of controller-view-model. The framework of Rails implements the code logic, database operation and web elements (HTML, JS, CSS) which are interacted with users in controller, model and view respectively. Separating three functions mentioned above can help Rails maintain a clear framework.

A routing process is left in the first step in this figure.

>We use a complete process to illustrate above process. Let's assume that users enter `http://whatever.com/users/index` into browser and then submit it. When the server, whatever.com, receives this request, the request will be handed over to the `index` methods in `users_controller` for later processing. Why the server knows how to put this request to the certain controller and the `index` method under it?

Because Rails have a file named `routes. rb`, which has been configured the mapping relationship of the request. Thus, when Rails need to handle the user request, it seek for the mapping relationship in file routes. Rb, and then hand over the request to the appropriate controller and the method according to the result of mapping. `Users `and `index` are called resources and the method of processing resources respectively. This process is called routing. Rails uses the concept of RESTful when it represents the resources.


----------


## 1.Create new Rails application


1.Click create a new workspace in the interface after login C9. 

![新的应用](http://img.blog.csdn.net/20160920225710475)

Above configuration will generate new Rails App by default.

2.C9 generate a document template in Rails framework for us. Firstly, let's have a look at the file tree:

![这里写图片描述](http://img.blog.csdn.net/20161025110640053)

- Controllers: Contents of control file
- Models: Contents of model file
- Views: Contents of view file
- Assets: Storing front files, such as CSS, JS, etc.
- Initializers: Files in the initializers should be automatically loaded and configuration of some external libraries should be initialized, before start the framework of Rails.
- database.yml: Describing the configuration of how to connect to the database and how to use varies kind of databases, as well as the name and the password of the database.
- routes.rb: Routing file, which describes how to convey user’s request to the particular controller. db: Storing the files related to database, including data migration file and seed file.

3.click Run Project button, then we can run this new Rails App


----------

**If you have trouble understanding Rails framework, read the official guidelines—-[Simple Blog](http://guides.rubyonrails.org/getting_started.html)**

----------


## 2. Create model

The development process of Rails can be divided into the following steps: Firstly, create model, then establish the corresponding controller, next configure related route, after that, create the corresponding method under the controller, finally, construct view of every corresponding method.

***What is model? Model represents a class of the underlying data table and encapsulates various methods of operating this data table.***

1.For developing a selective courses system, we definitely need a data table to store user's information, and another data table to store the information of courses, so we run the following command to create the user model and the course model.

``` ruby
pengzhaoqing:~/workspace $ rails generate model course
Running via Spring preloader in process 3251
      invoke  active_record
      create    db/migrate/20160920163906_create_courses.rb
      create    app/models/course.rb
      invoke    test_unit
      create      test/models/course_test.rb
      create      test/fixtures/courses.yml
pengzhaoqing:~/workspace $ rails generate model user
Running via Spring preloader in process 3262
      invoke  active_record
      create    db/migrate/20160920163927_create_users.rb
      create    app/models/user.rb
      invoke    test_unit
      create      test/models/user_test.rb
      create      test/fixtures/users.yml
```

 - We only need to use `rails generate model course` instruction and Rails will generate the models automatically (Here, such as `db/migrate/20160920163906_create_courses.rb`, `app/models/course.rb` and test files), which can leave out the trouble of generating a file by themselves.
   

2.Now we open the file, `db/migrate/20160920163906_create_courses.rb`, and fill in following information:

``` ruby
class CreateCourses < ActiveRecord::Migration
  def change
    create_table :courses do |t|

      t.string :name
      t.string :course_code
      t.string :course_type
      t.string :teaching_type
      t.string :exam_type
      t.string :credit
      t.integer :limit_num
      t.integer :student_num, default: 0
      t.string :class_room
      t.string :course_time
      t.string :course_week
      t.belongs_to :teacher

      t.timestamps null: false
    end
  end
end
```

 - This file describes the field and field type in class `courses`, and Rails will create data table according to the description in this file. We don't configure file `database.yml`. The default database inside Rails is Sqlite3 and you can find whether the `adapter` in this file is pointing `sqlite3` or not.




3.Similarly, we open the file, `db/migrate/20160920163927_create_users.rb`, and fill in following information:

``` ruby
class CreateUsers < ActiveRecord::Migration
  def change
    create_table :users do |t|

      t.string :name
      t.string :email
      t.string :num
      t.string :major
      t.string :department
      t.string :password_digest
      t.string :remember_digest

      t.boolean :admin, default: false
      t.boolean :teacher,default: false
      t.timestamps null: false
    end

    add_index :users, :email, unique: true

  end
end
```

 - We apply admin and teacher of Boolean field type to distinguish the three roles for users can be categorized into three groups, including student, teacher and administrator.
 -  `add_index :users, :email, unique: true` 
	 - The preceding line indicates that we construct index for `email` field in users table, in order to retrieve the `user` according to user's mailbox quickly.
 


4.In selective courses system, one students may have more than one classes and one course can be chosen by many students, so it is a many-to-many relationship between students and courses.

Besides, each course of one student only has one grade. There are three data entities involved here: student, course and grade. Thus, we need to establish a data table about grade, which should store user_id of student table, course_id of course table and the corresponding grade.

Now, run `rails generate model grade` to create grade model:


``` ruby
pengzhaoqing:~/workspace $ rails generate model grade
Running via Spring preloader in process 1144
      invoke  active_record
      create    db/migrate/20160921051153_create_grades.rb
      create    app/models/grade.rb
      invoke    test_unit
      create      test/models/grade_test.rb
      create      test/fixtures/grades.yml
```

Get in `db/migrate/20160921051153_create_grades.rb` to generate the corresponding field:

``` ruby
class CreateGrades < ActiveRecord::Migration
  def change
    create_table :grades do |t|
      t.belongs_to :course, index: true
      t.belongs_to :user, index: true
      t.integer :grade
      t.timestamps null: false
    end
  end
end
```

 - `t.belongs_to: :course` is as same as `t.integer :course_id` and they both record the course id. The former can express the dependency relationship between models more clearly. 


 >  **One class has multiple grades while one grade only belongs to one class; one student has multiple grades while one grade only belongs to one student; two attributes, students and course, can determine the only grade.**

5.Another crucial question is: **one teacher can teach several courses while each course only belongs to one teacher**, so it is a one-to-many relationship between teachers and courses (in order to simplify the issue, we do not design a many-to-many relationship here). Now review the code `t.belongs_to :teacher`, which is written in the course model’s data migration file `db/migrate/20160920163906_create_co urses.rb`. This code has already stored the user (teacher) id as a foreign key in the table of course model, **for realizing the one-to-many relationship between user (teacher) and course. Namely, you can get all of the course data related to one id by looking for this id in the data table of course model.**


6.We draw the ER diagram directly to see the relationship between these models: 


![这里写图片描述](http://img.blog.csdn.net/20161025112739293)

 - To better understand this figure requires certain knowledge of database, so you can find more detailed in [Rails Active Association](http://guides.ruby-china.org/association_basics.html) . 



> **It is important to note that the user model and course model associated with each other through grade model, because you can search all related course_id according to user_id, and vice versa. In other words, the many-to-many relationship between user model and course model had been achieved.**

7.Now, we have completed the process of establishing the data fields of model. Then we need to convey the association relationship between these fields to the Rails framework, so that Rails can organize the association relationship between models automatically. Firstly, open `app/models/course.rb`, and enter:

``` ruby
class Course < ActiveRecord::Base

  has_many :grades
  has_many :users, through: :grades

  belongs_to :teacher, class_name: "User"

  validates :name, :course_type, :course_time, :course_week,
            :class_room, :credit, :teaching_type, :exam_type, presence: true, length: {maximum: 50}

end
```

 - Class model associates with the one-to-many relationship of other models by method called `has_many`. Parameter `through` describes that this one-to-many relationship relates to a user model through grade model. 
 - Method `belongs_to` describes the one-to-many relationship with other models and `class_name` appoints the class name of user model, so that Rails can find the correlative user model correctly. 
 - Method `validate` appoints the verification of the field under the model. Eight fields are verified here`:name, :course_type, :course_time, :course_week, :class_room, :credit, :teaching_type, :exam_type`. The requirement of the verification is `presence: true, length: {maximum: 50}`, which represents that each field must exist (not empty) and can't more than 50 characters. **The verification will be performed every time when the course data saved.**


8.Next, open `app/models/grade.rb` and write:

``` ruby
class Grade < ActiveRecord::Base
  belongs_to :course
  belongs_to :user
end
```

 - The one-to-many relationship between user and course has been determine here.

9.Then open `app/models/user.rb` and write:

``` ruby
class User < ActiveRecord::Base

  before_save :downcase_email
  attr_accessor :remember_token
  validates :name, presence: true, length: {maximum: 50}
  validates :password, presence: true, length: {minimum: 6}, allow_nil: true

  has_many :grades
  has_many :courses, through: :grades

  has_many :teaching_courses, class_name: "Course", foreign_key: :teacher_id

  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: {maximum: 255},
            format: {with: VALID_EMAIL_REGEX},
            uniqueness: {case_sensitive: false}

  #1. The ability to save a securely hashed password_digest attribute to the database
  #2. A pair of virtual attributes (password and password_confirmation), including presence validations upon object creation and a validation requiring that they match
  #3. An authenticate method that returns the user when the password is correct (and false otherwise)
  has_secure_password
  # has_secure_password automatically adds an authenticate method to the corresponding model objects.
  # This method determines if a given password is valid for a particular user by computing its digest and comparing the result to password_digest in the database.

  # Returns the hash digest of the given string.
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
        BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  def User.new_token
    SecureRandom.urlsafe_base64
  end

  def user_remember
    self.remember_token = User.new_token
    update_attribute(:remember_digest, User.digest(remember_token))
  end

  def user_forget
    update_attribute(:remember_digest, nil)
  end

  # Returns true if the given token matches the digest.
  def user_authenticated?(attribute, token)
    digest = self.send("#{attribute}_digest")
    return false if digest.nil?
    BCrypt::Password.new(digest).is_password?(token)
  end

  private

  def downcase_email
    self.email = email.downcase
  end

end
```

 - You should notice the following information in user model： 
	 - `has_many :courses, through: :grades` 
	 - `has_many :teaching_courses, class_name: "Course", foreign_key: :teacher_id`

These two lines of codes both appoint one-to-many relationship with the course model. We can find that Rails can work normally while the first line of codes does not appoint the name and foreign key field of course model. This reflects one characteristics of Rails: **Convention over Configuration**, which means that as long as the developers write code according to a certain mode, the trouble of writing XML file to specify the file path can be averted. As for `has_many: courses, through:: grades`, by default, Rails will look for to see if there is a file called course.rb (change plural into singular) and will search the course model to see if there is a foreign key field named user_id. However, Rails cannot find the `course.rb` file correctly according to `teaching_courses` in the second line of codes. This is not simply change plural into singular. So, we need to manually assign it to find a model named `Course` class and the foreign key `teacher_id` of relational model.


 - The methods defined later, such as `digest, new_token, user_remember, user_forget, user_authenticated? and downcase_email` are standard functions which are required to achieve the user login function. You can find more details [here](https://www.railstutorial.org/book/advanced_login).


10.All models and their data migration files have been generated now, then we run data migration `rake db:migrate` to build data table. Seeing the following information means you have successfully completed our task.

``` ruby
pengzhaoqing:~/workspace $ rake db:migrate
== 20160920163906 CreateCourses: migrating ====================================
-- create_table(:courses)
   -> 0.0079s
== 20160920163906 CreateCourses: migrated (0.0080s) ===========================

== 20160920163927 CreateUsers: migrating ======================================
-- create_table(:users)
   -> 0.0014s
-- add_index(:users, :email, {:unique=>true})
   -> 0.0011s
== 20160920163927 CreateUsers: migrated (0.0028s) =============================

== 20160921051153 CreateGrades: migrating =====================================
-- create_table(:grades)
   -> 0.0039s
== 20160921051153 CreateGrades: migrated (0.0042s) ============================
```

## End

At this point, the model part is finished. The model part is mainly to handle the data, including the establishment of data table and the correlation between data tables.


Here are some references to help you. 

 - [Active Record Basic](http://guides.rubyonrails.org/active_record_basics.html)
 - [Active Record Migration](http://guides.rubyonrails.org/active_record_migrations.html)
 - [Active Record  Verification](http://guides.rubyonrails.org/active_record_validations.html)
 - [Active Record Association](http://guides.rubyonrails.org/association_basics.html)

Next chapter is about controller.
