## 通过使用Docker去构建TiDB的容器服务

> 基于 7.5.1的版本构建

### 1、拉取镜像【需要拉取三块：pv、titv、tidb 】

```
docker pull pingcap/pd:v7.5.1
docker pull pingcap/tikv:v7.5.1
docker pull pingcap/tidb:v7.5.1
```

### 2、通过 docker-compose进行构建服务

> 使用这个构建文件 需要确保的大前提:
>
> 1、挂在路径是  /data/tidb-data下pd 、tikv、tidb三个文件夹
>
> ​     没有则先执行：mkdir -p /data/tidb-data/{pd,tikv,tidb}
>
> 2、对外暴露的端口是 4000 如果习惯使用 mysql的端口，下面配置文件里 端口映射改成自己想要映射的端口

```yml
version: '3'
services:
  pd:
    image: pingcap/pd:v7.5.1
    container_name: tidb_pd
    cap_add:
      - SYS_ADMIN
    security_opt:
      - seccomp:unconfined
    privileged: true
    ports:
      - "2379:2379"
      - "2380:2380"
    environment:
      - PD_SERVER_NAME=pd
      - INITIAL_CLUSTER=pd=http://pd:2380
      - ETCD_INITIAL_CLUSTER_STATE=new
    volumes:
      - /data/tidb-data/pd:/pd
 
  tikv:
    image: pingcap/tikv:v7.5.1
    container_name: tidb_tikv
    cap_add:
      - SYS_ADMIN
    security_opt:
      - seccomp:unconfined
    privileged: true
    ports:
      - "20160:20160"
    environment:
      - PD_ADDR=pd:2379
    depends_on:
      - pd
    volumes:
      - /data/tidb-data/tikv:/tikv
 
  tidb:
    image: pingcap/tidb:v7.5.1
    container_name: tidb_server
    cap_add:
      - SYS_ADMIN
    security_opt:
      - seccomp:unconfined
    privileged: true
    ports:
      - "4000:4000"
    environment:
      - PATH="bin:$PATH"
      - MYSQL_HOST=0.0.0.0
      - MYSQL_PORT=4000
      - STORE=tikv
      - PATH=bin:$PATH
      - PD_ADDR=pd:2379
    depends_on:
      - tikv
      - pd
    volumes:
      - /data/tidb-data/tidb:/tidb
 
volumes:
  pd-data:
    driver: local
    driver_opts:
      type: none
      device: /data/tidb-data/pd
      o: bind
  tikv-data:
    driver: local
    driver_opts:
      type: none
      device: /data/tidb-data/tikv
      o: bind
  tidb-data:
    driver: local
    driver_opts:
      type: none
      device: /data/tidb-data/tidb
      o: bind
```

启动的话通过  docker-compose up -d 启动就行