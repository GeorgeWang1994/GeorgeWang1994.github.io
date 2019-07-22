


创建Dockerfile主要包括四个方面，`基础镜像信息`、`维护者信息`、`镜像操作命令`、`容器启动时操作命令`；



## 基础镜像信息命令



### FROM



告知创建的镜像来自于哪个基础镜像；每个Dockerfile文件的开头必须是该命令，如果执行多条`FROM`命令，代表操作多个镜像；



````dockerfile
FROM python:3.7-slim
````



## 维护者信息



### MAINTAINER



当前开发者信息



```dockerfile
MAINTAINER georgewang1994@163.com
```



## 镜像操作命令



镜像操作命令主要是`RUN`、`CMD`、`ENTRYPOINT`、`COPY`、`ADD`、`VOLUMN`、`WORKDIR`、`ENV`、`EXPOSE`、`USER`、`ARG`；



### RUN



执行命令，可以执行多条，每执行一条就增加一层镜像，如果需要在一个命令中执行多个操作，可以用`\`来划分；该命令通常用于下载/更新包；



```dockerfile
RUN python -m pip install --upgrade pip
```



### COPY



将Dockerfile文件所在目录的文件或者目录复制到指定的路径中，如果不存在则会创建；



```dockerfile
COPY requirements.txt requirements.txt 
```



### ADD



复制指定的文件、目录、URL、压缩文件等到指定的路径中（压缩文件会自动解压），也就是该命令可以完成所有`COPY`所有的工作；



```dockerfile
ADD filename.tar.gz .
```



### VOLUME



将本地路径的文件或目录挂载到镜像中，通常用于数据库的存放；



```dockerfile
VOLUME ["/data"]
```



### WORKDIR



为后续的`RUN、CMD、ENTRYPOINT`命令配置操作路径，如果执行多次，会在前面的基础上进行路径的叠加；



```dockerfile
WORKDIR /a  # 此时工作路径为/a

WORKDIR b  # 此时工作路径为/a/b

WORKDIR c  # 此时工作路径为/a/b/c
```



### ENV



设置环境变量，提供给后续的`RUN、CMD、ENTRYPOINT`命令，并且在容器运行时一直保存；



```dockerfile
ENV NAME "george"

RUN echo "Hello" $NAME  # 输出结果为Hello george
```



### EXPOSE



曝露端口号，可以指定多个端口号，在启动容器的时候提供`-p/-P`，它们的区别在于前者显式将一个或者一组端口从容器里绑定到宿主机上，后者自动的将 EXPOSE 指令相关的每个端口映射到宿主机的端口上；



```dockerfile
EXPOSE 20 80 8443
```



### ARG



用于指定传递给构建**运行时**的变量，而`ENV`则是使用在Dockerfile文件构建的过程中；



## 容器启动时操作命令



### CMD



启动容器时执行的默认命令，每个Dockerfile文件只能有一条改命令，如果存在多条，则以最后一条命令为准；需要注意的是，该命令会被`docker run`后面的命令给替代；执行的方式分为两种，分别是`executor`和`command`（`RUN命令同样如此`）；



```dockerfile
CMD python manage.py runserver 
```



### ENTRYPOINT



启动容器时执行的命令，和`CMD`不同的是，该命令不会被`docker run`后面的命令给替换掉，相反该命令会肯定执行，接着可以在`CMD`或者`docker run`后面的命令加上一定的参数；



```dockerfile
ENTRYPOINT python manage.py runserver

CMD 0:8000  # 最终执行的效果就是python manage.py runserver 0:8000
```



### USER



指定运行容器时的用户名或者用户ID，后续的`RUN`命令也会使用该指定的用户，当用户不需要管理员权限的时候，则可以指定该运行用户；需要注意的是`docker run`命令可以通过`-u`参数覆盖掉所指定的用户；



```dockerfile
USER george

RUN useradd 
```





## 参考



[RUN vs CMD vs ENTRYPOINT - 每天5分钟玩转 Docker 容器技术（17）](https://www.cnblogs.com/CloudMan6/p/6875834.html)

[Dockerfile 中的 COPY 与 ADD 命令](https://www.cnblogs.com/sparkdev/p/9573248.html)

[docker 网络之端口映射不完全探索](https://learnku.com/articles/25018)

[Dockerfile文件详解](https://www.cnblogs.com/panwenbin-logs/p/8007348.html)

[Conditional ENV in Dockerfile](https://stackoverflow.com/questions/37057468/conditional-env-in-dockerfile)