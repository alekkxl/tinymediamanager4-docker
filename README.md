# tinymediamanager-docker

带有GUI接口的 TinyMediaManager V4.0.6 CN & Cracker Docker.

## 数据迁移
将原镜像`/config/data` 文件夹复制出来即可。  
因为 `jlesage/baseimage-gui`镜像只有 x86_64 （后续会推出多重构架），目前只能在 x86_64 主机下运行，树莓派不支持。

镜像映射: 
- WEB 5800
- VNC 5900
- 配置和数据目录 /config
- 媒体目录 /media

命令行:

```bash
docker run -d --name=tinymediamanager \
-v /config:/config \
-v /media:/media \
-e GROUP_ID=0 -e USER_ID=0 -e TZ=Asia/Shanghai \
--add-host=api.themoviedb.org:13.224.161.90 \
--add-host=image.tmdb.org:104.16.61.155 \
--add-host=api.themoviedb.org:13.35.67.86 \
--add-host=www.themoviedb.org:54.192.151.79 \
-p 5800:5800 \
-p 5900:5900 \
wakongka/tinymediamanager-docker
```
docker-compose.yml:
```bash
version: "3"
services:
  tmm:
    image: wakongka/tinymediamanager-docker:latest
    container_name: tmm
    volumes:
            - /config:/config
            - /media:/media
    ports:
      - "5800:5800"
	  - "5900:5900"
    extra_hosts:
      - "api.themoviedb.org:13.224.161.90"
	  - "image.tmdb.org:104.16.61.155"
	  - "api.themoviedb.org:13.35.67.86"
	  - "www.themoviedb.org:54.192.151.79"
    environment:
      GROUP_ID: "0"
      USER_ID: "0"
      VNC_PASSWORD: "tmmadmin"
      TZ: "Asia/Shanghai"
    networks:
            tmmnet:
networks:
    tmmnet:

```
访问 `http://your-host-ip:5800` TinyMediaManager GUI.

## 环境变量

命令行 `-e` `<VARIABLE_NAME>=<VALUE>`  
或者docker-compose.yml
```    
environment:
      - USER_ID=1000
	  ...
```

| Variable       | Description                                  | Default |
|----------------|----------------------------------------------|---------|
|`USER_ID`| 用户ID | `1000` |
|`GROUP_ID`| 组ID | `1000` |
|`SUP_GROUP_IDS`| 补充组ID | (unset) |
|`UMASK`| 权限计算器 http://wintelguy.com/umask-calc.pl | (unset) |
|`TZ`| 时区 | `Etc/UTC` |
|`KEEP_APP_RUNNING`| 当设置为1，如果应用程序崩溃或用户退出，应用程序将自动重新启动。 | `0` |
|`APP_NICENESS`| 应用应运行的优先级。-20 的值是最高优先级，19 是最低优先级。默认情况下，不设置比较好 . | (unset) |
|`CLEAN_TMP_DIR`| 设置为1，/tmp目录中的所有文件将在容器启动期间删除。 | `1` |
|`DISPLAY_WIDTH`| 应用程序窗口的宽度（像素）。 | `1280` |
|`DISPLAY_HEIGHT`| 应用程序窗口的高度（像素）。 | `768` |
|`SECURE_CONNECTION`| 设置为1时，加密访问应用程序的GUI（通过 Web 浏览器或 VNC 客户端）。一般还是使用Nginx https反代比较好建议不设置 | `0` |
|`VNC_PASSWORD`| 连接到应用程序的GUI所需的密码，密码仅限于 8 个字符。 | (unset) |
|`X11VNC_EXTRA_OPTS`| 额外的选项传递到在Docker容器中运行的x11vnc服务器。警告：对于高级用户。除非你知道自己在做什么，否则不要使用。 | (unset) |
|`ENABLE_CJK_FONT`| 设置为1时，将安装开源计算机字体。此字体包含大量中文/日文/韩文字符。（本镜像已集成中文） | `0` |

## 数据目录


| 路径  | 权限 | 介绍 |
|-----------------|-------------|-------------|
|`/config`| rw | 这是应用程序存储其配置、日志和任何需要持久性的文件的位置。 |
|`/media`| rw | 这是存储媒体文件的地方。 |

## 端口

| 端口 |介绍 |
|------|-------------|
| 5800 | 用于通过 Web 界面访问应用程序的 GUI 的端口。 |
| 5900 | 用于通过 VNC 协议访问应用程序的 GUI 的端口。如果没有使用VNC客户端，则可选。 |

## 证书

当`SECURE_CONNECTION`设置时，需要以下证书文件。  
默认情况下，生成并使用自签名证书。所有文件都有PEM编码，x509证书。

| 容器路径                  | 目的                    | 内容 |
|---------------------------------|----------------------------|---------|
|`/config/certs/vnc-server.pem`   |VNC 连接加密。  |VNC服务器的私人密钥和证书，捆绑任何根和中间证书。|
|`/config/certs/web-privkey.pem`  |HTTP连接加密。|网络服务器的私人密钥。|
|`/config/certs/web-fullchain.pem`|HTTP连接加密。|Web 服务器的证书，与任何根证书和中间证书捆绑在一起。|

**注意**: 为避免浏览器或 VNC 客户端发出任何证书有效性警告/错误，请务必提供您自己的有效证书。

**注意**: 监控证书文件，检测到更改时请自动重启。

## 命令行访问

```
docker exec -ti CONTAINER sh
```

## NGINX配置

以下部分包含需要添加的 NGINX 配置，以便向此容器反向代理。

反向代理服务器可以根据主机名或 URL 路径路由 HTTP 请求。

### 基于主机名
例如：`tinymediamanager.domain.tld`

```
map $http_upgrade $connection_upgrade {
	default upgrade;
	''      close;
}

upstream tinymediamanager {
	# If the reverse proxy server is not running on the same machine as the
	# Docker container, use the IP of the Docker host here.
	# Make sure to adjust the port according to how port 5800 of the
	# container has been mapped on the host.
	server 127.0.0.1:5800;
}

server {
	[...]

	server_name tinymediamanager.domain.tld;

	location / {
	        proxy_pass http://tinymediamanager;
	}

	location /websockify {
		proxy_pass http://tinymediamanager;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection $connection_upgrade;
		proxy_read_timeout 86400;
	}
}

```

### 基于网址

例如：`tinymediamanager.domain.tld/tinymediamanager`

```
map $http_upgrade $connection_upgrade {
	default upgrade;
	''      close;
}

upstream tinymediamanager {
	# If the reverse proxy server is not running on the same machine as the
	# Docker container, use the IP of the Docker host here.
	# Make sure to adjust the port according to how port 5800 of the
	# container has been mapped on the host.
	server 127.0.0.1:5800;
}

server {
	[...]

	location = /tinymediamanager {return 301 $scheme://$http_host/tinymediamanager/;}
	location /tinymediamanager/ {
		proxy_pass http://tinymediamanager/;
		location /tinymediamanager/websockify {
			proxy_pass http://tinymediamanager/websockify/;
			proxy_http_version 1.1;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection $connection_upgrade;
			proxy_read_timeout 86400;
		}
	}
}

```

