# Jenkins 构建前端镜像并上传到制品库

上一章我们写到，如何使用 Nexus 托管我们的镜像。这一章我们就将 Nexus 和 Jenkins 结合起来构建部署。

## Jenkins 登录认证制品库

想使用 Jenkins 推送镜像到制品库，必须先登录制品库。进入 Jenkins 容器，使用 docker login 登录，然后退出即可：

```bash
docker exec -it jenkins /bin/bash
docker login 制品库地址:端口
exit;
```

### Nginx 服务器登录认证制品库

编辑 /etc/docker/daemon.json 文件，将制品库地址写入，然后重启 docker 服务

```bash
vi /etc/docker/daemon.json
sudo systemctl daemon-reload
sudo systemctl restart docker
```

```json
{
    "insecure-registries": [
        "192.168.1.42:8082"
    ]
}
```

接着使用 docker login 命令进行登录

```bash
docker login 制品库地址:端口
```

### 使用 DockerFile 构建前端镜像

在此之前，我们使用 **构建压缩包** => **直接上传到目标服务器** => **进行解压部署** 的方式部署前端。
在这里，我们更换为部署docker镜像。既然要制作自己的镜像，那就少不了 **DockerFile**。

在前面我们写到过，DockerFile 是一个镜像制作过程的步骤描述。那么我们也可以简单的描述下我们自己的步骤。

我们在代码(vue-demo)目录下，新建一个文件。叫 Dockerfile （注意，file全小写）：

```bash
FROM nginx:1.15-alpine
COPY dist /usr/share/nginx/html
WORKDIR /usr/share/nginx/html
```

此Dockerfile声明了以下步骤：
1. 拉取一个 nginx 1.15-alpine 版本的镜像。
2. 将当前目录下的 dist 文件夹拷贝到镜像的 /usr/share/nginx/html 文件夹。
3. 声明启动容器时，在 /usr/share/nginx/html 下面执行。

接着提交到代码库中。

### 修改 Jenkins 执行脚本

我们打开之前 Jenkins 任务的编辑页面，改为以下脚本：

```bash
set -e
timestamp=`date '+%Y%m%d%H%M%S'`

node -v
npm -v

npm install -g cnpm --registry=https://registry.npm.taobao.org

cnpm install

npm run build

# 编译docker镜像
docker build -t 192.168.1.42:8082/fe/nginx-fe-$timestamp .

# 推送docker镜像到制品库
docker push 192.168.1.42:8082/fe/nginx-fe-$timestamp

# 远程执行命令部署镜像
ssh -o StrictHostKeyChecking=no root@192.168.1.41 "docker pull 192.168.1.42:8082/fe/nginx-fe-$timestamp && \
docker stop jenkins-test && \
docker rm jenkins-test && \
docker run -p 80:80 -itd \
--name jenkins-test \
--restart always \
192.168.1.42:8082/fe/nginx-fe-$timestamp"
```

> 在shell脚本中，声明一个变量只需要 变量名=值 即可。在命令中用 $-变量名 进行使用。
timestamp=`date '+%Y%m%d%H%M%S'`：这个代表执行 date '+%Y%m%d%H%M%S' 这条命令，并赋值给 timestamp 这个变量。
date '+%Y%m%d%H%M%S' 代表输出当前时间的年月日时分秒
StrictHostKeyChecking: https://cikeblog.com/ssh-cant-established.html

这里可以看到，我们将之前脚本后半部分的流程：

* 压缩打包资源
* 通过 scp 上传压缩包
* ssh 远程执行解压部署命令

替换为：

* Dockerfile构建镜像
* 上传镜像到制品库
* 远程执行命令拉取镜像，停止容器，删除容器，启动新拉取的镜

成功

![deploy-done](./assets/deploy-done.jpg)

