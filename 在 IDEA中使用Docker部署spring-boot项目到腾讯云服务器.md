## 在 IDEA中使用Docker部署spring-boot项目到腾讯云服务器

准备环境：

开发工具：Win10  IntelliJ IDEA 2018.1.5 x64

云服务器：Centos7.5，docker1.13.1

默认Idea已经下载了Docker插件

默认虚拟机docker已经装了jdk

## 一、Docker开启远程访问

```
#先备份docker配置
[root@izwz9eftauv7x69f5jvi96z docker]# cp /usr/lib/systemd/system/docker.service /usr/lib/systemd/system/docker.service_bak
#编辑docker.service 
[root@izwz9eftauv7x69f5jvi96z docker]# vim /usr/lib/systemd/system/docker.service
#修改ExecStart
在ExecStart=/usr/bin/dockerd-current  后增加 -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
#修改后如下
ExecStart=/usr/bin/dockerd-current -H tcp://0.0.0.0:2375  -H unix:///var/run/docker.sock \
          --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current \
          --default-runtime=docker-runc \
          --exec-opt native.cgroupdriver=systemd \
          --userland-proxy-path=/usr/libexec/docker/docker-proxy-current \
          --init-path=/usr/libexec/docker/docker-init-current \
          --seccomp-profile=/etc/docker/seccomp.json \
          $OPTIONS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $ADD_REGISTRY \
          $BLOCK_REGISTRY \
          $INSECURE_REGISTRY \
          $REGISTRIES
ExecReload=/bin/kill -s HUP $MAINPID
#重新加载配置文件
[root@izwz9eftauv7x69f5jvi96z docker]# systemctl daemon-reload    
#重启服务
[root@izwz9eftauv7x69f5jvi96z docker]# systemctl restart docker.service 
#查看端口是否开启
[root@izwz9eftauv7x69f5jvi96z docker]# netstat -nptl
#直接curl看是否生效
[root@izwz9eftauv7x69f5jvi96z docker]# curl http://127.0.0.1（docker服务器IP）:2375/info
局域网测试是否通 用telnet 命令或 在浏览器里直接访问
telnet 192.168.5.100 2375
如果服务器测试都不通，那可能没有启动好，
如果服务器测试通，外网不通那检查一下防火墙
参考文章：https://blog.csdn.net/feiz3020/article/details/80257984
https://cloud.tencent.com/developer/article/1370022
```

## 二、IntelliJ IDEA安装Docker插件

![img](https://ask.qcloudimg.com/http-save/yehe-1231850/q5023hncys.jpeg?imageView2/2/w/1620) 

IDEA连接云服务器上的docker:

![img](https://images2018.cnblogs.com/blog/1353814/201808/1353814-20180816165514947-108846189.png) 

tcp://localhost(安装docker的宿主机的IP):2375

连接成功下方会提示**Connection successful**

## 三.Springboot项目添加docker-maven-plugin插件 

```
<!--使用docker-maven-plugin插件-->
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>1.0.0</version>
 
    <!--将插件绑定在某个phase执行-->
    <executions>
        <execution>
            <id>build-image</id>
            <!--将插件绑定在package这个phase上。也就是说，用户只需执行mvn package ，就会自动执行mvn docker:build-->
            <phase>package</phase>
            <goals>
                <goal>build</goal>
            </goals>
        </execution>
    </executions>
 
    <configuration>
        <!--指定生成的镜像名 随意取-->
        <imageName>baymax/${project.artifactId}</imageName>
        <!--指定镜像的标签-->
        <imageTags>
            <imageTag>latest</imageTag>
        </imageTags>
        <!-- 指定 Dockerfile 所在的路径-->
        <dockerDirectory>${project.basedir}/src/main/docker</dockerDirectory>
 
        <!--指定远程 docker 的服务器地址以及端口-->
        <dockerHost>http://192.168.159.130:2375</dockerHost>
 
        <!-- 这里是复制 jar 包到 docker 容器指定目录配置 -->
        <resources>
            <resource>
                <targetPath>/</targetPath>
                <!--jar 包所在的路径  此处配置的 即对应 target 目录-->
                <directory>${project.build.directory}</directory>
                <!-- 需要包含的 jar包 ，这里对应的是 Dockerfile中添加的文件名　-->
                <include>${project.build.finalName}.jar</include>
            </resource>
        </resources>
    </configuration>
</plugin>
```

注意：（参考原文：https://blog.csdn.net/u010046908/article/details/56008445 ）

Spring Boot Maven plugin 提供了很多方便的功能： 

1）它收集的类路径上所有 jar 文件，并构建成一个单一的、可运行的jar，这使得它更方便地执行和传输服务。 
2）它搜索的 public static void main() 方法来标记为可运行的类。 
3）它提供了一个内置的依赖解析器，用于设置版本号以匹配 Spring Boot 的依赖。您可以覆盖任何你想要的版本，但它会默认选择的 Boot 的版本集。

Spotify 的 docker-maven-plugin 插件是用于构建 Maven 的 Docker Image 

1）imageName指定了镜像的名字，本例为 springio/lidong-spring-boot-demo 
2）dockerDirectory指定 Dockerfile 的位置 

3）resources是指那些需要和 Dockerfile 放在一起，在构建镜像时使用的文件，一般应用 jar 包需要纳入。



## 四.创建docker文件夹和Dockerfile文件，docker要在src/main里 

![img](https://images2018.cnblogs.com/blog/1353814/201808/1353814-20180816165713892-288885632.png) 

Dockerfile的内容：

```
#指定基础镜像，在其上进行定制
FROM java:8

#维护者信息
MAINTAINER zhaozongyuan <mrzongyuanzhao@163.com>

#这里的 /tmp 目录就会在运行时自动挂载为匿名卷，任何向 /data 中写入的信息都不会记录进容器存储层
VOLUME /tmp

#复制上下文目录下的target/demo-1.0.0.jar 到容器里
ADD demo-0.0.1-SNAPSHOT.jar app.jar

#bash方式执行，使demo-0.0.1-SNAPSHOT.jar可访问
#RUN新建立一层，在其上执行这些命令，执行结束后， commit 这一层的修改，构成新的镜像。
RUN sh -c 'touch /app.jar'

ENV JAVA_OPTS=""

#指定容器启动程序及参数   <ENTRYPOINT> "<CMD>"
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar /app.jar" ]
```

VOLUME 指定了临时文件目录为/tmp。其效果是在主机 /var/lib/docker 目录下创建了一个临时文件，并链接到容器的/tmp。改步骤是可选的，如果涉及到文件系统的应用就很有必要了。/tmp目录用来持久化到 Docker 数据文件夹，因为 Spring Boot 使用的内嵌 Tomcat 容器默认使用/tmp作为工作目录 

项目的 jar 文件作为 “app.jar” 添加到容器的 

ENTRYPOINT 执行项目 app.jar。为了缩短 Tomcat 启动时间，添加一个系统属性指向 “/dev/urandom” 作为 Entropy Source 

原文：https://blog.csdn.net/u010046908/article/details/56008445 

## 五、在idea的右边找到Maven Projects，找到Lifecycle，双击package打包，第一次打包比较慢，需要骚骚的等等。

![img](https://images2018.cnblogs.com/blog/1353814/201808/1353814-20180816165833130-2033803430.png)  

看到以下就说明成功了 

![img](https://images2018.cnblogs.com/blog/1353814/201808/1353814-20180816165845439-2049986873.png) 

![img](https://images2018.cnblogs.com/blog/1353814/201808/1353814-20180816165852296-1512021986.png) 

## 六、创建容器，找到刚刚生成的镜像，点击创建容器 

![img](https://images2018.cnblogs.com/blog/1353814/201808/1353814-20180816165912037-1738386178.png) 

![img](C:\Users\zhao\Desktop\1353814-20180816165918487-1796173126.png) 

Image ID ：刚才上传到云服务器上的镜像名

Container name ：给启动起来的镜像（即容器）取个名字

Bind ports ：将云服务器上的8080端口映射到容器上的8082端口。

SpringBoot内置tomcat默认端口是8080，所以可以设置将宿主机上端口映射8080端口即可。

## 七、设置好后，启动容器，启动成功后去虚拟机查看是否启动成功

可以看到，容器启动成功。

![img](https://images2018.cnblogs.com/blog/1353814/201808/1353814-20180816165946389-1240775926.png)

10.访问虚拟机ip+端口号

例如：http://localhost:8080/hello



## 扩展

启动docker平台：
[xxx@root# systemctl start docker   #启动 docker 服务  
[xxx@root# systemctl enable docker  #设置开机启劢 docker 服务 
[xxx@root# docker version   #显示 Docker 版本信息
[xxx@root# docker info  #查看 docker 信息（确认服务运行）显示 Docker 系统信息，包括镜像和容器数。

访问正在运行的 container 容器实例 .
语法： docker exec -it <container id | name> /bin/bash 
[xxx@root]# docker exec -it 87fadc0249a9 /bin/bash  #进入容器 

删除镜像
[xxx@root]# docker  rmi   image_name/ID
若要删除所有的image, 使用命令：
[xxx@root]# docker rmi  $( docker  images -q )

根据容器当前状态做一个 image镜像：
语法： docker commit <container的ID> <image_name> 例
[xxx@root]# docker commit  1d3563200047 centos:nmap 
第二种方法是编写Dockerfile的形式（这个就不贴了，可以百度有很多）

删除指定 container ： rm 
[xxx@root]# docker rm 1a63ddea6571 
解决：你可以先把容器1a63ddea6571 关闭，然后再删除戒加-f 强制删除 
[xxx@root]# docker rm -f 1a63ddea6571

查看所有的容器 
[xxx@root]# docker ps -a 
查看容器的日志

[xxx@root]# docker logs c4a213627f1b

原文：https://blog.csdn.net/letian1992/article/details/80589184 

