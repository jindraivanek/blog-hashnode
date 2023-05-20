---
title: "Continuous deployment of SAFE apps on linux server"
datePublished: Sun Oct 21 2018 22:00:00 GMT+0000 (Coordinated Universal Time)
cuid: clhvmb4d600070akzat4wby7u
slug: continuous-deployment-of-safe-apps-on-linux-server
tags: docker, f, deploy, cicd, safe

---

For some time I was looking for a solution for how to do continuous deployment of my [SAFE stack](https://safe-stack.github.io/docs/) projects to the server. My needs are:

* automagically deploy a new version after *push* to master branch
    
* support multiple apps on one server with different URLs (`server.com/app1`, `server.com/app2`)
    

My final solution consists of:

* build as a Docker image
    
* GitLab CI / CD for building image and push it to the registry
    
* Watchtower - docker app for automatically updating to new version of docker images
    
* Nginx - reverse proxy for serving multiple apps
    

## Repository side

### Full build with Docker

Following 2-phase Dockerfile do a full build of SAFE app and run it.

```docker
FROM vbfox/fable-build:stretch AS builder

WORKDIR /build

RUN dotnet tool install fake-cli -g
ENV PATH="${PATH}:/root/.dotnet/tools"

# Package lock files are copied independently and their respective package
# manager are executed after.
#
# This is voluntary as docker will cache images and only re-create them if
# the already-copied files have changed, by doing that as long as no package
# is installed or updated we keep the cached container and don't need to
# re-download.

# Initialize node_modules
COPY package.json yarn.lock ./
RUN yarn install

# Initialize paket packages
COPY paket.dependencies paket.lock ./
COPY .paket .paket
RUN mono .paket/paket.exe restore

# Copy everything else and run the build
COPY . ./
RUN rm -rf deploy
RUN fake run build.fsx --target Bundle

FROM microsoft/dotnet:2.1-aspnetcore-runtime-alpine
WORKDIR /app
COPY --from=builder /build/deploy ./
WORKDIR /app/Server
EXPOSE 8085
ENTRYPOINT ["dotnet", "Server.dll"]
```

### Gitlab CI / CD and docker registry

This `.gitlab-ci.yml` config builds the docker image using the above Dockerfile, and pushes it to the built-in [Docker container registry](https://gitlab.com/help/user/project/container_registry). Docker image address will be `registry.gitlab.com/<user-name>/<repo-name>:<branch-name>`.

```yml
image: docker:stable
services:
  - docker:dind

variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_DRIVER: overlay2
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG

before_script:
  - docker info
  - docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY

build:
  stage: build
  script:
    - docker build -f build.Dockerfile -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
```

## Server side

### Docker run options

On my server, I am using the following command to run the docker image: `docker run -p <port>:8085 --restart unless-stopped --name <name> -d <docker-image>`

For each app, I am using a different port.

### Watchtower

[Watchtower](https://github.com/v2tec/watchtower) is a great docker app that watches over your other running docker images, and when a new version is uploaded, that docker image is updated and restarted.

Using watchtower is simple, just run once:

```bash
docker run -d --name watchtower -v /var/run/docker.sock:/var/run/docker.sock --restart unless-stopped v2tec/watchtower
```

### Multiple applications on one server with Nginx

Each app running on a different port can be mapped to url using [reverse proxy with Nginx](https://www.keycdn.com/support/nginx-reverse-proxy). My configuration may look like this:

`/etc/nginx/sites-enabled/reverse-proxy`

```bash
server {
        listen 80;
        location /app1/ {
             proxy_pass http://localhost:9001/;
        }
        location /api/app1/ {
             proxy_pass http://localhost:9001/api/app1/;
        }
        location /app2/ {
             proxy_pass http://localhost:9002/;
        }
        location /api/app2/ {
             proxy_pass http://localhost:9002/api/app2/;
        }
}
```

I used two entries for each app, one for the main app site, and the second for API that SAFE uses for communication between its client and server part. This way the app works in local and production environments.

## End note

A sample of the working repository with my setup can be found here: [safe-template](https://gitlab.com/jindraivanek/safe-template).

At the moment, I am using this pipeline for 2 projects now:

* [CHMU-air](http://ratatosk.dynu.net/chmu-air)
    
* [Fantomas online](http://ratatosk.dynu.net/fantomas)