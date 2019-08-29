创建容器
docker run --name=docker_nginx_v1 -d -p 80:80 nginx:v1

有些是一个减号，有些是两个减号 --name="容器新名字": 为容器指定一个名称；-d: 后台运行容器，并返回容器ID，也即启动守护式容器；-i：以交互模式运行容器，通常与 -t 同时使用；-t：为容器重新分配一个伪输入终端，通常与 -i 同时使用；-P: 随机端口映射；-v:文件映射  -p: 指定端口映射，有以下四种格

docker cp 1.txt containerName:/usr/local
docker exec -it containerName /bin/bash
docker inspect --format=‘{{.NetworkSetting.IpAdress}}’ containerName  获取容器所需要的信息
docker rm containerName
docker stop 
docker kill
docker start
docker rmi 镜像

docker logs -f -t --tail 容器ID
docker top 容器ID  查看容器内运行的进程

将镜像保存起来
1. docker commit 容器名 镜像名
2. docker save 镜像名 -o 路径/文件.tar
3. docker load -i 文件.tar

dockerfile

| **命令**                          | **作用**                                                     |
| --------------------------------- | ------------------------------------------------------------ |
| FROM image_name:tag               | 定义了使用那个基础镜像构建                                   |
| MAINTAINER user_name              | 声明镜像的创建者                                             |
| ENV  key value                    | 设置环境变量                                                 |
| RUN command                       |                                                              |
| Add source_dir/file dest_dir/file | 将宿主机的文件复制到容器内，压缩文件会自动解压               |
| COPY                              | 类似上  但不能解压                                           |
| WORKDIR path_dir                  | 设置工作目录 终端登录进来的落脚点                            |
| EXPOSE                            | 当前容器对外暴露的端口                                       |
| CMD                               | 指定一个容器启动时要运行的命令, CMD 会被docker run 之后的参数替换 |
| ENTRYPOINT                        | 指定一个容器启动时要运行的命令                               |

docker build -t 新镜象名字:TAG

```dockerfile
#基于哪个镜像
FROM frolvlad/alpine-oraclejdk8:slim
#将本地文件夹挂载到当前容器
VOLUME /tmp
#复制文件到容器，
ADD eureka-server-0.0.1-SNAPSHOT.jar app.jar
#RUN bash -c 'touch /app.jar'
#配置容器启动后执行的命令
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
#声明需要暴露的端口
EXPOSE 8761
```



Rancher 安装

docker pull rancher/server
docker run -di --name=rancher -p 9090:8080 rancher/server



var qsCloudALogin = getCookie('qsCloudALogin');
if(qsCloudALogin && qsCloudALogin.length > 0){
	localStorage.setItem('qsCloudALogin', qsCloudALogin);
} else {
	localStorage.removeItem('qsCloudALogin');
}