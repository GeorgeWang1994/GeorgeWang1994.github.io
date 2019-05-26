


<meta name="referrer" content="no-referrer" />

# Docker使用总结

下面是我自己总结的一些命令，如下：

![image](https://wx4.sinaimg.cn/large/007FyU7Tly1g35b37s8c1j31qc28onpe.jpg)



## 镜像使用

具体的命令对应的参数包括如下：

### docker search [OPTIONS] container

搜索镜像

```dockerfile
-f, --filter filter   Filter output based on conditions provided  # 指定需要进行搜索

--format string   Pretty-print search using a Go template  # 返回需要展示的列，可选项包括NAME、Description、StarCount、IsOfficial、IsAutomated

--limit int       Max number of search results (default 25)  # 限制搜索返回的结果个数

--no-trunc        Don't truncate output  # 是否需要截断描述的内容从而全部显示出来


eg: docker search -f star=3 -f is_automated=true django  # 搜索至少3颗星并且是由docker hub创建的官方镜像

eg: docker search --format "{{.NAME}}\t{{.Description}}\t{{.StarCount}}" django

eg: docker search --limit=15 django

eg: docker search --no-trunc=true django
```



### docker pull [OPTIONS] NAME[:TAG|@DIGEST]  

拉取镜像，通过标签或者简介其中一种都可以拉取

```dockerfile
-a, --all-tags                Download all tagged images in the repository # 下载关于镜像的所有相关的标签的镜像

--disable-content-trust   Skip image verification (default true)  # 跳过镜像认证

eg: docker pull -a django
```



### docker image [OPTIONS] IMAGE_ID

对镜像进行操作

```dock
build       Build an image from a Dockerfile
history     Show the history of an image
import      Import the contents from a tarball to create a filesystem image
inspect     Display detailed information on one or more images
load        Load an image from a tar archive or STDIN
ls          List images
prune       Remove unused images
pull        Pull an image or a repository from a registry
push        Push an image or a repository to a registry
rm          Remove one or more images
save        Save one or more images to a tar archive (streamed to STDOUT by default)
tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
```



### docker images [OPTIONS] [REPOSITORY[:TAG]]

获取镜像列表

```dockerfile
--all, -a :列出本地所有的镜像（含中间映像层，默认情况下，过滤掉中间映像层）；

--digests :显示镜像的摘要信息；

--filter, -f :显示满足条件的镜像；

--format :指定返回值的模板文件；

--no-trunc :显示完整的镜像信息；

--quiet, -q :只显示镜像ID。


eg: docker images --filter "dangling=true"  # 显示未打标签的镜像

eg: docker rmi $(docker images -f "dangling=true" -q)  # 删除未打标签的镜像

eg: docker images -f "before=image1"  # 显示在image1之前创建的镜像（不包括image1）

eg: docker images -f "since=image1"  # 显示在image1之后创建的镜像（不包括image1）

eg: docker images -f=reference="a*:*b"  # 显示 分支名由a开头，标签由b结尾的镜像
```



### docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

创建一个新的容器，并且执行一个命令，相当于create + start命令；

```dockerfile
-a stdin: 指定标准输入输出内容类型，可选 STDIN/STDOUT/STDERR 三项；

-d: 后台运行容器，并返回容器ID；

-i: 以交互模式运行容器，通常与 -t 同时使用；

-P: 随机端口映射，容器内部端口随机映射到主机的高端口

-p: 指定端口映射，格式为：主机(宿主)端口:容器端口

-t: 为容器重新分配一个伪输入终端，通常与 -i 同时使用；

--name="nginx-lb": 为容器指定一个名称；

--dns 8.8.8.8: 指定容器使用的DNS服务器，默认和宿主一致；

--dns-search example.com: 指定容器DNS搜索域名，默认和宿主一致；

-h "mars": 指定容器的hostname；

-e username="ritchie": 设置环境变量；

--env-file=[]: 从指定文件读入环境变量；

--cpuset="0-2" or --cpuset="0,1,2": 绑定容器到指定CPU运行；

-m :设置容器使用内存最大值；

--net="bridge": 指定容器的网络连接类型，支持 bridge/host/none/container: 四种类型；

--link=[]: 添加链接到另一个容器；

--expose=[]: 开放一个端口或一组端口；


eg: docker run -it nginx:latest /bin/bash
```



### docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]

提交镜像

```dockerfile
-a, --author string    Author (e.g., "John Hannibal Smith <hannibal@a-team.com>")
-c, --change list      Apply Dockerfile instruction to the created image
-m, --message string   Commit message
-p, --pause            Pause container during commit (default true)


eg: docker commit c3f279d17e0a  svendowideit/testimage:version3
```



### docker build [OPTIONS] PATH | URL | -

构建镜像

```dockerfile
命令非常多，可以自行查看；


eg: docker build -t honeypot/abchina:latest .
```



### docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

修改镜像的tag

```dockerfile
eg: docker tag httpd fedora/httpd:version1.0
```



### docker inspect [OPTIONS] NAME|ID [NAME|ID...]

获取镜像的详细信息

```dockerfile
-f, --format string   Format the output using the given Go template
-s, --size            Display total file sizes if the type is container
--type string     Return JSON for specified type


eg: docker inspect mysql:5.6

eg: docker inspect --format='{{.Config.Image}}' 4d7f05e45117 3ca389250692  # 返回两个镜像的镜像名字
```



### docker save [OPTIONS] IMAGE [IMAMGE...]

将镜像保存下来

```dockerfile
-o, --output string   Write to a file, instead of STDOUT  # 输出到屏幕上


eg: docker save -o my_ubuntu_v3.tar runoob/ubuntu:v3
```



### docker push [OPTIONS] NAME[:TAG]

推送镜像

```dockerfile
--disable-content-trust   Skip image signing (default true)  # 忽略镜像的校验


eg: docker push myapache:v1
```



### docker rmi [OPTIONS] IMAGE [IMAGE...]

删除镜像

```dockerfile
-f :强制删除；

--no-prune :不移除该镜像的过程镜像，默认移除；


eg: docker rmi -f runoob/ubuntu:v4
```



## 容器使用

下面是关于容器的使用



### docker ps [OPTIONS]

查看容器

```dockerfile
--all, -a :显示所有的容器，包括未运行的。

--filter, -f :根据条件过滤显示的内容。

--format :指定返回值的模板文件。

--latest, -l :显示最近创建的容器。

--last, -n :列出最近创建的n个容器。

--no-trunc :不截断输出。

-q :静默模式，只显示容器编号。

-s :显示总的文件大小。


eg: docker ps --filter status=paused/running  # 显示状态为暂停或者运行中的镜像

eg: docker ps --filter ancestor=nginx  # 显示祖先是nginx的镜像

eg: docker ps --filter "label=color"  # 显示颜色

eg: docker ps -n 5  # 显示最近创建的5个容器
```



查询条件比较丰富，具体如下：

![](https://wx4.sinaimg.cn/large/007FyU7Tly1g36f76fscfj30hs0gddk2.jpg)





### docker port CONTAINER [PRIVATE_PORT[/PROTO]]

列出指定端口和协议的容器

```dockerfile
eg: docker port test 127.0.0.1/tcp  # 列出指定端口和协议的容器
```



### docker logs [OPTIONS] CONTAINER 

查看容器的日志

```dockerfile
-f : 跟踪日志输出

--since :显示某个开始时间的所有日志

-t : 显示时间戳

--tail :仅列出最新N条容器日志


eg: docker logs --since="2019-05-18" --tail=10 container_id   # 列出从指定时间开始的最新的n条日志
```



### docker top CONTAINER

查看容器内正在运行的进程信息

```dockerfile
eg: docker top container_id
```



### docker stop [OPTIONS] CONTAINER [CONTAINER...]

停止容器

```dockerfile
--time , -t	10	Seconds to wait for stop before killing it  # 等待指定秒的时间后停止容器


eg: docker stop -t=20 container_id  # 等待20s后停止容器
```



### docker start [OPTIONS] CONTAINER [CONTAINER...]

启动容器

```dockerfile
eg: docker start -i container_id
```



### docker restart [OPTIONS] CONTAINER [CONTAINER...]

重启容器

```dockerfile
--time , -t	10	Seconds to wait for stop before killing the container  # 等待指定秒的时间后重启容器


eg: docker restart -t=20 container_id  # 等待20s后重启容器
```



### docker rm [OPTIONS] CONTAINER [CONTAINER...]

删除容器

```dockerfile
-f, --force     Force the removal of a running container (uses SIGKILL)  # 强制删除
-l, --link      Remove the specified link  # 移除容器之间的网络连接
-v, --volumes   Remove the volumes associated with the container  # 删除容器之间的卷


eg: docker rm -f db01 db02
```



### docker create [OPTIONS] IMAGE [COMMAND] [ARG...]

创建容器，但是不会启动它，参数和run相同，具体去[官网](https://docs.docker.com/engine/reference/commandline/create/)看

```dockerfile
eg: docker create -it --name mynginx nginx:latest  # 使用镜像nginx:latest来创建一个容器，并且命名为mynginx
```



### docker kill [OPTIONS] CONTAINER [CONTAINER...]

杀掉容器



和`docker stop`的区别在于，不管容器是否同意终止进程，而是强行终止容器；但是在docker stop会先发送`SIGTERM`信号，在过一段时间（10s）以后，再发送`SIGKILL`信号，Docker在接收到前者的信号以后，进行一些退出前的工作，比如保存状态、处理当前请求等等，接着才会执行退出的操作。因此，如果想要优雅的退出，则使用`docker stop`。

```dockerfile
-s, --signal string   Signal to send to the container (default "KILL")  # 向容器发送信号

eg: docker kill -s KILL mynginx
```



### docker attach [OPTIONS] CONTAINER

进入到容器中



根据网上别人的建议，不太建议使用于生产环境，因为如果多人访问到容器中，其中一个窗口阻塞也会导致其他的窗口阻塞，因此比较推荐使用`docker exec`。



```dockerfile
eg: docker attach container_id
```



### docker exec [OPTIONS] CONTAINER COMMAND [ARG...]

进入到容器中

```dockerfile
-d :分离模式: 在后台运行

-i :即使没有附加也保持STDIN 打开

-t :分配一个伪终端


eg: docker exec -it mynginx /bin/sh /root/runoob.sh  # 已交互模式在容器中执行runoob脚本
```



### docker export [OPTIONS] CONTAINER

将容器以tar文件的形式导出



和`docker save`的区别是，`docker save`持久化的是镜像，而`docker export`持久化的是容器。

```dockerfile
-o :将输入内容写到文件。


eg: docker export -o mysql-`date +%Y%m%d`.tar a404c6c174a2
```







