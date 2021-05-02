---
title: "Setting Up Drone CI"
date: 2020-05-18T14:39:34-05:00
draft: false
description: "Setting Up Drone CI"
tags: [
	"CI/CD",
	"Docker",
	"Linux",
]
---

![Drone](/images/drone.png)

# Setting Up Drone CI

[Drone CI](https://drone.io/) is an open-source and self-hostable automatic building system. It allows you to focus on programming by automatically building your code and also even publishing release binaries. There are also many available [plugins](http://plugins.drone.io/) that allows it to push binaries to a server over rsync, to publish latest release to a git repo, and even notifications to SLACK.

There are two Drone services:
- [Drone server](https://docs.drone.io/server/overview/): runs website and is in charge of webhooks
- [Drone runner](https://docs.drone.io/runner/overview/): receives commands

## Linux installation:

1. Make sure [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/) are installed and setup. [Guide](/posts/setting_up_docker/)

2. Create a directory for drone

``mkdir -p /home/(username)/docker/drone``

3. Read setup guides

- [Gitea](https://docs.drone.io/server/provider/gitea/)
- [Github](https://docs.drone.io/server/provider/github/)
- [Other](https://docs.drone.io/server/overview/)

4. Configure docker-compose.yml for Drone server.

```docker
#docker-compose.yml
version: "3"
services:
  drone:
    container_name: drone
    image: drone/drone:latest
    environment:
      - DRONE_GITEA_SERVER=										#change to DRONE_GITset to url of gitea/gitea
      - DRONE_GITEA_CLIENT_ID=                                  #set to client id from github/gitea
      - DRONE_GITEA_CLIENT_SECRET=								#set to client secret from github/gitea
      - DRONE_RPC_SECRET=										#generate a key $openssl rand -base64 (length)	
      - DRONE_SERVER_HOST=										#set domain name
      - DRONE_SERVER_PROTO=https
      - DRONE_USER_CREATE=username:(username),admin:true 		#create user that matches git/gitea username
      - DRONE_USER_FILTER=(username),admin						#create user that matches git/gitea username
      - DRONE_AGENTS_ENABLED=true
      - DRONE_DATABASE_DRIVER=postgres
      - DRONE_DATABASE_DATASOURCE=postgres://drone:(password)@drone_db:5432/drone?sslmode=disable	#remote (password) and insert password that matches POSTGRES_PASSWORD
#    labels:								                    #optional Traefik config
#      - "traefik.enable=true"									#remote (domain) and insert url
#      - "traefik.http.routers.drone.rule=Host(`(domain)`)"
#      - "traefik.http.routers.drone.tls=true"								
#      - "traefik.http.routers.drone.tls.certresolver=letsencrypt"
    restart: always
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    networks:
      - web
      - drone

  drone_db:
    container_name: drone_db
    image: postgres:latest
    environment:
      - POSTGRES_USER=drone
      - POSTGRES_PASSWORD=          #set password
      - POSTGRES_DB=drone
#    labels:			            #optional Traefik config
#      - "traefik.enable=false"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - drone_db:/var/lib/postgresql/data
    networks:
      - drone
    restart: always

volumes:
  drone_db:

networks:
  drone: 
```

5. Configure docker-compose.yml for a Drone runner on same or different host.

```docker
#docker-compose.yml
version: "3"
services: 
  drone-runner:
    image: drone/drone-runner-docker:latest
    container_name: drone-runner
    ports: 
      - 3000:3000
    environment:
      - DRONE_RPC_PROTO=https					#change to http/https
      - DRONE_RPC_HOST=drone.magnatox.com			#drone url
      - DRONE_RPC_SECRET=4862fe3bd5703c5686f1586996a39f00	#copy secret generated from previous step
      - DRONE_RUNNER_CAPACITY=2					#divides total host resources into n number of jobs (2 is a good starting point for 2+ core systems)
      - DRONE_RUNNER_NAME=					#set drone runner url
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock		#give drone runner access to docker 
    restart: always
```

6. On Drone server system start server.

``docker-compose up -d``

7. On Drone runner system start runner.

``docker-compose up -d``

8. Make sure to follow the Guides and configure website redirect.

9. Login to Drone server url and you will be prompted with a authentication request.

10. Setup will be complete with all repositories displayed.

11. Read [Quickstart](https://docs.drone.io/quickstart/) to learn more about creating pipelines in .drone.yml files. 
