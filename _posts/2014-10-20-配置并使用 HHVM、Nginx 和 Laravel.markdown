---
layout: post
title:  "配置并使用 HHVM、Nginx 和 Laravel"
date:   2014-11-20 11:22:46
categories: wuwenhan update
---

## 前提条件
此文章适用于 Ubuntu 12.04 LTS 和 14.04 LTS
在你的 Mac 上通过 Brew [在 Mac 上安装 nginx](http://learnaholic.me/2012/10/10/installing-nginx-in-mac-os-x-mountain-lion/) and 在 [Mac 上安装 hhvm](https://github.com/facebook/hhvm/wiki/Building-and-installing-HHVM-on-OSX-10.8)） 也可以安装。

关于如何在其他发行版或版本的服务器上安装 HHVM 可以看 [这里](http://docs.hhvm.com/manual/en/install.php)。

OK，我们开始吧！

## 安装基本组件
首先，我们先安装一些后续过程需要的基本组件：

<pre>
$ sudo apt-get update
$ sudo apt-get install -y unzip vim git-core curl wget build-essential python-software-properties
</pre>

## 安装Nginx

接下来我们安装 Nginx。由于 hhvm 包会检测系统中是否已经安装了 Nginx（或者 Apache），并对其配置做一些修改，所以我将先安装 Ngin。我们还需要添加 ppa:nginx/stable 仓库来获取最新版本的 Nginx，一般最新版本都会包含一些安全方面的更新。

<pre>
$ sudo add-apt-repository -y ppa:nginx/stable
$ sudo apt-get update
$ sudo apt-get install -y nginx
</pre>

## 安装HHVM
现在轮到 HHVM 了，我们将参照 [官方文档](https://github.com/facebook/hhvm/wiki/Prebuilt-Packages-on-Ubuntu-12.04) 来安装 HHVM，包括以 FashCGI 模式启动 HHVM。

在此之前，HHVM-Fastcgi 是一个单独的安装包。现在不是了。

### Ubuntu 12.04:
<pre>
$ sudo add-apt-repository -y ppa:mapnik/boost
$ wget -O - http://dl.hhvm.com/conf/hhvm.gpg.key | sudo apt-key add -
$ echo deb http://dl.hhvm.com/ubuntu precise main | sudo tee /etc/apt/sources.list.d/hhvm.list
$ sudo apt-get update
$ sudo apt-get install -y hhvm
</pre>

### Ubuntu 14.04:
<pre>
$ wget -O - http://dl.hhvm.com/conf/hhvm.gpg.key | sudo apt-key add -
$ echo deb http://dl.hhvm.com/ubuntu trusty main | sudo tee /etc/apt/sources.list.d/hhvm.list
$ sudo apt-get update
$ sudo apt-get install -y hhvm
</pre>

## 配置 HHVM
HHVM 安装完成之后，你会看到输出的一些重要信息，注意看一下：
<pre>
********************************************************************
* HHVM is installed. Here are some more things you might want to do:
* 
* Configure your webserver to use HHVM:
* $ sudo /usr/share/hhvm/install_fastcgi.sh
* $ sudo /etc/init.d/nginx restart
* $ sudo /etc/init.d/apache restart
* $ sudo /etc/init.d/hhvm restart
* 
* Run command line scripts with HHVM:
* $ hhvm whatever.php
* 
* Use HHVM for /usr/bin/php even if you have php-cli installed:
* $ sudo /usr/bin/update-alternatives --install /usr/bin/php php /usr/bin/hhvm 60
********************************************************************
</pre>

HHVM 自带了一个脚本来帮我们安装并配置 FastCGI！我们来运行这个脚本：
<pre>
$ sudo /usr/share/hhvm/install_fastcgi.sh # This restarts Nginx for us

# Set this to start on system bootup
$ sudo update-rc.d hhvm defaults

# Restart the service now
$ sudo service hhvm restart                      # We'll also restart HHVM
</pre>

## 运行 PHP

现在，HHVM 本质上就是一个完整的 PHP 代码的运行环境了，因此我们没有必要安装 PHP 了。

安装之后，你就可以像使用 PHP 一样使用 hhvm。比如，你可以像下面这样运行 PHP 源码文件：

<pre>
$ hhvm some_file.php
</pre>

由于某些组件(composer, phpunit) 默认还是会依赖 "php" (其实是 php-cli) ，我们需要找一种方式让他们运行 HHVM 而不是 PHP。回顾一下上面 HHVM 安装完之后输出的一些信息，那里面已经告诉你如何实现这个需求了：is

<pre>
$ sudo /usr/bin/update-alternatives --install /usr/bin/php php /usr/bin/hhvm 60
</pre>

现在就可以像往常一样运行 "php" 了！

<pre>
$ php -v
HipHop VM 3.0.0 (rel)  
Compiler: tags/HHVM-3.0.0-0-g59a8db46e4ebf5cfd205fadc12e27a9903fb7aae  
Repo schema: 48906efe08d29a403bbe13414f32ccd256708e0b 
</pre>

## Fast-CGI
HHVM 其实已经帮我们配置好 Nginx 了。注意到 /etc/nginx/hhvm.conf 这个文件了吗。如果你以前使用过 PHP-FPM ， hhvm.conf 也是类似的内容：

<pre>
# Note this will work with "hack" files as well as php files!
location ~ \.(hh|php)$ {  
    fastcgi_keep_conn on;
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include        fastcgi_params;
}
</pre>

打开 Nginx 自带的 /etc/nginx/sites-available/default 文件，你将看到这里面已经将 hhvm.conf 文件引入了：

<pre>
... Stuff above this omitted ...
    # Make site accessible from http://localhost/
    server_name localhost;
    include hhvm.conf;  # 就是这里!!!

    location / {
... Stuff below this omitted ...
</pre>

HHVM 的开发者很 Nice，帮我们做了很多工作，让 HHVM 的使用异常傻瓜化！

## 跑一把 PHP 代码
接下来我们就来玩一把吧！注意 Nginx 默认的文档根目录是 root/usr/share/nginx/html，这里包含了一个 index.html 文件，如果我们访问本地服务器，将看到如下文件内容

<pre>
vagrant@vaprobash:~$ curl localhost  
Welcome to nginx!  
</pre>

下面我们就来创建一个 index.php 文件测试一下吧。下面这一样代码就是 index.php 文件中的所有内容，也就是仅仅调用了 phpinfo() 函数：

<pre>
$ echo "<?php phpinfo();" | sudo tee  /usr/share/nginx/html/index.php;
</pre>
试着运行一下：

<pre>
vagrant@vaprobash:~$ curl localhost/index.php  
HipHop
</pre>  
太棒了！phpinfo() 函数在 HHVM 环境中输出了 "HipHop"，我们知道这样就运行成功了！

## Laravel
通过上面的设置，安装 Laravel 其实就和平时的做法一样。由于我们为 HHVM 建了一个软连接到 php 命令，所以只要像平时一样执行其他安装任务即可。

## 配置 Nginx
我们来为 Laravel 快速创建一个 Nginx 配置。我们假定 Laravel 项目在 /vagrant 目录下面（网站的根目录就要设置为 /vagrant/laravel/public 了）。现在创建 /etc/nginx/sites-available/laravel 文件如下：

<pre>
server {  
    listen 80 default_server;

    root /vagrant/laravel/public;
    index index.html index.htm index.php;

    server_name localhost;

    access_log /var/log/nginx/localhost.laravel-access.log;
    error_log  /var/log/nginx/locahost.laravel-error.log error;

    charset utf-8;

    location / {
        try_files \$uri \$uri/ /index.php?\$query_string;
    }

    location = /favicon.ico { log_not_found off; access_log off; }
    location = /robots.txt  { log_not_found off; access_log off; }

    error_page 404 /index.php;      

    include hhvm.conf;  # The HHVM Magic Here

    # Deny .htaccess file access
    location ~ /\.ht {
        deny all;
    }
}
</pre>

此配置将替代 Nginx 自带的 default 配置文件。为了我们的讲解方面，我们将删除 default 文件的软连接，使其不起作用，并且启用刚才创建的 laravel 配置文件：
<pre>
$ sudo rm /etc/nginx/sites-enabled/default   # Remove the sym-linked default config
$ sudo ln -s /etc/nginx/sites-available/laravel /etc/nginx/sites-enabled/laravel    # Create a sym-link to the laravel config
</pre>

然后让 Nginx 重新加载配置文件：
<pre>
$ sudo service nginx reload
</pre>

安装 Composer
按照平时的方式在全局环境中安装 composer 即可：
<pre>
$ curl -sS https://getcomposer.org/installer | php
$ sudo mv composer.phar /usr/local/bin/composer
</pre>

## 安装 Laravel
然后我们就可以利用 Composer 来创建一个新的 Laravel 项目了。我们就把这个新项目叫做 laravel 吧，那么, 根据上面的目录安排，我们执行如下指令：

<pre>
# Move to /vagrant so we install Laravel
# into /vagrant/laravel
$ cd /vagrant
$ composer create-project laravel/laravel laravel
</pre>

稍等一会儿，Composer 将自动安装 Laravel 所依赖的扩展包。接下来就可以在浏览器中看到 Laravel 的启动画面了！

我觉得 [Github Wiki](https://github.com/facebook/hhvm/wiki/_pages) 和 [HHVM](http://docs.hhvm.com/manual/en/index.php) 官网 上的信息会随着项目的推进而快速更新。幸好关于错误报告和设置方面的文档也将逐步完善。不过目前的文档要么就是不全，要么就是太旧了。如果你找到更好的文档，别忘了通知我一声！














