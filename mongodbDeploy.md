参考：

[https://www.cnblogs.com/evescn/p/16203350.html](https://www.cnblogs.com/evescn/p/16203350.html)



## docker-compose.yml

```yaml
version: "3"

services:
  #主节点
  mongodb1:
    image: mongo:6.0.2
    container_name: mongo1
    restart: always
    ports:
      - 37017:27017
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=123456
    command: mongod --replSet rs0 --keyFile /mongodb.key
    volumes:
      - /etc/localtime:/etc/localtime
      - ./data/mongodb/mongo1/data:/data/db
      - ./data/mongodb/mongo1/configdb:/data/configdb
      - ./data/mongodb/mongo1/mongodb.key:/mongodb.key
    networks:
      - mongoNet
    entrypoint:
      - bash
      - -c
      - |
        chmod 400 /mongodb.key
        chown 999:999 /mongodb.key
        exec docker-entrypoint.sh $$@
  # 副节点
  mongodb2:
    image: mongo:6.0.2
    container_name: mongo2
    restart: always
    ports:
      - 37018:27017
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=123456
    command: mongod --replSet rs0 --keyFile /mongodb.key
    volumes:
      - /etc/localtime:/etc/localtime
      - ./data/mongodb/mongo2/data:/data/db
      - ./data/mongodb/mongo2/configdb:/data/configdb
      - ./data/mongodb/mongo2/mongodb.key:/mongodb.key
    networks:
      - mongoNet
    entrypoint:
      - bash
      - -c
      - |
        chmod 400 /mongodb.key
        chown 999:999 /mongodb.key
        exec docker-entrypoint.sh $$@
  # 副节点
  mongodb3:
    image: mongo:6.0.2
    container_name: mongo3
    restart: always
    ports:
      - 37019:27017
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=123456
    command: mongod --replSet rs0 --keyFile /mongodb.key
    volumes:
      - /etc/localtime:/etc/localtime
      - ./data/mongodb/mongo3/data:/data/db
      - ./data/mongodb/mongo3/configdb:/data/configdb
      - ./data/mongodb/mongo3/mongodb.key:/mongodb.key
    networks:
      - mongoNet
    entrypoint:
      - bash
      - -c
      - |
        chmod 400 /mongodb.key
        chown 999:999 /mongodb.key
        exec docker-entrypoint.sh $$@
networks:
  mongoNet:
    driver: bridge


```



## <font style="color:rgb(49, 70, 89);">生成 </font><font style="color:rgb(49, 70, 89);background-color:rgb(242, 244, 245) !important;">keyFile</font>

```shell
## 400权限是要保证安全性，否则mongod启动会报错
openssl rand -base64 756 > mongodb.key
chmod 400 mongodb.key

```

## 部署

```shell
# mkdir /data/mongodb/mongo{1,2,3}/{data,configdb} -pv

## 提供redis.conf配置
cp mongodb.key ./data/mongodb/mongo1/
cp mongodb.key ./data/mongodb/mongo2/
cp mongodb.key ./data/mongodb/mongo3/

docker-compose up -d

```

## 配置集群

```shell
# 选择第一个容器mongo1，进入mongo 容器
docker exec -it mongo1 bash
 
# 登录mongo
mongosh -u root -p 123456


# 认证
> use admin
> db.auth('root', '123456')
```

## 初始化集群

> <font style="color:rgb(49, 70, 89);">单主机模式部署 </font><font style="color:rgb(49, 70, 89);background-color:rgb(242, 244, 245) !important;">3副本集</font><font style="color:rgb(49, 70, 89);"> 添加节点必须使用宿主机IP+PORT，</font>  
> <font style="color:rgb(49, 70, 89);">使用容器内部IP的情况下代码层面连接到 </font><font style="color:rgb(49, 70, 89);background-color:rgb(242, 244, 245) !important;">mongodb-cluster</font><font style="color:rgb(49, 70, 89);"> 集群，</font>  
> <font style="color:rgb(49, 70, 89);">获取到的集群地址信息为 </font><font style="color:rgb(49, 70, 89);background-color:rgb(242, 244, 245) !important;">docker</font><font style="color:rgb(49, 70, 89);"> 容器内部 </font><font style="color:rgb(49, 70, 89);background-color:rgb(242, 244, 245) !important;">IP</font><font style="color:rgb(49, 70, 89);">，</font>  
> <font style="color:rgb(49, 70, 89);">若业务代码没有部署在 </font><font style="color:rgb(49, 70, 89);background-color:rgb(242, 244, 245) !important;">mongodb</font><font style="color:rgb(49, 70, 89);"> 主机则无法访问</font>

```shell
# 配置文件
> config={_id:"rs0",members:[ 
{_id:0,host:"172.168.14.209:37017"}, 
{_id:1,host:"172.168.14.209:37018"}, 
{_id:2,host:"172.168.14.209:37019"}] 
}

# 初始化集群
> rs.initiate(config)
```

## 配置权重

```shell
> cfg = rs.conf()

# 修改权重
> cfg.members[0].priority=5
> cfg.members[1].priority=3

# 从新配置
> rs.reconfig(cfg)

```

## 验证副本集

```shell
# 切换节点查看同步状态：

> rs.printReplicationInfo()

# 仅当建立了集合后副节点才会进行同步

```

