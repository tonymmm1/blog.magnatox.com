---
title: "Setting Up Private Pastebin With Privatebin"
date: 2020-05-19T13:46:17-05:00
draft: false
tags: [
	"Docker",
	"Linux",
]
---

![Privatebin](/images/privatebin.png)

# Setting Up Private Pastebin with Privatebin

[Pastebin](https://pastebin.com/) is a very useful utility for sharing debug and error mesasges online. [Privatebin](https://privatebin.info/) is a self-hosted alternative that has multiple security options like burn after read mode and uses 256bit AES-GCM for all messages.

## Linux installation:

### Step 1. Make sure [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/) are installed and setup. [Guide](/posts/setting_up_docker/)

### Step 2. Make a directory for privatebin configs.

``mkdir -p /home/(username/docker/privatebin``

### Step 3. Configure docker-compose.yml for privatebin.

```
version: "3"
services:
  privatebin:
    image: privatebin/nginx-fpm-alpine:latest
    ports:
      - 8080:8080
    container_name: privatebin
#    labels:								#optional Traefik config
#      - "traefik.enable=true"						
#      - "traefik.http.routers.privatebin.rule=Host(`(domain)`)"	#replace (domain) with domain
#      - "traefik.http.routers.privatebin.tls=true"
#      - "traefik.http.services.privatebin.loadbalancer.server.port=8080"
#      - "traefik.http.routers.privatebin.tls.certresolver=letsencrypt"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
      - ./privatebin-data:/srv/data
      - ./conf.php:/srv/cfg/conf.php:ro
      - ./php.ini:/etc/php7/php.ini:ro
    networks:
      - privatebin
    restart: always

  privatebin_db:
    image: mariadb:latest
    container_name: privatebin_db
    environment:
      - MYSQL_ROOT_PASSWORD=	#set mysql root password
      - MYSQL_USER=privatebin
      - MYSQL_PASSWORD=		#set mysql user password
      - MYSQL_DATABASE=privatebin
#    labels:			#optional Traefik config
#      - "traefik.enable=false"
    volumes:
      - privatebin_db:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro
    networks:
      - privatebin
    restart: always

volumes:
  privatebin_db:

networks:
  privatebin:
```

### Step 4. Configure conf.php which stores the main configuration settings for Privatebin.

[conf.php](/files/privatebin/conf.php)

- Configure website title on line 7
- Configure file upload on line 19
- Configure size limit on line 32(php.ini requires Step 5)
- Configure forwarded headers on line 129 by uncommenting ;
- Configure MYSQL_PASSWORD on line 165 from docker-compose.yml
- Other configurations maybe changed according to preferences

### Step 5. Configure php.ini if file upload is enabled.

- Configure upload_max_filesize on line 841
- Configure post_max_size on line 689

[php.ini](/files/privatebin/php.ini)

### Step 6. Make directory for metadata and set required permissions.

```
mkdir -p /home/(username)/docker/privatebin/privatebin-data
chown 65534:82 /home/(username)/docker/privatebin/privatebin-data
```

### Step 7. Start Privatebin service and connect to url on port 8080.

``docker-compose up -d``

### Step 8. To update or make changes to service make sure to rebuild.

destroys containers without affecting volumes

``docker-compose down``

pulls new docker images

``docker-compose pull``

starts docker containers in the background

``docker-compose up -d``

