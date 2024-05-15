## 安装：

```sh
# 设置docker仓库
yum install -y yum-utils \
           device-mapper-persistent-data \
           lvm2 --skip-broken
           
# 设置docker镜像源
yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    
sed -i 's/download.docker.com/mirrors.aliyun.com\/docker-ce/g' /etc/yum.repos.d/docker-ce.repo

yum makecache fast


# 安装docker-ce社区免费版本。稍等片刻，docker即可安装成功
yum install -y docker-ce


# 关闭防火墙
systemctl stop firewalld
# 禁止开机启动防火墙
systemctl disable firewalld

systemctl start docker  # 启动docker服务

systemctl stop docker  # 停止docker服务

systemctl restart docker  # 重启docker服务


# 配置docker阿里镜像
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://as08lme3.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 命令：

![image-20221110171813983](C:\Users\Yefl\AppData\Roaming\Typora\typora-user-images\image-20221110171813983.png)

![image-20221110171845441](C:\Users\Yefl\AppData\Roaming\Typora\typora-user-images\image-20221110171845441.png)

```sh
docker -v //查看版本
docker pull nginx //默认 最新 latest
docker pull nginx:版本号
docker images 查看版本
docker save 镜像名 -o /输出的位置
docker load -i /输入的镜像文件

docker ps //查看运行的容器
docker ps -a //查看全部容器
docker rm 容器名 //删除容器
docker exec -it 容器名 /bin/bash //进入容器
docker logs 容器名 //查看日志
docker stop 容器名 //停止容器
docker start 容器名 //启动容器
docker pause 容器名 //暂停
docker unpause 容器名 //取消暂停
docker run --name containerName -p 80:80 -d nginx
# 说明 ###################
docker run ：创建并运行一个容器
--name : 给容器起一个名字，比如叫做mn
-p ：将宿主机端口与容器端口映射，冒号左侧是宿主机端口，右侧是容器端口
-d：后台运行容器
nginx：镜像名称，例如nginx
##########################

# docker中mysql的运行
docker run --name msql -v /msql/conf:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=root -d mysql --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci

# docker中rabbitmq的运行
docker run \
 -e RABBITMQ_DEFAULT_USER=root \
 -e RABBITMQ_DEFAULT_PASS=123456 \
 --name mq \
 --hostname mq1 \
 -p 15672:15672 \
 -p 5672:5672 \
 -d \
 rabbitmq:3-management
 
# docker中redis的运行
docker run --name myredis -p 6379:6379 -d redis redis-server --appendonly yes
# 创建和启动容器
# redis-server --appendonly yes 设置持久化手段

# docker中es的安装
# 1.配置网络 让es 搜索数据库】和kibana 【可视化工具】容器互联！
docker network create es-net
# 2.需要进入到对应存储文件的位置
# 导入数据
docker load -i es.tar
docker load -i kibana.tar
# 3.启动容器
#单点es容器的运行
docker run -d \
  --name es \
    -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
    -e "discovery.type=single-node" \
    -v es-data:/usr/share/elasticsearch/data \
    -v es-plugins:/usr/share/elasticsearch/plugins \
    --privileged \
    --network es-net \
    -p 9200:9200 \
    -p 9300:9300 \
elasticsearch:7.12.1
#在浏览器中输入：[http://ip地址:9200即可看到elasticsearch的响应结果！
#单点运行kibana! 提供数据可视化！
docker run -d \
--name kibana \
-e ELASTICSEARCH_HOSTS=http://es:9200 \
--network=es-net \
-p 5601:5601  \
kibana:7.12.1
#此时，在浏览器输入地址访问：http://ip地址:5601，即可看到结果
# 4.配置ik【中文】分词器
# 查看es配置文件地址
docker volume inspect es-plugins
#将ik解压导入到，查出来的文件_data地址下，然后重启es即可
docker restart es

# docker中注册中心的安装
docker pull nacos/nacos-server
# 启动
docker run --name nacos-quick -e MODE=standalone -p 8848:8848 -d nacos/nacos-server:latest
# http://ip地址:8848/nacos  输入账号： nacos 密码 nacos
```

## Dockerfile

```dockerfile
# 指定基础镜像
FROM java:8-alpine

# 拷贝java项目的jar包
COPY ./store-admin-1.0-SNAPSHOT.jar /tmp/admin.jar
COPY ./store-front-carousel-1.0-SNAPSHOT.jar /tmp/carousel.jar
COPY ./store-front-cart-1.0-SNAPSHOT.jar /tmp/cart.jar
COPY ./store-front-category-1.0-SNAPSHOT.jar /tmp/category.jar
COPY ./store-front-collect-1.0-SNAPSHOT.jar /tmp/collect.jar
COPY ./store-front-order-1.0-SNAPSHOT.jar /tmp/order.jar
COPY ./store-front-product-1.0-SNAPSHOT.jar /tmp/product.jar
COPY ./store-front-user-1.0-SNAPSHOT.jar /tmp/user.jar
COPY ./store-gateway-1.0-SNAPSHOT.jar /tmp/gateway.jar
COPY ./store-search-1.0-SNAPSHOT.jar /tmp/search.jar
COPY ./store-static-1.0-SNAPSHOT.jar /tmp/static.jar

# PORT
EXPOSE 3000

# 入口，java项目的启动命令
ENTRYPOINT java -jar /tmp/admin.jar & java -jar /tmp/carousel.jar & java -jar /tmp/cart.jar & java -jar /tmp/category.jar & java -jar /tmp/collect.jar & java -jar /tmp/order.jar & java -jar /tmp/product.jar & java -jar /tmp/user.jar & java -jar /tmp/gateway.jar & java -jar /tmp/search.jar & java -jar /tmp/static.jar
```

### 构建

```sh
docker build -t yccloudstore:1.0 .
```
