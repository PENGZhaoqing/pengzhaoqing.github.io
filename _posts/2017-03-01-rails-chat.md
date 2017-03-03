---
date: 2017-03-01
title: [Rails应用实战]WebChat的敏捷开发
categories: 学习笔记
tags: [rails]
---

即时通讯的应用如微信WeChat大家一定很熟悉，那么今天就来详解一下如何用Rails快速开发一款网页版微信（网页版聊天室）

所有代码在[Github](https://github.com/PENGZhaoqing/RailsChat)中，目前已有功能：

* 即时通讯
* 增添好友
* 创建聊天
* 拉人，删人
* 转移房屋权限

### Todo

1. UI界面修改（类似WeChat）
2. 未读信息的提醒（包括声音）
3. 加入更多的ajax提高用户体验

## 代码实现

我们主要应用了Render_sync库来提供服务器监听并实时推送信息的功能，下面的实现贴出一些主要的部分，详细的请到Github中下载代码查看：


1. 首先加入render_sync等库，编辑Gemfile文件，加入以下

	``` ruby
	gem 'faye'
	gem 'thin', require: false
	gem 'render_sync'
	```
2. 创建User，Message，Chat，FriendShip四个模型，之间的关系分别为：


	``` ruby
	class Chat < ActiveRecord::Base
	  has_and_belongs_to_many :users
	  has_many :messages, :dependent => :destroy
	end
	```


	``` ruby
	class Friendship < ActiveRecord::Base
	  belongs_to :user
	  belongs_to :friend, :class_name => "User"
	end
	```


	``` ruby
	class Message < ActiveRecord::Base
	  belongs_to :user
	  belongs_to :chat
	
	  sync :all
	  sync_scope :by_chat, ->(chat) { where(chat_id: chat.id) }
	end
	```


	``` ruby
	class User < ActiveRecord::Base
	
	  has_many :messages
	  has_and_belongs_to_many :chats
	
	  has_many :friendships
	  has_many :friends, :through => :friendships
	  has_many :inverse_friendships, :class_name => "Friendship", :foreign_key => "friend_id"
	  has_many :inverse_friends, :through => :inverse_friendships, :source => :user
	  
	  .....
	  
	end
	```

3. 编写各个模型的migrate文件，这里不贴出来，详细请看[这里](https://github.com/PENGZhaoqing/RailsChat/tree/master/db/migrate)，创建seed.rb文件，写入初始数据：

	``` ruby
	(1..100).each do |index|
	  User.create(
	      name: Faker::Name.name,
	      email: "user#{index}@test.com",
	      password: 'password',
	      role: Faker::Number.between(1, 4),
	      sex: ['male', 'female'].sample,
	      phonenumber: Faker::PhoneNumber.phone_number,
	      status: Faker::Company.profession
	  )
	end
	
	User.first.friendships.create(:friend_id => 2)
	User.first.friendships.create(:friend_id => 3)
	```

4. 编写ChatsController，FriendshipsController, MessagesController, SessionsController以及UsersController，如[这里](https://github.com/PENGZhaoqing/RailsChat/tree/master/app/controllers)所示，以及各个控制器下的试图，如[这里]( https://github.com/PENGZhaoqing/RailsChat/tree/master/app/views)所示

5. 这里主要讲解Render_sync库的用法，在ChatsController的show方法的视图中，这两句起到了推送实时信息的功能：

	``` ruby
	<%= sync partial: 'message_row', collection: Message.by_chat(@chat), refetch: true %>
	<%= sync_new partial: 'message_row', resource: Message.new, scope: @chat, refetch: true %>
	```

 -  `refetch: true`选项能决定推送的方式为ajax
 -  `scope: @chat` 能决定推送消息的范围，例如只给当前聊天室页面的client实时推送新的消息
 -  `collection: Message.by_chat(@chat)` 能决定历史信息为当前聊天室的信息，而不是所有的所有的信息都在当前聊天室显示

	局部视图文件需要位于`app/views/sync/messages/refetch/_message_row.html.erb`才能生效，这个局部视图负责就是每条消息的显示，例如消息的主题，发送人和发送时间，简化后为：

	``` ruby
	<p> 
	<%= message.user.name %>
    <%= message.created_at.to_formatted_s(:db) %>
	<%= message.body %>
	</p>
	```

6. 最重要的为MessagesController的create方法，这个方法接受两个参数，一个是发送的message，一个是聊天室的id，创建发送的message后，将此message与当前聊天室关联，然后调用sync_new方法同步推送此message，推送的范围为此聊天室，最后重定向至此聊天室 

	``` ruby
	class MessagesController < ApplicationController
	  ...
	  
	  def create
	    @message = current_user.messages.build(message_params)
	    chat=Chat.find_by_id(params[:chat_room])
	    @message.chat=chat
	    if @message.save
	      sync_new @message, scope: chat
	    end
	    redirect_to chat_path(chat)
	  end
	  
	  ....
	  
	end
	```

7.  最后编辑聊天室中的发送框即可，在ChatsController的show方法视图下，加入：

	``` ruby
	<%= form_for @new_message, remote: true do |f| %>
	    <div class="input-group">
	      <%= f.text_field :body, class: "form-control input-sm", placeholder: "Type your message here..." %>
	      <span class="input-group-btn"> <%= f.submit "发送", class: "btn btn-warning btn-sm" %> </span>
	      <%= hidden_field_tag :chat_room, @chat.id %>
	    </div>
	<% end %>
	```
	 
	 - 使用hidden_field_tag传入此聊天室的id
	 - `remote: true` 使表单用ajax方法提交

8. 完成其他模块，运行服务器

## 结果截屏


![这里写图片描述](http://img.blog.csdn.net/20170301165213282?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHBwODMwMDg4NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


![这里写图片描述](http://img.blog.csdn.net/20170301165225423?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHBwODMwMDg4NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](http://img.blog.csdn.net/20170301165237472?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHBwODMwMDg4NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](http://img.blog.csdn.net/20170301165248845?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHBwODMwMDg4NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


## 如何使用

1. Fork项目，https://github.com/PENGZhaoqing/RailsChat

  ``` 
  git clone https://github.com/your_user_name/RailsChat
  cd RailsChat
  bundle install
  rails server
  ```

2. 然后再打开另外一个终端，运行以下命令启动另外一个server来监听聊天室的用户并实时推送最新的消息：

  ```
  rackup sync.ru -E production
  ```

### Note：如果要部署到云上或者本地局域网内，需要修改`config/sync.yml`文件

以本地局域网为例：

1. 若本机的ip地址为192.168.0.14（使用`ifconfig`查看），那么需要将config/sync.yml中的localhost全改为此ip地址，例如
 
 ``` ruby
  development:
    server: "http://192.168.0.14:9292/faye"
    adapter_javascript_url: "http://192.168.0.14:9292/faye/faye.js"
    auth_token:  "97c42058e1466902d5adfac0f83e84c1794b9c3390e3b0824be9db8344eae82b"
    adapter: "Faye"
    async: true
    
  test:
    ...
  production:
    ...
  ```

2. 然后运行rake tmp:clear来清除缓存，不然修改不会生效（运行前先将所有相关的运行停止：如rails s,rackup sync.ru等）

3. 再次运行rails服务器和监听程序，并指定监听程序运行的ip地址

  ```
  rails s
  rackup sync.ru -E production --host 192.168.0.14 
  ```



## Debug

1. 当遇到消息并没有实时推送的情况时，先F12查看浏览器的Js文件加载情况，若faye.js加载成功则一般不会出现问题

2. 以上加载完成但是仍然没有推送的时候，请查看Rails服务器的log文件

3. 需要在两个浏览器中登录不同的账号来检验聊天室功能
