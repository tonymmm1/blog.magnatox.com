---
title: "Setting Up Docker"
date: 2020-05-18T14:22:41-05:00
draft: false
description: "Setting up Docker"
tags: [
	"Docker",
	"Linux",
]
---

![docker](/images/docker.png)


Setting up Docker 

Docker is a versatile container engine that allows application separation from the host machine and also scalability not possible with more heavy virtual machines.

1. To start installing Docker first navigate to [Docker](https://www.docker.com/get-started) and install for corresponding platform. 

2. Next make sure to install Docker Compose by navigating to [Docker Compose](https://docs.docker.com/compose/install/) and install for corresponding platform.

3. Make sure to start and enable Docker service on boot.

```
systemctl enable docker
systemctl start docker
```

4. For Linux systems make sure to grant user permission to Docker.

```
sudo usermod -aG docker (username)
docker info
```
