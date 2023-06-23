# 个人服务器


**内容待更新**

<!--more-->

​		由于服务器暂时采用的是云服务器，是一个1核2G的服务器，没有搭建较多的东西，后续购买的 nas 到了之后，会迁移部分服务至 nas 中。

## 服务器采用的方法

​		由于服务器较小，而且搭建的内容仅供个人使用，所以才用了 docker 来进行部署，后续也容易进行备份与迁移；核心使用 [docker](https://docs.docker.com/engine/install/)，采用 [portainer](https://docs.portainer.io/start/install) 来进行在线管理 docker ，使用 [nginxproxymanger](https://nginxproxymanager.com/guide/#quick-setup) 来进行域名配置。

## 服务器所部属的内容

### 服务器相关

- [portainer](https://docs.portainer.io/start/install) docker 管理
- [nginxproxymanager](https://nginxproxymanager.com/guide/#quick-setup) nginx 代理

​		其中比较特殊的 nginxproxymanager 需要将端口映射至服务器上，其他的容器可自行配置 ip 来进行部署，无需映射端口至服务器（一些数据库的端口或者 udp 等特殊端口采用映射更方便），如此一来，容器的服务都可通过 nginxproxymanager 来进行域名配置。

​		由于 portainer 也不对外映射端口，所以在部署之时需要先新建 network，并且同时安装了数据库 postgres 以及 mariadb，还有用于在线访问数据库的 adminer，这样以后如需要用到数据库，可以通过 adminer 来进行操作，部署完成完成之后，可通过服务器81访问 nginxproxymanager 来进行二级域名配置，docker-compose 文件如下

```yaml
version: '3'
services:
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    environment:
      - LANG=C.UTF-8
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer:/data
    networks:
      internel:
        ipv4_address: 172.19.0.6
  postgresql:
    container_name: postgresql
    image: postgres:14-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres_password
      - LANG=C.UTF-8
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
      - ./postgresql:/var/lib/postgresql/data
    ports:
      - 5432:5432
    networks:
      internel:
        ipv4_address: 172.19.0.7
  mariadb:
    container_name: mariadb
    image: mariadb
    restart: unless-stopped
    environment:
      - MARIADB_ROOT_PASSWORD=root_password
      - MARIADB_DATABASE=proxy
      - MARIADB_USER=proxy
      - MARIADB_PASSWORD=proxy_password
      - LANG=C.UTF-8
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
      - ./mariadb:/var/lib/mysql
    ports:
      - 3306:3306
    networks:
      internel:
        ipv4_address: 172.19.0.8
  proxy:
    container_name: proxy
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    environment:
      - DB_MYSQL_HOST=172.19.0.8
      - DB_MYSQL_PORT=3306
      - DB_MYSQL_USER=proxy
      - DB_MYSQL_PASSWORD=proxy_password
      - DB_MYSQL_NAME=proxy
      - LANG=C.UTF-8
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
      - ./proxy/data:/data
      - ./proxy/letsencrypt:/etc/letsencrypt
    networks:
      internel:
        ipv4_address: 172.19.0.9
  adminer:
    container_name: adminer
    image: adminer
    restart: unless-stopped
    environment:
      - LANG=C.UTF-8
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    networks:
      internel:
        ipv4_address: 172.19.0.10
networks:
  internel:
    driver: bridge
    ipam:
      config:
        - subnet: 172.19.0.0/16
```

### 开发相关

- [gitea](https://docs.gitea.io/en-us/install-with-docker/) git服务器，英文文档为最新的
- [drone](https://docs.drone.io/) 自动打包及部署
- [frp](https://gofrp.org/docs/) 内网穿透，采用第三方或者自己手写dockerfile

​		在部署之前可以通过之前部署的 adminer 来新建 gitea 和 drone 所需要的数据库，然后在部署时填写上对应的 bridge 配置的 ip 以及数据库名和密码等相关参数，而 drone 则需要在 gitea 部署之后再部署，至于原因可以查看 drone 官方文档，frp 则是为了本地开发时所需要。

### 博客相关

- [hugo](https://gohugo.io/documentation/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod)（主题）
- [nginx](https://hub.docker.com/_/nginx/) docker
- [walinejs](https://waline.js.org/guide/get-started.html) 评论
- [umami](https://umami.is/docs/getting-started) 访问统计
- [minio](https://min.io/docs/minio/container/index.html) 图床

​		其次在采用 docker nginx 进行发布是通过挂载 `/var/run/docker.sock` 至 docker 镜像中，挂载需要在drone中配置项目为 trusted，这样在容器中使用 docker 命令即可控制服务器的 docker，然后可采用 `docker cp ./ nginx:/usr/share/nginx/html` 来进行发布，具体 .drone.yml 如下

```yaml
kind: pipeline

steps:
  - name: build
    image: alpine/git
    commands:
      - git submodule set-url themes/PaperMod https://github.com/adityatelange/hugo-PaperMod.git
      - git submodule init
      - git submodule update --recursive --remote
      - sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories
      - apk update
      - apk add hugo
      - hugo config check
      - hugo

  - name: publish
    image: docker
    volumes:
      - name: dockersock
        path: /var/run
    commands:
      - cd public
      - sleep 5
      - docker cp ./ nginx:/usr/share/nginx/html

volumes:
  - name: dockersock
    host:
      path: /var/run

```

## 结语

由于服务器较小，后续肯定要进行修改以及迁移等操作。待更新。。。

