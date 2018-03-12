---
title: 初识go
date: 2018-03-09 15:03:57
tags:
---
之前小小的调研了下使用golang的成本，当然是希望能够快速上手。    
一切从环境搭建开始，为了让大家能够快速搭建完成golang的环境，并进行web开发，所以我借助于docker来搭建环境，下面就分步骤说下是如何搭建的。    
1. 安装docker  自行google    
2. docker中安装golang, docker pull golang:latest，可以自己定义版本
3. docker中安装mysql, nginx镜像，命令同上
4. 定制自己的golang镜像，因为想借助于beego这个框架来搭建web应用，所以Dockerfile文件如下：
<pre><code>
FROM golang   
\# Install beego and the bee dev tool    
RUN go get github.com/astaxie/beego    
    && go get github.com/beego/bee    
    && go get github.com/go-sql-driver/mysql    
\# Expose the application on port 8080    
EXPOSE 8080    
\# Set the entry point of the container to the bee command that runs the application and watches for changes    
CMD ["bee", "run"]
</code> </pre>   
这个file主要做了三件事情，安装beego和mysql相关，暴露接口以及启动应用并监听改变。
然后在dockerfile所在目录运行    
	`docker build -t 'your-image-name' // 我取名叫supplier，下文有用到`   
至此，准备工作做完了，接下来我们借助于docker-compose来启动镜像对应的container，以及关联container。
步骤如下：    
	0. 新建一个网络环境，供接下来的容器使用    
	`docker network create -d bridge custom`    
	1. 新建mysql容器，并启动，因为beego镜像的构建文件中最后一步启动应用了，所以需要mysql先启动起来，下面是mysql对应的docker-compose.yml
<pre><code>
version: "3"
services:
    mysql:
        image: mysql:5.5
    environment:
        \- MYSQL_ROOT_PASSWORD=root
        \- MYSQL_DATABASE=supplier
    networks:
        \- custom
networks:
    custom:
        external: true
</code></pre>  
这里面使用了mysql5.5版本的镜像，通过environment设置了密码和使用的数据库，同时指定了使用的网络    
在所在目录运行`docker-compose up`即可    
	2. 建立nginx和golang的容器    
<pre><code>
version: "3"
services:
    web:
        image: nginx:stable
    volumes:
        \- /Users/didi/Documents/supplier/nginx:/etc/nginx/conf.d:ro
        \- /Users/didi/Documents/supplier/project:/etc/nginx/html
    ports:
        \- 8666:80
    command: /bin/bash -c "nginx -g 'daemon off;'"
    links:
        \- gosupplier
    networks:
        \- custom
    gosupplier:
        image: supplier:latest
        volumes:
          \- /Users/didi/Documents/supplier/project:/go/src/supplier
        ports:
          \- 8777:8080
        working_dir: /go/src/supplier
    external_links:
      \- mysql
    networks:
      \- custom
networks:
    custom:
        external: true
</code></pre>    
同样这个文件启动了nginx和golang的容器，相当于应用已经启动了，同时使用了外部容器mysql，就是我们刚刚启动的mysql容器，为了保证容器之间能够ping到，所以使用了同样的网络custom。

至此环境就算搭建完成了，我们可以通过localhost:8666 来访问，因为我把本机的8666端口映射到docker中nginx容器的80端口了。    
在项目文件中，我们可以借助于beego的官方文档 https://beego.me/docs/intro/ 来构建目录和使用，默认会有一个main.go作为入口文件，有一个main函数作为入口函数，这是目录。    
<pre><code>
├── conf
│   └── app.conf
├── controllers
│   ├── admin
│   └── default.go
├── main.go
├── models
│   └── models.go
├── static
│   ├── css
│   ├── ico
│   ├── img
│   └── js
└── views
    ├── admin
    └── index.tpl
</code></pre>    
beego有集成mysql数据库，可以借助于他的orm来进行连接，以及相关的CRUD操作，可以在init函数里面注册driver，和数据库，以及model。这里面的mysql就是刚刚依赖的容器名字，只有这样，才能保证能够找到。    
<pre><code>
func init() {
    orm.RegisterDriver("mysql", orm.DRMySQL)
    orm.RegisterDataBase("default", "mysql", "root:root@(mysql:3306)/supplier?charset=utf8")
    orm.RegisterModel(new(User))
}
</code></pre>   
我研究的有限，只是初步跑通，后面继续。