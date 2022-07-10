### docker安装qBittorrent

```bash
docker pull linuxserver/qbittorrent

docker create --name=qbittorrent -e PUID=1000 -e PGID=1000 -e TZ=Aisa/Shanghai -e UMASK_SET=022 -e WEBUI_PORT=8079 -p 8999:8999 -p 8999:8999/udp -p 8079:8079 -v /usr/local/qbittorrent/appdata/config:/config -v /usr/local/qbittorrent/downloads:/downloads --restart unless-stopped linuxserver/qbittorrent

docker start qbittorrent
```

如果启动失败，解决方案如下

```bash
进入docker容器中
docker exec -it 容器id /bin/bash

apt update

apt install binutils

strip --remove-section=.note.ABI-tag /usr/lib/x86_64-linux-gnu/libQt5Core.so.5
```

#### 取回本地

安装caddy

```bash
docker pull caddy

docker run -d -p 8078:2015 -v /usr/local/qbittorrent/downloads:/etc/Caddyfile -v $PWD/site:/srv -v /usr/local/caddy/caddy_data:/data -v /usr/local/caddy/caddy_config:/config --name caddy abiosoft/caddy
```

```
docker run -d -p 8078:80 -v /usr/local/caddy/caddy_data:/data -v /usr/local/caddy/caddy_config:/config -v /usr/local/qbittorrent/downloads:/etc/caddy/Caddyfile --name caddy caddy
```

