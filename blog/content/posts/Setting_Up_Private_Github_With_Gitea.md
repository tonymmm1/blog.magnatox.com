---
title: "Setting Up Private Github With Gitea"
date: 2020-05-18T15:30:06-05:00
draft: false
description: "Setting Up Private Github With Gitea"
tags: [
	"Docker",
	"Linux",
]
---
![Gitea](/images/gitea.png)

# Setting Up Private Github With Gitea

Setting up a private [Git](https://git-scm.com/) repository is very useful for creating personal projects and having full control of source code creation. [Gitea](https://gitea.io/) is a very useful utility with a clean interface resembling [Github](https://github.com/). If you want to try test out Gitea follow this [Link](https://try.gitea.io/).


## Linux Installation:

1. Make sure [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/) are installed and setup. [Guide](/posts/setting_up_docker/)

2. Create a directory for Gitea.

``mkdir -p /home/(username)/docker/gitea``

3. Configure docker-compose.yml for Gitea server.

```docker
#docker-compose.yml
version: "2"

services:
  gitea:
    image: gitea/gitea:1.11.5	#update and or change if outdated
    container_name: gitea
    ports:
      - 3000:3000
    environment:
      - USER_UID=1000		#change accordingly
      - USER_GID=1000		#change accordingly
      - DB_TYPE=mysql
      - DB_HOST=db:3306
      - DB_NAME=gitea		
      - DB_USER=gitea
      - DB_PASSWD=		#set password to match db 
      - ROOT_URL=		#set url with http/https://
    restart: always
    networks:
      - gitea
#    labels:			#optional Traefik configuration
#      - "traefik.enable=true"
#      - "traefik.http.routers.gitea-web.rule=Host(`(domain)`)"	#replace (domain) with domain
#      - "traefik.http.routers.gitea-web.tls=true"
#      - "traefik.http.routers.gitea-web.tls.certresolver=letsencrypt"
#      - "traefik.http.services.gitea-web.loadbalancer.server.port=3000" 
    volumes:
      - gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    depends_on:
      - db

  db:
    image: mariadb:latest
    restart: always
#    labels:			#optional Traefik configuration
#      - "traefik.enable=false"
    environment:
      - MYSQL_ROOT_PASSWORD=	#set root user password
      - MYSQL_USER=gitea
      - MYSQL_PASSWORD=		#set gitea user password
      - MYSQL_DATABASE=gitea
    networks:
      - gitea
    volumes:
      - db:/var/lib/mysql

networks:
  gitea:
    external: false
volumes:
    gitea:
    db:
```

4. Go to url and configure admin user and site settings.



5. Additional [Gitea configuration](https://docs.gitea.io/en-us/config-cheat-sheet/) is found inside the container volume. Disable registration is highly recommended.

```
sudo vim /var/lib/docker/volumes/(volume name should resemble gitea)/_data/gitea/conf/app.ini
docker-compose down
docker-compose up -d
```

6. Make sure that container is rebuilt after every configuration change.

7. Create regular users with admin user if registration is disabled.

8. Gitea should be good to go and updates can easily be applied by editing gitea image in docker-compose.yml and docker-compose pull. 
