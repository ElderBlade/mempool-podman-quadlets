# mempool-podman-quadlets
You can deploy [mempool](https://github.com/mempool/mempool) on Podman using quadlets. This particular setup uses **Bitcoin Core** + **Electrum Server**.

Provided below are sample rootless quadlet container files that can be placed in `/etc/containers/systemd/users/${ID}` where ID is the uid of whatever your rootless user is or in `$HOME/.config/containers/systemd`. See podman [documentation](https://docs.podman.io/en/stable/markdown/podman-systemd.unit.5.html) for details. 

# Quadlet Files
It is recomended to run these rootless for better security.

## Pod
mempool.pod
```bash
[Pod]
Network=mempool.network
PodName=Mempool
PublishPort=80:8080 # use port 80 or bind to a different one on the system you will access the web app

[Service]
Restart=unless-stopped

[Install]
WantedBy=multi-user.target default.target
```
## Containers
mempool-web-service.container
```bash
[Unit]
Description=Mempool Web App
Requires=mempool-db-service.service
After=mempool-db-service.service

[Container]
Pod=mempool.pod
ContainerName=web
Image=docker.io/mempool/frontend:latest

User=1000:1000

Environment=FRONTEND_HTTP_PORT=8080
Environment=BACKEND_MAINNET_HTTP_HOST=api

Exec=./wait-for db:3306 --timeout=720 -- nginx -g \'daemon off\;\'

[Service]
Restart=unless-stopped

[Install]
WantedBy=multi-user.target default.target
```
mempool-api-service.container
```bash
[Unit]
Description=Mempool API
Requires=mempool-db-service.service
After=mempool-db-service.service

[Container]
Pod=mempool.pod
ContainerName=api
Image=docker.io/mempool/backend:latest

Environment=ELECTRUM_HOST=127.0.0.1
Environment=ELECTRUM_PORT=50002
Environment=ELECTRUM_TLS_ENABLED=true
Environment=MEMPOOL_BACKEND=electrum
Environment=CORE_RPC_HOST=127.0.0.1
Environment=CORE_RPC_PORT=8332
Environment=CORE_RPC_USERNAME=customuser
Environment=CORE_RPC_PASSWORD=custompassword
Environment=DATABASE_ENABLED=true
Environment=DATABASE_HOST=db
Environment=DATABASE_USERNAME=mempool
Environment=DATABASE_PASSWORD=mempool
Environment=STATISTICS_ENABLED=true

Exec=./wait-for-it.sh db:3306 --timeout=720 --strict -- ./start.sh
Volume=mempool-api.volume:/backend/cache

[Service]
Restart=unless-stopped

[Install]
WantedBy=multi-user.target default.target
```
mempool-db-service.container
```bash
[Unit]
Description=Mempool MariaDB container

[Container]
Pod=mempool.pod
ContainerName=db
Image=docker.io/mariadb:10.5.21
Environment=MYSQL_ROOT_PASSWORD=admin
Environment=MYSQL_USER=mempool
Environment=MYSQL_PASSWORD=mempool
Environment=MYSQL_DATABASE=mempool
Volume=mempool-db.volume:/var/lib/mysql

[Service]
Restart=unless-stopped
TimeoutStartSec=1m

[Install]
WantedBy=multi-user.target default.target
```

## Volumes
mempool-api.volume
```bash
[Unit]
Description=Mempool API Volume

[Volume]
```
mempool-db.volume
```bash
[Unit]
Description=Mempool MariaDB Volume

[Volume]
```
## Network
mempool.network
```bash
[Unit]
Description=Mempool Network
After=network-online.target

[Network]
Subnet=172.18.0.0/24
Gateway=172.18.0.1
DNS=1.1.1.1

[Install]
WantedBy=default.target
```
## Run your quadlet
Generate the systemd files:
```bash
systemctl --user daemon-reload
```

You can verify the files were generated correctly by checking the status of one of the containers:
```bash
systemctl --user status mempool-api-service.service
```

Start your mempool pod:
```bash
systemctl --user start mempool-pod
```

Mempool should now be available at `http://127.0.0.1:80`
