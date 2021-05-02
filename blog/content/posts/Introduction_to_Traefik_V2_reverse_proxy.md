---
title: "Introduction to Traefik V2 reverse proxy"
date: 2020-05-17T21:04:27-05:00
description: "Introduction to Traefik V2 reverse proxy"
draft: false
tags: [
	"Docker",
	"Linux",
	"Traefik",
]
---
(Original Post from 2020-04-05_21:45)

![traefik](/images/traefik.png)

Traefik is a very versatile reverse-proxy for Docker containers and has the ability to route both http and tcp based protocols. It also includes an automatic certificate renewal with LetsEncrypt. 

In this introduction Docker and Docker Compose will be used to arrange services to be routed through Traefik. 

This config below defines a secure proxy for Traefik to issue commands to the Docker Engine itself as a security measure and an additional service whoami which is useful for testing http/https connections.

```
mkdir /opt/traefik
cd /opt/traefik
touch traefik.log
touch acme.json
chmod 600 acme.json
docker network create web
vim docker-compose.yml
```
``` docker
#docker-compose.yml
version: "3"
services:
#Docker proxy to obfuscate socket from Traefik for additional security
  dockerproxy:
    container_name: dockerproxy
    environment:
      CONTAINERS: 1
    image: tecnativa/docker-socket-proxy
    networks:
      - traefik
    ports:
      - 2375
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      
#Traefik service declaration
  traefik:
    image: traefik:v2
    container_name: traefik
    ports:
      - 80:80
      - 443:443
    volumes:
      - /etc/localtime/:/etc/localtime:ro
      - /traefik.toml:/traefik.toml:ro
      - /acme.json:/acme.json
      - /traefik.log:/traefik.log
    network:
      - traefik
      - web
    restart: always
    
#Whoami service declaration
  whoami:
    image: containous/whoami
    container_name: whoami
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      - "traefik.http.routers.whoami.rule=Host(`domain.com`)"
      - "traefik.http.routers.whoami.tls=true"
      - "traefik.http.routers.whoami.tls.certresolver=letsencrypt"
    networks:
      - web
    restart: always
```
```
vim traefik.toml
```
```
#Traefik Static Config File
[global]
  checkNewVersion = true
  sendAnonymousUsage = false

[log]
  level = "DEBUG"
  filePath = "./traefik.log"
  format = "toml"

[entryPoints]
  [entryPoints.web]
    address = ":80"
    [entryPoints.web.http]
      [entryPoints.web.http.redirections]
        [entryPoints.web.http.redirections.entryPoint]
	  to = "websecure"
	  scheme = "https"
  [entryPoints.websecure]
    address = ":443"
 
[providers]
  [providers.docker]
    endpoint = "tcp://dockerproxy:2375"
    network = "traefik"
    exposedByDefault = false
    watch = true

[api]
  dashboard = false #Traefik Dashboard is disabled

[tls]
  [tls.options]
    [tls.options.default]
      minVersion = "VersionTLS12"
      sniStrict = true
      cipherSuites = [
          "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
          "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
          "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305",
          "TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305",
          "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256",
          "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
          ]

[certificatesResolvers]
  [certificatesResolvers.letsencrypt.acme]
    email = "mail@domain.com"
    storage = "acme.json"
    #uncomment line below to use LetsEncrypt Staging
    #caServer = "https://acme-staging-v02.api.letsencrypt.org/directory"
    [certificatesResolvers.letsencrypt.acme.httpChallenge]
      entryPoint = "web"
    [certificatesResolvers.letsencrypt.acme.tlsChallenge]
```
