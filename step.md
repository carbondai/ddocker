1. 运行数据库容器rethinkdb
`docker run -ti -d --restart=always --name shipyard-rethinkdb rethinkdb`

2. 运行后台服务器发现程序etcd
```bash
docker run -ti -d -p 4001:4001 -p 7001:7001 \
    --restart=always \
    --name shipyard-discovery \
    microbox/etcd -name discovery
```

3. 运行docker代理服务proxy
```bash
docker run -ti -d -p 2375:2375 \
    --hostname=$HOSTNAME \
    --restart=always \
    --name shipyard-proxy \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -e PORT=2375 \
    shipyard/docker-proxy:latest
```

4. 运行swarm manager
```bash
docker run -ti -d \
    --restart=always \
    --name shipyard-swarm-manager \
    swarm:latest \
    manage --host tcp://0.0.0.0:3375 etcd://<IP-OF-HOST>:4001
```

5. 运行swarm agent
```bash
docker run -ti -d \
    --restart=always \
    --name shipyard-swarm-agent \
    swarm:latest \
    join --addr <ip-of-host>:2375 etcd://<ip-of-host>:4001
```

6. 运行shipyard controller
```bash
docker run -ti -d \
    --restart=always \
    --name shipyard-controller \
    --link shipyard-rethinkdb:rethinkdb \
    --link shipyard-swarm-manager:swarm \
    -p 8080:8080 \
    shipyard/shipyard:latest \
    server \
    -d tcp://swarm:3375
```

#### 一些docker命令
----------------
docker save -o ubuntu.tar ubuntu:14.04导出本地镜像
docker load < ubuntu.tar 载入镜像文件

#### 制作自己的docker镜像


