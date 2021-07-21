My notes from the book "Docker In Action"

# Basics 

```
// start a nginx container named web in detached mode (not attached to current terminal). For daemon/services.
$docker run --detach --name web nginx:latest

// attach an interactive terminal to a running container and run a command (here the bash shell)
$docker exec -it web /bin/bash
>>> root@49838061f301:/# ls / ..

// list running containers, -a for all states (created/running/paused/restarting/removing/exited)
$docker ps 

// tail logs and dump to stdout
$docker logs web -f

// print metadata json
$docker inspect web

//TIP: Running Containers can be renamed
$docker rename web old-web

$docker stop/start web

// remove containers
$docker rm web
// auto cleanup short-lived containers i.e. when it exits
$docker run --rm busybox:1.29 date
```

##### network linking 
creates unidirectional links - injects IPs into dependent containers. For 2 way, use docker networks.
```
MAILER_CID=$(docker run -d dockerinaction/ch2_mailer)
WEB_CID=$(docker run -d nginx)

AGENT_CID=$(docker run -d \    --link $WEB_CID:insideweb \    --link $MAILER_CID:insidemailer \    dockerinaction/ch2_agent)
```

###### Durability
- you can specify a restart flag to attempt restarting the container if it exits/fails. Docker uses exp backoff strategy
```
$docker run -d --name backoff --restart always busybox:1.29 date
$docker logs backoff
Tue Jul 20 17:38:59 UTC 2021
Tue Jul 20 17:39:01 UTC 2021
Tue Jul 20 17:39:05 UTC 2021
Tue Jul 20 17:39:12 UTC 2021
```
- more advanced solution is to use **init systems** which launch & maintain other processes (restarting them if reqd). Used when container is running multiple processes or uses child processes. e.g. an entire LAMP stack in one container



## Isolation features of containers
- multiple PID namespaces for processes.
- Env agnostic 
  - Read only filesystems 
  - Env var injection
  - Volumes
- Union file systems: Image is made up of readonly layers. UnionFS uses a **copy-on-write** pattern. Only the bottom most layer is writable.

A full fledged example 

- attach container to a network
- set env variables to pass in configuration
- specify a docker volume to be mounted at a specific location in the container (data that outlives the container)
```
docker run -d ^
    --name my-database ^
    --network my-network ^
    -e POSTGRES_DB=test ^
    -e POSTGRES_USER=test-user ^
    -e POSTGRES_PASSWORD=test-pwd ^
    -e PGDATA=/var/lib/postgresql/data/pgdata ^
    -v D:\dockervolumes\test-data:/var/lib/postgresql/data ^
    lbm-postgres:13.2
```

# Identify / Find / Install
### Naming

- docker.io/dockerinaction/ch3_hello i.e. [registy host:port]/[user or org]/[name]:[tag]
- Each image can be tagged with multiple tags.
- Docker hub is the default registry, where images are looked up if explicit registry is not specified

```
docker pull quay.io/me/anImage:latest

// list all images
docker image ls

// remove image
docker rmi some-image
```

### Export / import

```
// current folder has a dockerFile
docker build -t [image:tag] . 
docker save -o MyImage.tar [image name]
docker load -i MyImage.tar
```

# Storage
Linux unifies all storage into a single tree. Storage devices such as disk/usb are attached to specific locations on the tree called mount points {location + properties e.g. readonly + data source }

### Docker volumes
named FS trees managed by Docker. Can be backed by host FS or cloud storage.

```
docker volume create 
  --driver local
  --label purpose=cassandra
  cass-shared

docker run 
  --volume cass-shared:/var/lib/cassandra/data
  cassandra:2.2
```

Anonymous volumes: auto deleted when the container is removed or via `docker volume rm`
```
docker run --name fowler \
--mount type=volume,dst=/library/PoEAA \
--mount type=bind,src=/tmp,dst=/library/DSL \ 
alpine:latest \
echo "Fowler collection created."
```

### Bind mounts
attach a specified location on the host FS to a point on the container file tree.

Usecase: app in container needs access to a config file/dir on the host FS OR app generates files that needs to be accessible outside of the container

```
CONF_SRC=~/example.conf \
CONF_DST=/etc/nginx/conf.d/default.conf \
docker run -d \
  --mount type=bind,src=${CONF_SRC},dst=${CONF_DST},readonly=true \
  -p 80:80 \
  nginx
```

### In-Memory storage
For sensitive info like keys/passwords - they should never be part of the image. 

```
docker run 
  --mount type=tmpfs,dst=/tmp
  --entrypoint mount
  alpine:latest -v

```
# Networking


###  Building Images

A Dockerfile is a text file with instructions for building the image. A new layer is added to the image, after each step.

TIP: You should combine instructions whereever possible
e.g.

``` RUN apt-get update && apt-get install xyz ```

```
docker image build --tag myimage:mytag .
```

- .  signfies current folder has the Dockerfile
- -f can be used to specify the file explicitly

.dockerignore contains files that should be ignored during builds

#### Instructions

```
FROM debian:buster-20190910
LABEL maintainer="dia@allingeek.com"
RUN groupadd -r -g 2200 example && \
    useradd -rM -g example -u 2200 example
ENV APPROOT="/app" \
    APP="mailer.sh" \
        VERSION="0.6"
LABEL base.name="Mailer Archetype" \
      base.version="${VERSION}"
WORKDIR $APPROOT
ADD . $APPROOT
ENTRYPOINT ["/app/mailer.sh"]
EXPOSE 33333
```

* FROM image:tag - use this image as the base
* LABEL maintainer="email" - author
* RUN command - run these commands e.g. apt-get install or update
* ENTRYPOINT ["cmd"] 
  * shell form - will be executed as `/bin/sh -c 'cmd'`. This will override any CMD / command line args
  * exec form is a string array ['cmd' 'arg1'...]
* CMD - represents args[] for ENTRYPOINT (default /bin/sh)
* LABEL : key-value pairs of metadata about the image
* WORKDIR : set working dir to value
* EXPOSE : creates layer to open port XXXX
* USER user:group - sets the default user. Switched from root - cannot install.
* ADD - file to container FS. works with remote URLs and extracts input archives
* COPY [args...]  [dest] - from build FS to container FS. Ownership is set to root.
* VOLUME - creates the location in FS and update metadata. Better to set this on command line
* ONBUILD instructions - specified in based image. Executed between FROM [base-image] and next instruction during build of derived image.
* ARG Key=value - this can be overridden with --build-arg key=value during build.

#### Multi stage builds
e.g. Reduces size of final image by seperating build from runtime deps

```
FROM golang:1-alpine as builder

FROM scratch as runtime

COPY --from=builder /go/bin/proj /proj
ENTRYPOINT ["/proj"]
```

Usually entrypoints are scripts (usually bash or sh) that validate all pre-conditions and log informative 

#### Health checks
HEALTHCHECK instruction or command line. Should return 0 for healthy else 1
checked every 30s. 3 failures => unhealthy

#### Hardening

1. Ensure that the base image has not been tampered with. Note the digest value while pulling the base image.

`FROM debian@sha256:6aedee3ef827...`

2. Drop permissions to 1000:1000 user. Or better, create a custom user + group
3. If a file has the SUID or SGID bit set, it executes with root user. Ensure all files have this permission removed.
` chmod ug-s [file] `

