#### 容器（Container）：轻量级、可移值的运行环境，包含了应用程序及其所有依赖项。容器共享操作系统内核。
#### 镜像（Image）：Docker 容器的 “源代码”，是一个只读的文件系统模板，包含了运行应用所需的所有文件、依赖和配置。它类似于虚拟机的镜像，但更轻量、更高效。

#### 下载镜像
##### 检索：docker search
##### 下载：docker pull
```shell
docker pull nginx:latest
```
下载特定版本使用 docker pull 镜像名：标签(版本)
##### 列表：docker images
##### 删除：docker rmi

---
#### 启动容器
##### 运行：docker run
```Shell
docker run -d --name mynginx -p 80:80 nginx
```
- -d 代表后台启动，不会阻塞终端
- --name mynginx 为容器指定自定义名称，在后续的start，stop，rm，exec都需要指定容器的标识
- -p 80:80 用来设置端口映射，前一个80是宿主机端口，后一个80是容器端口
##### 查看：docker ps 
##### 停止：docker stop 
##### 启动：docker start
##### 重启：docker restart
##### 状态：docker stats 
##### 日志：docker logs
##### 进入：docker exec
``` Shell
docker exec -it mynginx /bin/bash
```
- 进入到名为 `mynginx` 的 Docker 容器内部，并且启动一个交互式的 Bash 终端
``` Shell
echo "<h1>Hello, Docker!!!</h1>" > index.html 
```
- 创建或者覆盖一个名为 `index.html` 的文件，并且把指定的 HTML 内容写入到这个文件中

##### 删除：docker rm
---
#### 保存镜像
##### 提交：docker commit
将容器当前状态保存为一个新的镜像
##### 保存：docker save
将Docker镜像导出为一个tar文件
##### 加载：docker load
将导出的tar文件重新加载为Docker镜像

---
#### 分享社区
##### 登录：docker login
##### 命名：docker tag
为本地镜像添加一个新的标签，相当于给镜像起一个别名
##### 推送：docker push
将本地镜像推动到远程镜像仓库中

### Docker存储：通过多种方式实现了数据的持久化
#### 1. 目录挂载（Bind Mounts）
定义：直接将主机文件系统中的路径映射到容器内部，容器对挂载路径的读写操作会直接影响主机文件。
#### 2. 卷映射（Volumes）
定义：Docker 在主机上创建并管理一个专用目录（如 `/var/lib/docker/volumes/my-volume/_data`），容器通过挂载该目录访问数据。

---
#### Docker网络
**Docker 网络是容器间通信、容器与外部世界交互的基础架构。**

**# docker为每个容器分配唯一ip，使用容器ip+容器端口可以相互访问**

#### [[Docker Compose]]
