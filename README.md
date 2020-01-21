在上一篇 [Docker + node(koa) + nginx + mysql 开发环境搭建](https://github.com/tonysoul/docker-koa)，我们进行了本地开发环境搭建

现在我们就来开始线上环境部署

如果本地环境搭建没有什么问题，那么线上部署的配置也就很简单了

> 我所使用的环境，Linux Mint，命令有不同可以适当更改

## 目录结构
```
- compose   新建，线上环境配置
- data      
- conf      
- node_modules
- static        
- docker-compose.yml
- docker-compose-prod.yml   新建，线上环境配置
- package.json
- server.js
- yarn.lock
```

## 线上服务配置
我们现在需要3个服务：
- Node
- Nginx
- Mysql

在根目录下compose文件夹内，创建对应的Dockerfile配置文件，mysql是使用的镜像文件，就不用创建Dockerfile
```
compose/node/Dockerfile
compose/nginx/Dockerfile
```

## 1.Node服务配置
`compose/node/Dockerfile`
```
FROM node:12-alpine     # 使用的基础镜像文件
WORKDIR /code           # 工作目录的路径
COPY ./package.json ./server.js /code/  # 拷贝文件到/code
COPY ./static /code/static              # 拷贝文件到/code/static
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories \
	&& apk update && apk add yarn \
	&& yarn config set registry https://registry.npm.taobao.org/ \
	&& yarn             # 这一窜命令，设置镜像源到国内（加速），更新，安装yarn，yarn设置国内镜像源，yarn安装node依赖
CMD ["npm", "start"]    # 容器启动时运行
```

- `FROM`指定基础镜像文件
- `WORKDIR`工作目录的路径
- `COPY`拷贝本地文件到容器内
- `RUN`在 docker build 时运行命令
- `CMD`在 docker run 时运行命令



关于Dockerfile更详细的介绍可以查看 [这里](https://www.runoob.com/docker/docker-dockerfile.html)


## 2.Nginx服务配置
`compose/nginx/Dockerfile`

这个配置文件很简单，我们只需要把本地的nginx conf文件拷贝到容器里，删除默认配置
```
FROM nginx:1.17
RUN rm /etc/nginx/conf.d/default.conf
COPY ./conf/default.conf /etc/nginx/conf.d/default.conf 
```

### nginx conf域名设置
线上部署是需要一个域名的，我们现在假设一个域名测试：

`conf/default.conf`
```
server {
    listen       80;
    server_name  localhost yoursite.com;    # 这里添加你们的域名
    ...
}
```
我们在本地修改/etc/hosts，就可以测试了
```
127.0.1.1       yoursite.com
```


## 3.compose
和本地环境配置一样，要管理多个服务，我们需要compose文件来管理

在之前我们已经说明了，mysql是直接使用镜像，没有额外的修改，就不需要Dockerfile了

`docker-compose-prod.yml`根目录新建配置文件

```
version: "3"

volumes:
        static:         # 数据卷名称
        db:             # 数据卷名称

services:
        web:
                build:                                          # 构建镜像
                        context: ./                             # 上下文环境
                        dockerfile: ./compose/node/Dockerfile   # Dockerfile路径
                ports:
                        - "3000:3000"
                volumes:
                        - static:/code/static       # 使用前面声明的static挂在容器/code/static
                restart: always                     # 总是重启
        nginx:
                build:
                        context: ./
                        dockerfile: ./compose/nginx/Dockerfile
                ports:
                        - "80:80"
                volumes:
                        - static:/code/static
                restart: always
        mysql:
                image: mysql:5.6        # 直接使用镜像构建
                env_file: .env          # 环境变量env，我们写入到了配置文件里，避免密码泄漏
                volumes:    
                        - db:/var/lib/mysql
                ports:
                        - "3306:3306"
                restart: always

```
和之前的配置基本一样，不过有一些小的改动

`volumes`数据卷，我们设置了2个数据卷，static和db，下面就可以直接使用，它会自动创建并挂在到系统制定的目录/var/lib/docker/volumes/xxxx_static

- `build`我们之前是使用的image镜像构建，同样build也可以，因为我们在Dockerfile里面已经声明了镜像了
    - `context`指定上下文运行环境到根目录，我们build时需要一个上下文环境
    - `dockerfile`指定当前服务的Dockerfile配置

`restart`总是重启


## 测试服务
配置已经写好，我们来测试服务是否正常
```
sudo docker-compose -f docker-compose-prod.yml build
sudo docker-compose -f docker-compose-prod.yml up
```
服务运行起来，如果你能通过`yoursite.com`访问的话，说明配置成功


## 正式上线
我们只需要把源代码copy到服务器上然后运行上面两条命令即可

至于如何上传到服务器，FTP，GIT等...都可以



## 总结
在服务器已经安装好docker的情况下，和我们本地跑服务没有任何区别，各种顺滑，无障碍

从此以后就不再纠结线上线下环境不一致的问题了

> 珍爱生命，LOVE & PEACE

## 附录

### 服务器上安装Docker
服务器上也需要安装Docker，docker-compose，具体安装步骤有详细的官方文档：
- [Docker安装](https://docs.docker.com/install/)
- [Compose安装](https://docs.docker.com/compose/install/)

### Dockerfile
- [Dockerfile](https://www.runoob.com/docker/docker-dockerfile.html)
