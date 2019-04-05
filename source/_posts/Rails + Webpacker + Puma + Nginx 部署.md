---
title: Rails + Webpacker + Puma + Nginx 部署
date: 2019-01-01 12:58:12
categories: Rails
tags:
    - Puma
    - Nginx
    - Webpacker
---
# 准备

## ssh 登录

首先 ssh 登录服务器，免密码登录可以参考 [ssh 免密码登录服务器](https://www.cnblogs.com/luocaodan/p/10478614.html)

## 创建部署用户

```shell
$ sudo adduser deploy
```

## 安装依赖

### Ruby

这里使用 RVM 安装和管理 Ruby

```shell
$ gpg --keyserver hkp://pool.sks-keyservers.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
$ curl -sSL https://get.rvm.io | bash
```

等待安装完成

```shell
# 显示可用的 Ruby 版本
$ rvm list known
# 安装
$ rvm install 2.5.3
```

更换 Gem 源（使用 ruby-china 源）

```shell
$ gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
$ gem sources -l
$ bundle config mirror.https://rubygems.org https://gems.ruby-china.com
```

### Node

```shell
$ curl -sL https://deb.nodesource.com/setup_9.x | sudo -E bash -
$ sudo apt update
$ sudo apt install -y nodejs
$ node -v
$ npm -v
```

更换 npm 源

```shell
$ npm config set registry http://registry.npm.taobao.org/
$ npm config set disturl https://npm.taobao.org/dist
$ npm config set sass_binary_site https://npm.taobao.org/mirrors/node-sass
```

### Yarn

```shell
$ curl -sL https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
$ echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
$ sudo apt update && sudo apt-get install yarn
```

更换 Yarn 源

```shell
$ yarn config set registry http://registry.npm.taobao.org/
$ yarn config set disturl https://npm.taobao.org/dist
$ yarn config set sass_binary_site https://npm.taobao.org/mirrors/node-sass
```

# Nginx

## Nginx 安装

```shell
$ sudo apt install nginx
```

如果遇到类似的问题：

```shell
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
invoke-rc.d: initscript nginx, action "start" failed.
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: failed (Result: exit-code) since 三 2019-03-06 11:50:31 CST; 11ms ago
  Process: 32601 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=1/FAILURE)
  Process: 32596 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
```

可能是 80 端口已经被占用了，可以编辑 `/etc/nginx/sites-enabled/default` 更换端口

## Nginx 配置

编辑文件 `/etc/nginx/sites-enabled/default`

我们把 Nginx 作为反向代理，将收到的 Http 请求转发给 Rails 的 Puma 服务器，这里的 Puma 服务器就作为Nginx 的上游（upstream），Nginx 和 Puma 之间通过 Unix 套接字连接，所以首先配置 Nginx 的上游，假设用户为 deploy，项目名为 sample_app，项目位于 deploy 用户主目录

```shell
upstream app {
    server unix:/home/deploy/sample_app/shared/sockets/puma.sock;
}
```

然后配置 Nginx 服务器

```shell
server {
    listen 80;
    server_name 192.168.1.2; # 服务器域名或 IP

    # public 静态文件
    root /home/deploy/sample_app/public;

	# 找不到文件发给上游 Rails 应用
    try_files $uri/index.html $uri @app;

    location @app {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_pass http://app;
    }
	
	# 改为使用自定义的错误页面
    error_page 500 502 503 504 /500.html;
    error_page 404 /404.html;
    keepalive_timeout 10;
}
```

# PostgreSQL

## 安装

```shell
$ sudo apt update
$ sudo apt-get install postgresql postgresql-contrib libpq-dev
```

## 创建数据库账号

```shell
# 切换到 postgres 账户
$ sudo su postgres
# 创建数据库用户
$ createuser -s deploy
# 连接数据库 psql 客户端
$ psql
# 修改密码
postgres=# \password deploy
# 退出
postgres=# \q
```

### 远程连接配置

#### `pg_hba.conf`

```shell
$ sudo vi sudo vi /etc/postgresql/9.5/main/pg_hba.conf 
```

找到 `# IPv4 local connections:`添加：

```sh
host    all             all             10.0.0.1/8（根据需要配置）              md5
```

### `postgresql.conf`

```shell
$ sudo vi /etc/postgresql/9.5/main/postgresql.conf
```

找到 `#listen_addresses = 'localhost'`，改为

```shell
listen_addresses = '*'
```

## Rails 配置

### Gemfile

```ruby
gem 'pg'
```

### database.yml

```yaml
production:
  <<: *default
  adapter: postgresql
  encoding: unicode
  database: appname
  username: deploy
  password: <%= ENV['APPNAME_DATABASE_PASSWORD'] %>
```

# Puma

## Puma 配置

编辑项目 `config/puma.rb`

添加：

```ruby
# workers 设置为 CPU 核心数
workers 1
threads 1, 6
# 后台运行
daemonize true
rails_env = ENV['RAILS_ENV'] || "production"
environment rails_env
app_dir = File.expand_path("../..", __FILE__)
shared_dir = "#{app_dir}/shared"
bind "unix://#{shared_dir}/sockets/puma.sock"
stdout_redirect "#{shared_dir}/log/puma.stdout.log", "#{shared_dir}/log/puma.stderr.log", true
pidfile "#{shared_dir}/pids/puma.pid"
state_path "#{shared_dir}/pids/puma.state"

on_worker_boot do
  require "active_record"
  ActiveRecord::Base.connection.disconnect! rescue ActiveRecord::ConnectionNotEstablished
  ActiveRecord::Base.establish_connection(YAML.load_file("#{app_dir}/config/database.yml")[rails_env])
end
```

新建 log，pids，sockets 文件夹

```shell
$ mkdir -p shared/log shared/pids shared/sockets
```

# 部署项目

## 上传项目到服务器

```shell
$ scp -r /path/to/project deploy@1.2.3.4:~/
# 也可以使用 Git
```

## 安装项目依赖

```shell
$ bundle install
```

## 安装 Webpacker

```shell
$ bundle exec rails webpacker:install
```

## 配置生产环境密钥

#### 生成密钥

```shell
$ RAILS_ENV=production rake secret
```

#### 设置环境变量

编辑 `~/.bashrc`，添加：

```shell
export SECRET_KEY_BASE='刚才生成的密钥'
```

使设置生效

```shell
$ source ~/.bashrc
```

## 编译 Assets

```shell
$ RAILS_ENV=production rake assets:precompile
```

### 创建数据库

```shell
$ bundle exec rake db:create
```

## 数据库迁移

```shell
$ RAILS_ENV=production rails db:migrate
```

## 启动服务器

```shell
$ puma
# 重启
$ pumactl restart
# 停止
$ pumactl stop
```
