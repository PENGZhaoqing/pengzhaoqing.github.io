---
date: 2016-09-21
title: Rails Web应用开发实战－学生选课系统基础版(二)
categories: Rails教程
tags: [rails]
---

在上一篇教程中，我们在Cloud9中跑通了整个演示代码，下面我们将从零建立一个新的Rails应用。

在我们写代码之前，我们先看看Rails框架的结构：

![](/assets/img/posts/2016-09-21-course-select-2/1.png)

Rails框架最主要的特点是遵循MVC的设计模式（M：model，V：view，C：controller），也就是控制器-视图-模型模式。Rails框架将代码逻辑实现在控制器（controller）中，将与数据库的操作实现在模型（model）中，将与用户交互的网页元素（HTML、JS、CSS）实现在视图中。通过分离这三大部分的功能，Rails能很好的维持一个清晰的架构。

在上图的第一步中还遗留了一个routing（路由）的过程。

> 我们利用一个完整的过程来阐述上述过程，假设用户在浏览器中输入这个网址并提交：`http://whatever.com/users/index`，当服务器（whatever.com）接收到这个请求，将会把请求交给控制器`users_controller`中的方法`index`处理，为什么服务器知道如何把这个请求交给这个控制器以及这个控制器下的`index`方法呢？

是因为Rails有个`routes.rb`文件，里面配置了请求的映射关系，因此Rails在每次处理用户请求时，都会去这个文件里查找映射关系，再根据映射的结果交给合适的控制器以及控制下的方法，在这里`users`叫做资源，`index`叫做处理资源的方法。这个过程就叫做路由。Rails在表示资源的时候利用了RESTful的理念。

----------


## 1.创建新的Rails应用

1.在C9登录后的界面中点击创建新的项目，如下：

![](/assets/img/posts/2016-09-21-course-select-2/2.png)

以上的配置会默认创建新的Rails App

2.C9为我们生成好了一个Rails框架的文件模板，首先我们看一下文件树：

![](/assets/img/posts/2016-09-21-course-select-2/3.png)

 - controllers：存放控制文件的目录
 - models：存放模型文件的目录
 - views：存放视图文件的目录
 - assets：存放前端文件，如CSS，JS等
 - initializers：Rails框架在启动前，会自动加载initializers中的文件，初始化一些外部库的配置
 - database.yml：描述了与数据库连接的配置，如使用何种数据库，数据库名和密码等等
 - routes.rb：路由文件，描述了如何将用户请求交给特定的控制器处理
 - db：存放数据库相关文件，包括数据迁移文件和种子文件
 - gemfile：描述了所需要的外部库，`bundle install` 这个指令就是安装所有在这个列表下的外部库。

3.点击Run Project，我们就可以运行这个新的Rails App


----------

**如果理解Rails框架有困难，请先看官方指南-[简易博客](http://guides.ruby-china.org/getting_started.html)**
------------------------------------------------------------------------

----------


## 2.创建模型（Model）

Rails开发流程是先创建模型，再创建相应控制器，配置相关路由，然后在控制器下创建相应方法，最后对应每个方法创建视图。

***模型是什么，模型代表了一个底层数据表的类，封装了操作这个数据表的各种方法。***

1.由于是学生选课系统，我们肯定需要一个数据表来储存用户的信息，需要一张数据表来储存课程的信息，那么我们运行以下指令创建用户模型和课程模型：

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

 - 我们只通过`rails generate model course` 指令，Rails就帮我们创建了
   `db/migrate/20160920163906_create_courses.rb`、`app/models/course.rb`、等test文件，省去了开发人员自己创建文件的麻烦。

2.现在我们到打开`db/migrate/20160920163906_create_courses.rb`文件，填入：

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

 - 这个文件描述了`courses`这个表中的字段和字段类型，Rails将会按照这个文件中描述的进行建立数据表。这里我们并没有配置`database.yml`文件，Rails默认的就是内置的Sqlite3数据库，可以自己去看看那个文件里面的`adapter`是不是指定的`sqlite3`




3.相同地，我们打开`db/migrate/20160920163927_create_users.rb`，填入：

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

 - 因为用户可以分为学生，老师和管理员三个角色，因此在用户的字段里，我们用 `admin` 和 `teacher`
   布尔类型的字段来区分三种角色。
 - `add_index :users, :email, unique: true`
   表示我们为`users`表的`email`字段建立索引，为以后能快速地根据用户邮箱检索到某个用户。

4.在选课系统中，一个学生有多门课，一门课可以被多名学生选择，因此学生和课程就是多对多的关系，而且，每个学生的每一门课都有一个成绩。这里涉及了三个数据实体，一个是学生，一个是课程，一个是成绩。所以，我们还需要创建一个关于成绩的数据表，里面储存着学生表的主键（user_id）和课程表的主键（course_id）以及对应的成绩。

下面，我们运行`rails generate model grade` 建立成绩模型：

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

进入`db/migrate/20160921051153_create_grades.rb`，建立相应的字段：

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

 - 这里的`t.belongs_to: :course` 就等于  `t.integer :course_id`，记录了课程表中课程的id。如此表示的意图在于更加清晰地表达模型之间的从属关系：

 >  **一个课程拥有多个成绩，一个成绩只属于一个课程；一个学生拥有多个成绩，一个成绩只属于一个学生；学生和课程两个属性能唯一确定成绩**

5.分析到这里，我们是不会遗忘了什么，对，就是这个：**一个老师上多门课，每一门属于一个老师**。这里老师和课程是也是一对多的关系（为了简化，就不再设计为多对多的关联关系了），回顾我们在课程模型的数据迁移文件`db/migrate/20160920163906_create_courses.rb` 中写的一句代码`t.belongs_to :teacher`，就已经将用户（老师）的id作为外键储存在课程模型的数据表中，**目的就是要实现用户（老师）与课程模型的一对多关系，即在课程模型的数据表中查找某个用户（老师）的id，就能将取出与这个id相关的所有课程的数据**。


6.我们直接画出ER图来看各个模型之间的关系：

![ER图](http://img.blog.csdn.net/20160921140809884)

 - 理解这个图需要一定的数据库知识，更加详细的在 [Rails Active 关联](http://guides.ruby-china.org/association_basics.html)。

> **需要注意的是，上图中，用户模型和课程模型通过成绩模型关联起来了（因为可以根据user_id查询所有相关的course_id，而且根据course_id也可以反过来查询所有的user_id），也就实现了用户模型和课程模型多对多的关系。**

7.到这里，我们已经完成了模型数据字段的建立过程，下面我们要把这些字段之间的关联关系告诉Rails框架，然后Rails就能自动帮我们组织各模型之间的关联关系。首先打开`app/models/course.rb`，填入：

``` ruby
class Course < ActiveRecord::Base

  has_many :grades
  has_many :users, through: :grades

  belongs_to :teacher, class_name: "User"

  validates :name, :course_type, :course_time, :course_week,
            :class_room, :credit, :teaching_type, :exam_type, presence: true, length: {maximum: 50}

end
```

 - 这里，课程模型通过`has_many`方法描述与其他模型的**一对多**的关联关系。`through` 参数描述了：此一对多关系是通过成绩模型才关联到用户模型的。
 - `belongs_to` 方法描述了与其他模型的**多对一**的关联关系，`class_name` 指定了用户模型的类名，这样Rails才能正确找到关联的用户模型。
 - `validates` 方法指定了对模型下的字段的验证，这里一共验证了`:name, :course_type, :course_time, :course_week, :class_room, :credit, :teaching_type, :exam_type` 8个字段，验证要求是 `presence: true, length: {maximum: 50}`，表示各个字段要存在（不为空）而且最大长度不能超过50个字符。**验证会在每次保存课程数据的时候执行**


8.下一步，打开`app/models/grade.rb`，填入：

``` ruby
class Grade < ActiveRecord::Base
  belongs_to :course
  belongs_to :user
end
```

 - 指定了与课程模型和用户模型一对多的关系

9.接着打开`app/models/user.rb`，填入：

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

 - 用户模型里，注意：
	 - `has_many :courses, through: :grades`
	 - `has_many :teaching_courses, class_name: "Course", foreign_key: :teacher_id`
	 --这里，两句代码都指定了与课程模型的一对多关联关系，然而第一句并没有指定课程模型的名字和课程模型中的外键字段，但是Rails却能工作正常。这体现了Rails一个特性：**convention over configuration (约定大于配置)**，意思就是说只要开发人员按照一定的约定模式写代码，就可以省去很多编写xml文件来指定文件路径的麻烦。这里的`has_many :courses, through: :grades`，Rails在解释代码的时候就会默认去找有没有叫course.rb（复数变单数）的文件，而且还会去这个course模型下找有没有叫user_id的外键字段。但是，根据第二句代码中的`teaching_courses`，Rails无法正确找到`course.rb`文件（这就不是仅仅是复数变单数那么简答了！），所以我们需要手动指定它去找一个叫做Course类的模型和关联模型下的外键`teacher_id`
 - 后面定义的方法，例如`digest，new_token，user_remember，user_forget, user_authenticated?, downcase_email`都是在后面用户登录功能需要用到的标准功能，详细内容请进[传送门](https://www.railstutorial.org/book/advanced_login)


9.所有的模型和模型的数据迁移文件我们都建立好了，下面运行数据迁移`rake db:migrate`建立数据表，成功会看到以下信息：

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

## 结束语

到此，所有关于模型（model）部分已经完结了，模型部分主要是对数据进行操作，包括数据表的建立，数据表之间的关联关系。

有困难的同学请先看看

 - [Active Record 基础](http://guides.ruby-china.org/active_record_basics.html)
 - [Active Record 数据迁移](http://guides.ruby-china.org/active_record_migrations.html)
 - [Active Record 数据验证](http://guides.ruby-china.org/active_record_validations.html)
 - [Active Record 关联](http://guides.ruby-china.org/association_basics.html)

下一章将会讲解控制器（controller）部分。

