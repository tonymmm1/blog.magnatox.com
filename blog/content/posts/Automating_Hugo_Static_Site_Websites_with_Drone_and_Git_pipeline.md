---
title: "Automating Hugo Static Site Websites With Drone and Git Pipeline"
date: 2020-05-19T15:11:31-05:00
draft: false
tags: [
	"CI/CD",
	"Docker",
	"Linux",
]
---

![](/images/gitea_drone_hugo.png)

# Automating Hugo Static Site Websites With Drone and Git Pipeline

This post will demonstrate how to build an automated build pipeline that will automatically rebuild website based on git changes. Based on what is used to maintain this site. This will be a mostly Linux and Docker based post since the applications are hosted as Docker containers but this can easily be applied to virtual and physical hosts as well. 

Pipeline Diagram: 

[Git] -> [Drone] -> [Drone runner] -> [Remote server] -> [Nginx Service] -> [Site]

## Requirements:

- [Docker](posts/setting_up_docker/)
- [Drone CI](/posts/setting_up_drone_ci/)
- [Gitea](posts/setting_up_private_github_with_gitea/)/[Github](https://github.com)/[Others](https://docs.drone.io/)
- [Hugo](https://gohugo.io/) 


## Installation and setup:

### Step 1: Setup a new Git repository on a platform that is [supported](https://docs.drone.io/).
[Reference](https://git.magnatox.com/tonymmm1/blog.magnatox.com)
![Git repo](/images/screenshot_2020-05-19_15:23:30-01.png)

### Step 2: Clone Git repository locally.

``git clone url``

### Step 3: Create a [Hugo site](https://gohugo.io/getting-started/quick-start/) within the repo and choose a directory name.

### Step 4: Commit site and any files then push to Git repo. 

```
git add .
git commit -m "initial Hugo site files" 
git push
```

### Step 5: Make sure that [Drone](https://docs.drone.io/) is configured with Git repo and that webhooks have been configured.

### Step 6: Configuring Drone Pipeline with .drone.yml.

[Reference](https://git.magnatox.com/tonymmm1/blog.magnatox.com/src/branch/master/.drone.yml)

```
kind: pipeline
type: docker
name: 			#set name

steps:
  - name: submodules
    image: alpine/git
    commands:
      - git submodule update --init --recursive --remote
  - name: build-drafts	#development build non-master branches
    image: plugins/hugo
    settings:
      hugo_version: 0.70.0
      url: 		#set site url
      source: blog/
      output: output/
      buildDrafts: true	#builds draft posts
      validate: true
    when:
      branch:
        exclude:
          - master
  - name: build		#production build master only
    image: plugins/hugo
    settings:
      hugo_version: 0.70.0
      url: 		#set site url
      source: blog/
      output: output/
      buildDrafts: false #builds non-draft posts 
      validate: true
    when:
      branch:
        - master
      event:		
        - push
        - rollback
        - tag
        - promote
  - name: deploy	#production deploy
    image: drillster/drone-rsync
    target:
    settings:	#all secrets below are defined Drone
      hosts:
        from_secret: remote_hosts	
      source: blog/output
      recursive: true
      user:	#rsync user
        from_secret: remote_user
      key:	#rsync private user key
        from_secret: remote_key
      port:	#rsync port
        from_secret: remote_port
      target:	#rsync file directory
        from_secret: remote_target
      script:	#rsync commands after initial copy
        - cd /home/drone/docker/hugo/blog #remote host docker-compose directory
        - docker-compose down		
        - docker-compose up -d
    when:
      branch:
        - master
      event:
        - push
        - rollback
        - tag
        - promote
```

### Step 7: Commit .drone.yml to Git repo and make sure repo is activated in Drone.

![](/images/screenshot_2020-05-19_15:48:53-01.png)

### Step 8: Configure secrets with Drone matching with .drone.yml values.

![](/images/screenshot_2020-05-19_15:51:31-01.png)

### Step 9: Configure Hugo to run on remote host.

#### Step 9.1: Create directory.
```
mkdir -p /home/(username)/docker/hugo
```

#### Step 9.2: Make sure to use the directory just created for secret remote_target.

#### Step 9.3: Configure docker-compose.yml.

```
version: "3.7"
services:
  blog:
    image: nginx:alpine
    container_name: blog
    ports:
      - 8080:80	#exposes on external 8080 port
#    labels:	#optional Traefik config
#      - "traefik.enable=true"
#      - "traefik.http.routers.ghost.rule=Host(`(domain)`)"	#replace (domain) with domain
#      - "traefik.http.routers.ghost.tls=true"
#      - "traefik.http.routers.ghost.tls.certresolver=letsencrypt"
#      - "traefik.http.services.ghost.loadbalancer.server.port=80"
    volumes:
      - ./output:/usr/share/nginx/html:ro 
    environment:
      - DOMAIN=		#set domain
    restart: always
```

#### Step 9.4: Setup ssh keys and create a private pair for Drone to use.

``echo "public key of drone" >> ~/.ssh/authorized_keys``

### Step 10: Trigger webhook even in git or push a commit to Git repo.

![](/images/screenshot_2020-05-19_16:03:06-01.png)

![](/images/screenshot_2020-05-19_16:05:01-01.png)

Remote host container status:

![](/images/screenshot_2020-05-19_16:08:46-01.png)

### Step 11: If build completed successfully there should be a green check otherwise check secrets and or ssh key for issues.
