---
date: 2016-03-19
title: 自建Oauth2协议服务器(Doorkeeper Gem)
categories: Web_development
tags: [rails, oauth] 
---

自建一个Oauth2的服务器（provider），使用了<a href="https://github.com/doorkeeper-gem/doorkeeper" style="color:cornflowerblue">doorkeeper gem</a>

最终效果: 开源项目<a href="https://github.com/PENGZhaoqing/CampusPortal" style="color:cornflowerblue">CampusPortal</a>

## 安装：

1 在Gemfile中添加doorkeeper gem

``` ruby
  gem 'doorkeeper'
```

2 run
 
``` ruby
bundle install
rails generate doorkeeper:install
rails generate doorkeeper:migration
rake db:migrate
```

## 配置：

能翻墙的可以上看[http://dev.mikamai.com/post/110722727899/oauth2-on-rails](http://dev.mikamai.com/post/110722727899/oauth2-on-rails)
 
## 手动测试:

不需要配置client的应用，直接用rails console测试搭建的provider，如果能看到以下结果说明配置成功

``` ruby
PENG-MacBook-Pro:oauth-client PENG-mac$ irb -r oauth2
2.2.4 :012 > callback= "http://localhost:3000/auth/doorkeeper/callback"
 => "http://localhost:3000/auth/doorkeeper/callback" 
2.2.4 :013 > secret = "f34da17c18e9017be0dff73307adbaae3a9e347d4384f7089454211612f8637c"
 => "f34da17c18e9017be0dff73307adbaae3a9e347d4384f7089454211612f8637c" 
2.2.4 :014 > app_id = "72b420c3dd55417b5e7170df6736c53abf8f8058dd1aebc17d463084c5f66dd8"
 => "72b420c3dd55417b5e7170df6736c53abf8f8058dd1aebc17d463084c5f66dd8" 
2.2.4 :015 >  client = OAuth2::Client.new(app_id, secret, site: "http://localhost:3001")
 => #<OAuth2::Client:0x007fa1609d3bf0 @id="72b420c3dd55417b5e7170df6736c53abf8f8058dd1aebc17d463084c5f66dd8", @secret="f34da17c18e9017be0dff73307adbaae3a9e347d4384f7089454211612f8637c", @site="http://localhost:3001", @options={:authorize_url=>"/oauth/authorize", :token_url=>"/oauth/token", :token_method=>:post, :connection_opts=>{}, :connection_build=>nil, :max_redirects=>5, :raise_errors=>true}> 
2.2.4 :016 > client.auth_code.authorize_url(redirect_uri: callback)
 => "http://localhost:3001/oauth/authorize?client_id=72b420c3dd55417b5e7170df6736c53abf8f8058dd1aebc17d463084c5f66dd8&redirect_uri=http%3A%2F%2Flocalhost%3A3000%2Fauth%2Fdoorkeeper%2Fcallback&response_type=code" 
2.2.4 :017 > access_grant_token = "1dc474acad92216f76c235155a1f334e95b2034f7d68bd75e2c4cc3f51b6338b"
 => "1dc474acad92216f76c235155a1f334e95b2034f7d68bd75e2c4cc3f51b6338b" 
2.2.4 :018 > access = client.auth_code.get_token(access_grant_token, redirect_uri: callback)
 => #<OAuth2::AccessToken:0x007fa1613a7208 @client=#<OAuth2::Client:0x007fa1609d3bf0 @id="72b420c3dd55417b5e7170df6736c53abf8f8058dd1aebc17d463084c5f66dd8", @secret="f34da17c18e9017be0dff73307adbaae3a9e347d4384f7089454211612f8637c", @site="http://localhost:3001", @options={:authorize_url=>"/oauth/authorize", :token_url=>"/oauth/token", :token_method=>:post, :connection_opts=>{}, :connection_build=>nil, :max_redirects=>5, :raise_errors=>true}, @auth_code=#<OAuth2::Strategy::AuthCode:0x007fa1609c9f38 @client=#<OAuth2::Client:0x007fa1609d3bf0 ...>>, @connection=#<Faraday::Connection:0x007fa1609c9dd0 @parallel_manager=nil, @headers={"User-Agent"=>"Faraday v0.9.2"}, @params={}, @options=#<Faraday::RequestOptions (empty)>, @ssl=#<Faraday::SSLOptions (empty)>, @default_parallel_manager=nil, @builder=#<Faraday::RackBuilder:0x007fa1609c9a38 @handlers=[Faraday::Request::UrlEncoded, Faraday::Adapter::NetHttp], @app=#<Faraday::Request::UrlEncoded:0x007fa1609a8810 @app=#<Faraday::Adapter::NetHttp:0x007fa1609a8888 @app=#<Proc:0x007fa1609a8978@/Users/PENG-mac/.rvm/gems/ruby-2.2.4/gems/faraday-0.9.2/lib/faraday/rack_builder.rb:152 (lambda)>>>>, @url_prefix=#<URI::HTTP http://localhost:3001/>, @proxy=nil>>, @token="e61459fcbba7ff68d7b6e5fa3cd7e9cafb40a8a1d25bfbbe1f45d7800ccfbad1", @refresh_token=nil, @expires_in=7200, @expires_at=1458362045, @options={:mode=>:header, :header_format=>"Bearer %s", :param_name=>"access_token"}, @params={"token_type"=>"bearer", "created_at"=>1458354845}> 
2.2.4 :019 >  access.get("/me").parsed
 => {"id"=>1, "email"=>"peng@qq.com", "created_at"=>"2016-03-18T13:45:20.560Z", "updated_at"=>"2016-03-19T02:24:35.164Z"} 

```

## Console(控制台输出) 

1.client(application side)

```ruby
Started GET "/" for ::1 at 2016-04-25 02:22:07 +0800
  ActiveRecord::SchemaMigration Load (15.5ms)  SELECT "schema_migrations".* FROM "schema_migrations"


Started GET "/auth/doorkeeper" for ::1 at 2016-04-25 02:22:07 +0800
I, [2016-04-25T02:22:07.794163 #9810]  INFO -- omniauth: (doorkeeper) Request phase initiated.


Started GET "/auth/doorkeeper/callback?code=6324f49b09e982f36f087f6bfa14559347aab5740f33802f3260b1daca693acd&state=92816b4dd097eeeaf1340b59329c5a9c8ab3a38bd251dc33" for ::1 at 2016-04-25 02:22:08 +0800
I, [2016-04-25T02:22:08.006914 #9810]  INFO -- omniauth: (doorkeeper) Callback phase initiated.
Processing by SessionsController#create as HTML
  Parameters: {"code"=>"6324f49b09e982f36f087f6bfa14559347aab5740f33802f3260b1daca693acd", "state"=>"92816b4dd097eeeaf1340b59329c5a9c8ab3a38bd251dc33", "provider"=>"doorkeeper"}
  User Load (12.3ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = ? LIMIT 1  [["id", 3]]
Redirected to http://localhost:3000/page1/view1
Completed 302 Found in 76ms (ActiveRecord: 13.2ms)
```


2.Server(Provider)

```ruby
Started GET "/oauth/authorize?client_id=60ff3577c8a8b0b1e68cb814d5a1775bbc93502342f54f698e44f8b64b1d9079&redirect_uri=http%3A%2F%2Flocalhost%3A3000%2Fauth%2Fdoorkeeper%2Fcallback&response_type=code&state=92816b4dd097eeeaf1340b59329c5a9c8ab3a38bd251dc33" for ::1 at 2016-04-25 02:22:07 +0800
Processing by Doorkeeper::AuthorizationsController#new as HTML
  Parameters: {"client_id"=>"60ff3577c8a8b0b1e68cb814d5a1775bbc93502342f54f698e44f8b64b1d9079", "redirect_uri"=>"http://localhost:3000/auth/doorkeeper/callback", "response_type"=>"code", "state"=>"92816b4dd097eeeaf1340b59329c5a9c8ab3a38bd251dc33"}
  User Load (0.1ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = ? LIMIT 1  [["id", 1]]
  Doorkeeper::Application Load (11.2ms)  SELECT  "oauth_applications".* FROM "oauth_applications" WHERE "oauth_applications"."uid" = ? LIMIT 1  [["uid", "60ff3577c8a8b0b1e68cb814d5a1775bbc93502342f54f698e44f8b64b1d9079"]]
  CACHE (0.0ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = ? LIMIT 1  [["id", 1]]
  CACHE (0.0ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = ? LIMIT 1  [["id", 1]]
  Doorkeeper::AccessToken Load (1.2ms)  SELECT  "oauth_access_tokens".* FROM "oauth_access_tokens" WHERE "oauth_access_tokens"."application_id" = ? AND "oauth_access_tokens"."resource_owner_id" = ? AND "oauth_access_tokens"."revoked_at" IS NULL  ORDER BY created_at desc LIMIT 1  [["application_id", 1], ["resource_owner_id", 1]]
  CACHE (0.0ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = ? LIMIT 1  [["id", 1]]
   (0.1ms)  begin transaction
  Doorkeeper::AccessGrant Exists (0.7ms)  SELECT  1 AS one FROM "oauth_access_grants" WHERE "oauth_access_grants"."token" = '6324f49b09e982f36f087f6bfa14559347aab5740f33802f3260b1daca693acd' LIMIT 1
  SQL (1.2ms)  INSERT INTO "oauth_access_grants" ("application_id", "resource_owner_id", "expires_in", "redirect_uri", "scopes", "token", "created_at") VALUES (?, ?, ?, ?, ?, ?, ?)  [["application_id", 1], ["resource_owner_id", 1], ["expires_in", 600], ["redirect_uri", "http://localhost:3000/auth/doorkeeper/callback"], ["scopes", ""], ["token", "6324f49b09e982f36f087f6bfa14559347aab5740f33802f3260b1daca693acd"], ["created_at", "2016-04-24 18:22:07.992402"]]
   (1.7ms)  commit transaction
Redirected to http://localhost:3000/auth/doorkeeper/callback?code=6324f49b09e982f36f087f6bfa14559347aab5740f33802f3260b1daca693acd&state=92816b4dd097eeeaf1340b59329c5a9c8ab3a38bd251dc33
Completed 302 Found in 86ms (ActiveRecord: 17.3ms)


Started POST "/oauth/token" for ::1 at 2016-04-25 02:22:08 +0800
Processing by Doorkeeper::TokensController#create as */*
  Parameters: {"client_id"=>"60ff3577c8a8b0b1e68cb814d5a1775bbc93502342f54f698e44f8b64b1d9079", "client_secret"=>"[FILTERED]", "code"=>"[FILTERED]", "grant_type"=>"authorization_code", "redirect_uri"=>"http://localhost:3000/auth/doorkeeper/callback"}
  Doorkeeper::AccessGrant Load (0.2ms)  SELECT  "oauth_access_grants".* FROM "oauth_access_grants" WHERE "oauth_access_grants"."token" = ? LIMIT 1  [["token", "6324f49b09e982f36f087f6bfa14559347aab5740f33802f3260b1daca693acd"]]
  Doorkeeper::Application Load (0.2ms)  SELECT  "oauth_applications".* FROM "oauth_applications" WHERE "oauth_applications"."uid" = ? AND "oauth_applications"."secret" = ? LIMIT 1  [["uid", "60ff3577c8a8b0b1e68cb814d5a1775bbc93502342f54f698e44f8b64b1d9079"], ["secret", "3729d6966fa325e0fc43c33110f9f56d0d737b2e8ae7837e13a7aed7c853203c"]]
   (0.1ms)  begin transaction
  Doorkeeper::AccessGrant Load (0.1ms)  SELECT  "oauth_access_grants".* FROM "oauth_access_grants" WHERE "oauth_access_grants"."id" = ? LIMIT 1   [["id", 71]]
  SQL (0.3ms)  UPDATE "oauth_access_grants" SET "revoked_at" = ? WHERE "oauth_access_grants"."id" = ?  [["revoked_at", "2016-04-24 18:22:08.062961"], ["id", 71]]
  Doorkeeper::Application Load (0.2ms)  SELECT  "oauth_applications".* FROM "oauth_applications" WHERE "oauth_applications"."id" = ? LIMIT 1  [["id", 1]]
  CACHE (0.0ms)  SELECT  "oauth_applications".* FROM "oauth_applications" WHERE "oauth_applications"."id" = ? LIMIT 1  [["id", 1]]
  Doorkeeper::AccessToken Exists (15.3ms)  SELECT  1 AS one FROM "oauth_access_tokens" WHERE "oauth_access_tokens"."token" = '2f0b032027141db886d91f613921f35bed6d8c18f35e8da7dc3b6f38dfe8fea2' LIMIT 1
  SQL (0.7ms)  INSERT INTO "oauth_access_tokens" ("application_id", "resource_owner_id", "scopes", "expires_in", "token", "created_at") VALUES (?, ?, ?, ?, ?, ?)  [["application_id", 1], ["resource_owner_id", 1], ["scopes", ""], ["expires_in", 7200], ["token", "2f0b032027141db886d91f613921f35bed6d8c18f35e8da7dc3b6f38dfe8fea2"], ["created_at", "2016-04-24 18:22:08.107892"]]
   (1.9ms)  commit transaction
Completed 200 OK in 55ms


Started GET "/me" for ::1 at 2016-04-25 02:22:08 +0800
Processing by ApplicationController#me as */*
  Doorkeeper::AccessToken Load (0.2ms)  SELECT  "oauth_access_tokens".* FROM "oauth_access_tokens" WHERE "oauth_access_tokens"."token" = ? LIMIT 1  [["token", "2f0b032027141db886d91f613921f35bed6d8c18f35e8da7dc3b6f38dfe8fea2"]]
  User Load (0.1ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = ? LIMIT 1  [["id", 1]]
  Resource Load (15.2ms)  SELECT  "resources".* FROM "resources" WHERE "resources"."user_id" = ? LIMIT 1  [["user_id", 1]]
  Access Load (0.9ms)  SELECT  "accesses".* FROM "accesses" WHERE "accesses"."resource_id" = ? AND "accesses"."app_id" = ? LIMIT 1  [["resource_id", 1], ["app_id", 1]]
Completed 200 OK in 94ms (Views: 1.4ms | ActiveRecord: 17.0ms)


```

### 最终返回结果

```ruby
{"provider":"doorkeeper","uid":1,"info":{"name":"peng1","number":"9453739065","role":"undergraduate","department":"undergraduate","node":[1,1,1,4,6,9,10,2,2,1,2,8,9,10,3,1,4,1,3,4,6,8,8,1,4,6,8,10,9,1,2,3,4,6,8,10,1,2,3,5,6,10],"path":[1,11,111,114,116,119,1110,112,12,121,122,128,129,1210,13,131,14,141,143,144,146,148,18,181,184,186,188,1810,19,191,192,193,194,196,198,110,1101,1102,1103,1105,1106,11010]},"credentials":{"token":"cc4333385d6a66e72d22c61a4d0bfdfdc6942161b809946415cdf6703f607e3b","expires_at":1462166337,"expires":true},"extra":{"raw_info":{"user":{"id":1,"name":"peng1","email":"user1@test.com","number":"9453739065","role":"undergraduate","department":"Information Science","password_digest":"$2a$10$pGdyNjt5VPlqWW1irKph.enbIqK4YoNmKFVTR9eo108YF5cRF0dVq","remember_digest":null,"activation_digest":"$2a$10$8m2jiq2cpG4HkQVpntcvzu9yn9iscyZ4M/NwHzQOJLq6fWxBXOy46","activated":true,"activated_at":"2016-04-23T14:49:49.585Z","reset_digest":null,"reset_sent_at":null,"created_at":"2016-04-23T14:49:49.765Z","updated_at":"2016-04-23T14:49:49.765Z","icon":{"url":"/uploads/user/icon/1/default.png"}},"access":{"id":10,"node":[1,1,1,4,6,9,10,2,2,1,2,8,9,10,3,1,4,1,3,4,6,8,8,1,4,6,8,10,9,1,2,3,4,6,8,10,1,2,3,5,6,10],"path":[1,11,111,114,116,119,1110,112,12,121,122,128,129,1210,13,131,14,141,143,144,146,148,18,181,184,186,188,1810,19,191,192,193,194,196,198,110,1101,1102,1103,1105,1106,11010],"app_id":1,"resource_id":1,"created_at":"2016-04-23T14:49:57.408Z","updated_at":"2016-04-27T13:01:14.279Z"}}}}
```


## Debug refrence

[https://github.com/doorkeeper-gem/doorkeeper/issues/732](https://github.com/doorkeeper-gem/doorkeeper/issues/732)