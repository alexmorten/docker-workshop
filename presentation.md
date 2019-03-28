theme: Courier, 6

![original 20%](docker-logo.png)

---

# Who are we?

---

# What do we want?

---

[.build-lists: true]

# Things I planned to cover

- Why Docker
- Installation
- Basics of docker
- Containers in depth
  - Networks
  - Volumes

---

[.build-lists: true]

# Things I planned to cover

- Docker-Compose
- Excercise with own projects
- Useful tricks
- Pitfalls when using Docker

---

[.build-lists: true]

# Why Docker?

- Reproducibly installing dependencies
- Development very close to production
- Easily deploying apps
- Ecosystem of useful, shared components
- Little overhead

---

# Installation

##[Fit] docs.docker.com/install/

##[Fit] docs.docker.com/compose/install/

---

```bash
$ docker version
Client: Docker Engine - Community
 Version:           18.09.2
 API version:       1.39
 Go version:        go1.10.8
 Git commit:        6247962
 Built:             Sun Feb 10 04:12:39 2019
 OS/Arch:           darwin/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.2
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.10.6
  Git commit:       6247962
  Built:            Sun Feb 10 04:13:06 2019
  OS/Arch:          linux/amd64
  Experimental:     false
```

---

```bash
docker-compose --version
docker-compose version 1.23.2, build 1110ad01
```

---

# Docker Basics

---

[.build-lists: true]

# What is an image?

- Blueprint
- Contains everything necessary to run an application
- A layered, read-only File System

---

[.build-lists: true]

# Why layers?

- Enable caching of shared requirements
- Each layer has a globally unique id
- only changed layers need to be pushed/pulled

---

[.build-lists: true]

# What is a container?

- Contains application state
- Thin read/write layer

---

### Each layer is a File System tree

![left fit](container-layers.jpg)

---

## Reading a file

### go down the layered File System, take first occurence

![left fit](container-layers.jpg)

---

## Writing a file

### Write in the containers read/write layer

![left fit](container-layers.jpg)

---

## Changing an existing file

### create copy of read-only file in read/write layer first

![left fit](container-layers.jpg)

---

# [Fit] An Image can be reused in multiple containers

![inline](sharing-layers.jpg)

---

# Example

## clone github.com/alexmorten/docker-example

---

# Dockerfile

```dockerfile
# Use an official Python runtime as a parent image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
# ignore files specified in .dockerignore
COPY . /app

# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

---

[.build-lists: true]

# Build the image

`docker build -t example-app .`

- `-t` tags the image with a nice name

---

[.build-lists: true]

# Create a running container of the image

`docker run --name app -p 4000:80 example-app`

- `--name` gives the container a nice name
- `-p` maps the host port `4000` to the container port `80`
- go to `localhost:4000` in your browser

---

# Cleanup

- `ctrl + c` cancel process running in terminal
- `docker ps --all` to see all existing containers
- `docker start app` to start container again
- `docker stop app` to stop container

---

# Cleanup

- `docker rm app` remove container
- `docker run --rm ...` would have automatically removed the container after it is stopped
  -> especially helpful for stateless containers

---

# Containers run in isolated Networks

-> no calls to things running on the host
-> no calls to other containers

---

[.build-lists: true]

# [Fit]... but our app wants to talk to Redis?!

- containers can share networks

---

[.build-lists: true]

# How?

- `docker network create my-network`
- `docker run --name app -p 4000:80 --network my-network example-app`
- `docker run --name redis --network my-network redis`
- go to `localhost:4000` in your browser

---

## When calling specific containers

### Use the containers name as a host name

```python
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)
```

---

# Isn't this a bit cumbersome?

How do I remember all of these commands?

---

# [Fit] Docker-Compose to the rescue

---

[.code: auto(1)]

## [Fit] Define your setup in a `docker-compose.yml`

```yaml
version: "3"
services:
  app:
    build: .
    ports:
      - "4000:80"
  redis:
    image: redis
    ports:
      - "6379:6379"
    volumes:
      - "./redis-data:/data"
    command: "redis-server --appendonly yes"
```

---

## `docker-compose up`

Will handle everything:

- creates the containers `app` and `redis`
- containers are part of the same network
- containers have the correct host ports bound

---

# What is

```yaml
volumes:
  - "./redis-data:/data"
```

## about?

---

# Without Volumes

### Disk writes happen in the containers read/write layer

-> Slower than normal writes to disk

### Remove the container / Update the image

-> all data **gone**

---

# With Volumes

### You can bind a host directory/file inside a container

-> No overhead in writing
-> Data persists after a container was destroyed

---

# Without Docker-Compose

`docker run ... -v "$(pwd)/redis-data:/data" redis`

---

# Questions so far?

---

[.build-lists: true]

# Exercise

1. Pick one of your projects (ideally with some kind of Database)
1. Write a `Dockerfile` for it
1. Write a `docker-compose.yaml` including all services your app depends on

---

# Leads

- What is my app supposed to do?
- Is there an appropriate Base Image?
- What dependencies do I need?

---

# Useful things

`docker run -d ...`

Run docker container in background

---

# Useful things

`docker history <container name>`

Examine size of each layer of the container

---

# Useful things

`docker exec -it <container name> /bin/bash`

Get an interactive terminal inside a running container

---

# Useful things

## Multistage builds

```Dockerfile
# use the golang dev image to compile binary
FROM golang:1.11 as builder
WORKDIR /go/src/github.com/alexmorten/mhist
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o mhist main/main.go

# use the stripped down alpine image
FROM alpine:latest
RUN apk --no-cache add ca-certificates
RUN mkdir app
# Copy the binary from the builder stage
COPY --from=builder /go/src/github.com/alexmorten/mhist/mhist /app
WORKDIR /app
CMD ["./mhist"]
EXPOSE 6666 6667
```

15MB tiny image

---

# [Fit] Pitfalls when using Docker

---

# Docker for mac

## Runs Linux inside a VM

-> `--network host` will refer to the VM

-> carries performance overhead of a VM + Docker

---

# Tiny Images

## Alpine images are **really** tiny

_Normally_ `localhost` is an alias for `127.0.0.1`

Alpine images **don't** have that alias

---

# Tiny Images

## Alpine images are **really** tiny

_Normally_ the OS contains ca-certificates to validate TLS certificates from servers for `https` calls

-> ... you should `RUN apk --no-cache add ca-certificates`

---

# That's it!

---

# Thank you!

---

# Links

Docker Installation:
`docs.docker.com/install/`
`docs.docker.com/compose/install/`

Example repo
`github.com/alexmorten/docker-example`

Slides:
`docker.alexmorten.info` (PDF)
`github.com/alexmorten/docker-workshop-presentation`
