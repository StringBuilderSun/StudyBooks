# Docker学习笔记

## 一、docker的安装

要求linux环境为CentOS7

清除以前的docker安装(如果以前安装过需要执行下)

```
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

安装依赖包

```shell
yum install -y yum-utils device-mapper-persistent
```

添加Docker软件包源

```shell
yum-config-manager --add-repo  https://download.docker.com/linux/centos/docker-ce.repo
```

更新yum包索引

```shell
yum makecache fast
```

安装Docker CE

```shell
yum install docker-ce
```

启动

```shell
systemctl start docker
```

测试

```shell
docker run hello-world
docker version
```

卸载

```shell
yum remove docker-ce
rm -rf /var/lib/docker 
```



## 二、镜像

​      简单的说，docker是一个不包含linux内核而又精简的linux操作系统。

### 镜像工作原理

当我们启动一个新的容器时， Docker会加载只读镜像，并在其之上添加一个读写层，如果运行中的容器修改一个已经存在的文件，那么会将该文件从下面的只读层复制到读写层，只读层的这个文件就会覆盖，但还存在，这就实现了文件系统隔离，当删除容器后，读写层的数据将会删除，只读镜像不变。

### 镜像文件存储结构

docker相关文件存放在： /var/lib/docker目录下，从只读层复制到最上层可读写层的文件系统数据在建立镜像时，每次写操作，都被视作一种增量操作，即在原有的数据层上添加一个新层；所以一个镜像会有若干个层组成。每次commit提交就会对产生一个ID，就相当于在上一层有加了一层，可以通过这个ID对镜像回滚 

### 镜像基本命令

docker基础命令详细的使用方式请参见：https://docs.docker.com/engine/reference/commandline/search/

#### search

从docker hub上搜索镜像

命令格式

```powershell
docker search [OPTIONS] TERM
```

常用参数：

--limit：最多显示的条数

--stars , -s：仅仅展示strars>n的结果	

--filter , -f: 在后面添加过滤条件

--format：以自定义的格式显示 比如  "{{.Name}}: {{.StarCount}}" 镜像将这样显示:nginx: 5441

```shell
#从docker hub搜索镜像busybox
docker search busybox
#最多显示三条
docker search --limit 3 busybox
#只显示stars>3的镜像 并且输出镜像的完整描述
docker search --stars=3 --no-trunc busybox
#用filter过滤
docker search --filter stars=3 busybox
```

这个命令如果没有特殊需求，直接使用docker search  Image 就可以了。

#### pull

从docker hub上拉取镜像到本地，查看本地所有镜像的方法是docker images 或者docker image ls -a

命令格式

```powershell
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
```

常用参数

--all-tags , -a ：拉取所有镜像

```powershell
#后面直接跟镜像 会拉取最新的镜像 一般为latest
docker pull debian
#镜像后跟tag 可以拉取指定layer的镜像
docker pull debian:jessie
#镜像跟版本 可以拉取指定版本的进行
docker pull ubuntu:14.04
#拉取指定仓库的镜像 myregistry.local:5000是指定仓库的url
docker pull myregistry.local:5000/testing/test-image
#拉取fedora的所有的镜像
docker pull --all-tags fedora
```

#### push

推送镜像到远程仓库，如果需要推送一个新的镜像到仓库，先要找到一个容器id，根据容器id通过commit提交一个镜像，然后通过docker tag 标记一个镜像，最终通过docker push 推送到远程仓库去。

命令格式

```
docker push [OPTIONS] NAME[:TAG]
```

常用参数

--disable-content-trust：跳过镜像签名

推送到默认docker hub，推送前需要先要docker login 登录

```powershell
#根据容器id创建一个镜像
docker commit      tomcat_test
#tag本地镜像这里默认是用了docker hub仓库，需要先login，其中stringbuildersun是用户名
docker tag tomcat_test stringbuildersun/tomcat_test_01
#将前两步生成的镜像上传到远端。这里默认是docker hub，stringbuildersun是用户名
docker push stringbuildersun/tomcat_test_01
```

推送到自定义远程仓库，推送前需要先docker login 登录

```powershell
#根据容器id创建一个镜像
docker commit      tomcat_test
#上传到自定义仓库 其中registry-host:5000是仓库url  username是用户名
docker tag tomcat_test registry-host:5000/username/tomcat_test_01
#将前两步生成的镜像上传到远端， registry-host:5000是仓库urlusername使用户名
docker push registry-host:5000/username/rhel-httpd
```

#### images

列出本机中的所有镜像

命令格式

```
docker images [OPTIONS] [REPOSITORY[:TAG]]
```

常用参数

--all , -a：显示所有镜像

--digests：显示镜像摘要，类似于git的commitId

--filter , -f：基于查询出的所有镜像过滤符合条件的镜像,看官方文档更好

--format：让镜像列表按照某种格式显示 看官方文档更好

--quiet , -q：仅仅显示镜像的image id,在批量处理镜像的时候很有用

```powershell
#列出本机所有镜像
docker images
#列出本机所有java镜像
docker images java
#列出备本机所有java tag=8的镜像
docker images java:8
```

#### commit

创建一个基于容器改变后的镜像,创建的新镜像可以通过Dockerfile命令改变镜像信息

命令格式

```powershell
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
```

常用参数

--author , -a：添加作者

--change , -c：通过Dockerfile指令创建一个镜像

--message , -m:添加提交信息

--pause , -p true：在容器进行commit操作时候 暂停容器

```powershell
#启动一个ubuntu的容器
docker run -itd  --name ubuntu_base ubuntu
#获取ubuntu容器的id：62c263440451
docker container ls  -q
#查看当前容器镜像的环境信息
docker inspect -f "{{ .Config.Env }}" 62c263440451
#使用--change改变镜像环境信息，并创建一个新镜像 ID：e70228ba285，此时不会影响老镜像
docker commit --change "ENV DEBUG true" 62c263440451 stringbuildersun/unbuntu:commit_1
#查看新镜像的环境信息 已经改变
docker inspect -f "{{ .Config.Env }}" e70228ba285
```

#### build

根据Dockerfile构建一个镜像

#### export

#### import

#### save

#### load

### 搭建LNMP网站

1、拉取mysql镜像

```
docker pull mysql
```

2、拉取nginx-php-fpm镜像

```powershell
#nginx和php组成的镜像
docker pull richarvey/nginx-php-fpm 
```

3、启动mysql镜像

```powershell
#将密码作为参数传入容器，并映射外部端口3308为mysql端口 并设置字符编码
docker run -itd --name lnmp_mysql  -p  3308:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql  --character-set-server=utf8
#登录数据库 显示所有的数据库
docker exec lnmp_mysql sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "show databases"'
#创建一个wp数据库用来作为wordpress的数据库
docker exec lnmp_mysql sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "create database wp"' 
#docker中的mysql需要修改规则 使得远程主机也可以访问到mysql
#进入mysql容器
docker exec -it lnmp_mysql bash
mysql -uroot -p
#更改规则 让远程主机可以通过root用户访问到mysql
alter user 'root'@'%' identified with mysql_native_password by 'password';
#刷新权限
flush privileges;
```

4、启动nginx-php-fpm镜像

```powershell
#连接lnmp_mysql以便lnmp_web访问数据库， -v 将容器的/var/www/html 挂载搭配主机的workspace的目录下
docker run -itd --name lnmp_web --link lnmp_mysql:db -p 88:80 -v ~/workspace:/var/www/html richarvey/nginx-php-fpm
```

5、下载wordpress放入到共享目录

```powershell
 #下载wordpress
 wget https://cn.wordpress.org/wordpress-4.7.4-zh_CN.tar.gz
 #解压
  tar -svf wordpress-4.7.4-zh_CN.tar.gz 
 #移动到工作目录workspace，因为映射了容器的/var/www/html，所以nginx可以展示这些内容、
 mv  wordpress/* 。/workspace
 #访问站点
 http://ip:88
```

6、workpress填写信息

```shell
数据库名：wp
用户名：root
密码：******
数据库主机：db
```

连接成功就可以使用wordpress创建站点了。