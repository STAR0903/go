# Docker操作redis

#### 笔记来源

`https://blog.csdn.net/l1028386804/article/details/105682096`

#### 查找镜像

```bash
按名称搜索图像
docker search redis
```

```bash
按名称搜索并显示非截断描述（--no-trunc）
docker search --stars=3 --no-trunc redis
```

```bash
按名称redis搜索出星数至少为3颗星的镜像
docker search --filter stars=3 redis
```

```bash
显示名称中包含“redis”的图像，并且是自动构建
docker search --filter is-automated redis
```

```bash
显示的图像名称包含“redis”，至少3颗星，并且是官方版本
$ docker search --filter "is-official=true" --filter "stars=3" redis
```

```bash
格式化选项（--format）使用Go模板漂亮地打印搜索输出
1.使用不带标头的模板，Name并StarCount为所有图像输出 以冒号分隔的条目和条目：
docker search --format "{{.Name}}:{{.StarCount}}" redis
2.输出表格格式:
docker search --format "table {{.Name}}\t{{.IsAutomated}}\t{{.IsOfficial}}" redis
```

#### 拉取镜像

```bash
不指定版本,则拉取最新版本的镜像
docker pull redis
```

```bash
指定版本
docker pull redis:5.0.5
```

#### 查看拉取成功的镜像

```bash
docker images
```

#### 启动镜像及参数说明

```bash
docker run --name redis -p 6379:6379 --restart=always -v $PWD/data:/data  -d redis:5.0.5 redis-server --appendonly yes daemonize yes
参数说明：
#本地运行
-d
#本地端口:Docker端口
6379:6379
#指定驱动盘
-v
#Redis的持久化文件存储
$PWD/data
#docker的镜像名
redis
#redis服务器
redis-server
#开启持久化
--appendonly yes
#这个运行的镜像的名称
--name
#守护进程
daemonize yes
#Docker启动容器就启动
--restart=always
```

#### 停止正在运行的镜像

```bash
redis为前面设置的镜像名称
docker stop redis
```

#### 删除镜像

```bash
docker rm redis
```

#### 重启镜像

```bash
docker start redis
```

#### 获取 container ID 或者名字

```bash
docker container ls -a
```

#### 删除指定的container

```bash
如果你要删除的 container 还是运行状态，那么就要先把容器停止了：
docker  container  stop   CONTAINER_ID
删除：(这两条命令都是删除同一个容器)
docker   container  rm  CONTAINER_ID  
或者 docker  container  rm  CONTAINER_NAME 
```

#### 批量获取容器ID

```bash
docker container ls -a -q
```

#### 批量获取镜像ID

```bash
docker image ls -a -q   
```

#### 批量停止容器

```bash
docker container   stop   $(docker  container  ls   -a  -q)
```

#### 批量删除容器

```bash
docker   container   rm  $(docker  container  ls   -a  -q)
```

#### 通过image的id来指定删除镜像

```bash
docker rmi <image id>
```

#### 删除untagged images

也就是那些id为 `<None>`的image的话可以用

```bash
docker rmi $(docker images | grep "^<none>" | awk "{print $3}")
```

#### 删除全部images

```bash
docker rmi $(docker images -q)
```

#### 访问容器

```bash
docker exec -it redis bash
```

#### 使用redis-cli访问容器内redis

```bash
docker exec -it redis redis-cli
```
