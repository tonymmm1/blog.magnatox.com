---
title: "Nextcloud Docker with Traefik V2"
date: 2020-05-17T21:35:13-05:00
draft: false
---
(Originally posted 2020-04-05_21:57)

![nextcloud](/images/nextcloud.png)

This is an example config of running Nextcloud as Docker container under Traefik V2. 

```
mkdir /opt/nextcloud
cd /opt/nextcloud
vim docker-compose.yml
```
```
version: '3'
services:
  db:
   image: mariadb:latest
   command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
   restart: always
   volumes:
     - db:/var/lib/mysql
   environment:
     - MYSQL_ROOT_PASSWORD=
     - MYSQL_PASSWORD=
     - MYSQL_DATABASE=nextcloud
     - MYSQL_USER=nextcloud
   labels:
     - "traefik.enable=false"
   networks:
     - nextcloud

  nextcloud:
    image: nextcloud:apache
    container_name: nextcloud
    restart: always
    volumes:
      - nextcloud:/var/www/html
    environment:
      - VIRTUAL_HOST=domain.com
      - LETSENCRYPT_HOST=domain.com
      - MYSQL_PASSWORD=
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      - "traefik.http.routers.nextcloud.rule=Host(`domain.com`)"
      - "traefik.http.routers.nextcloud.tls=true"
      - "traefik.http.routers.nextcloud.tls.certresolver=letsencrypt"
      - "traefik.http.middlewares.nextcloud.redirectregex.regex=https://(.*)/.well-known/(card|cal)dav"
      - "traefik.http.middlewares.nextcloud.redirectregex.replacement=https://$$1/remote.php/dav/"
      - "traefik.http.middlewares.nextcloud.redirectregex.permanent=true"
      - "traefik.http.middlewares.nextcloud.headers.customFrameOptionsValue=SAMEORIGIN"
    depends_on:
      - db
    networks:
      - web
      - nextcloud
  cron:
   image: nextcloud:apache
   restart: always
   labels:
     - "traefik.enable=false"
   volumes:
     - nextcloud:/var/www/html
   entrypoint: /cron.sh
   depends_on:
     - db
   
networks:
  web:
    external: true
  nextcloud:
volumes:
  nextcloud:
    driver: local
  db:
    driver: local
```
```
docker-compose up -d
```
