# 🛠️ Deployment Steps

## Create Pod & Verify
```bash
[danny@rhel ~]$ podman pod create --name zabbix-stack -p 8080:8080 -p 10051:10051
c5e42a822897e4ac09504df58bd094b57a99b9c5cfc78ef540c2ba87c94b6775

[danny@rhel web_project]podman pod ls
POD ID        NAME          STATUS      CREATED         INFRA ID      # OF CONTAINERS
c5e42a822897  zabbix-stack  Running     19 minutes ago  507d1cf79642  2
```
## 📦 Prepare Storage (Deploy MySQL - Zabbix DB)
```bash
[danny@rhel web_project]$ cd /home/danny/web_project
[danny@rhel web_project]$ rm -rf ./zabbix_data/mysql
mkdir -p ./zabbix_data/mysql

[danny@rhel web_project]$ podman run -d --name zabbix-db --pod zabbix-stack \
    --env-file ./zabbix.env \
    -v ./zabbix_data/mysql:/var/lib/mysql:Z \
    docker.io/library/mysql:8.0

[danny@rhel web_project]$ podman logs -f zabbix-db
```

Should see something like this when database ready:
```bash
2026-04-01T09:32:50.682924Z 0 [System] [MY-011323] [Server] X Plugin ready for connections. Bind-address: '::' port: 33060, socket: /var/run/mysqld/mysqlx.sock
2026-04-01T09:32:50.683016Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.45'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server - GPL.
```

## 🧠 Deploy Zabbix Server
```bash
[danny@rhel web_project]$ podman run -d --name zabbix-server \
    --pod zabbix-stack \
    --env-file ./zabbix.env \
    -e ZBX_DBHOST=127.0.0.1 \
    -e DB_SERVER_HOST=127.0.0.1 \
    docker.io/zabbix/zabbix-server-mysql:alpine-6.4-latest
d21ae15b568730493be71fbb9e8c88b9399b1d7d6c1ee7e94c430576444bd6c9
```
Verify if server #N and worker started:
```bash
[danny@rhel web_project]$ podman logs -f zabbix-server
  242:20260401:084115.747 server #41 started [availability manager #1]
   244:20260401:084115.748 server #43 started [odbc poller #1]
   208:20260401:084115.765 [1] thread started [preprocessing worker #1]
   208:20260401:084115.766 [3] thread started [preprocessing worker #3]
   208:20260401:084115.766 [2] thread started [preprocessing worker #2]
```

## 🌐 Deploy Zabbix Web Interface
```bash
[danny@rhel web_project]$ podman run -d --name zabbix-web \
    --pod zabbix-stack \
    --env-file ./zabbix.env \
    -e ZBX_SERVER_HOST=127.0.0.1 \
    -e ZBX_DBHOST=127.0.0.1 \
    docker.io/zabbix/zabbix-web-apache-mysql:alpine-6.4-latest
Trying to pull docker.io/zabbix/zabbix-web-apache-mysql:alpine-6.4-latest...
Writing manifest to image destination
af77b15dcc86603c5861fc8f28868cf71d9a671b8849376eea558239bfd18f04
```
## 📊 Verify Running Containers
You should see:
- zabbix-db
- zabbix-server
- zabbix-web
- infra container (Pod)
```bash
[danny@rhel web_project]$ podman ps --pod
CONTAINER ID  IMAGE                                                       COMMAND               CREATED            STATUS            PORTS                                                                  NAMES               POD ID        PODNAME
a717d5c02e73  localhost/my-custom-nginx:v1                                nginx -g daemon o...  About an hour ago  Up About an hour  0.0.0.0:8081->80/tcp                                                   custom-app                        
507d1cf79642                                                                                    28 minutes ago     Up 19 minutes     0.0.0.0:8080->8080/tcp, 0.0.0.0:10051->10051/tcp                       c5e42a822897-infra  c5e42a822897  zabbix-stack
0c00eef583f6  docker.io/library/mysql:8.0                                 mysqld                11 minutes ago     Up 11 minutes     0.0.0.0:8080->8080/tcp, 0.0.0.0:10051->10051/tcp, 3306/tcp, 33060/tcp  zabbix-db           c5e42a822897  zabbix-stack
d21ae15b5687  docker.io/zabbix/zabbix-server-mysql:alpine-6.4-latest      /usr/sbin/zabbix_...  5 minutes ago      Up 5 minutes      0.0.0.0:8080->8080/tcp, 0.0.0.0:10051->10051/tcp                       zabbix-server       c5e42a822897  zabbix-stack
af77b15dcc86  docker.io/zabbix/zabbix-web-apache-mysql:alpine-6.4-latest                        2 minutes ago      Up 2 minutes      0.0.0.0:8080->8080/tcp, 0.0.0.0:10051->10051/tcp, 8443/tcp             zabbix-web          c5e42a822897  zabbix-stack
```
Success setup in Zabbix
<img width="1869" height="651" alt="Screenshot 2026-04-01 175211" src="https://github.com/user-attachments/assets/ee8c1d6c-0d62-4ea4-a4b7-a8eedfa65f7f" />


