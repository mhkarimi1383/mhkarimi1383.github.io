---
title: "Docker Builds Made Faster"
date: 2021-11-21T01:30:12+03:30
draft: false
categories: [docker]
tags: [docker, buildx, buildkit, ci, kubernetes]
keywords: [docker, buildx, buildkit, ci, kubernetes, moby, k8s, docker-cache, cache, runner]
---

# This Post will give you an introduction to Docker BuildKit & Buildx

## What is buildkit?
---
Buildkit is a project made by moby it's working on top of docker to give you ability to have custom volumes while building images (caching, secret, etc.)

Using buildkit you cat easily cache package managers (even if you change packages), change network mode per-layer, secret files only for one layer, ssh keys (cloning a repo), or running one layer as privileged, and also faster than normal docker :)

## What about buildx?
---
buildx makes buildkit even more powerful, it supports all syntaxes from buildkit it features:
1. multiple types of builders (kubernetes [also buildkit got that option but this one is better], qemu/kvm[multi-arch builds], docker container, docker)
2. custom layer cache exporting (registry with our custom name, or local)
3. pushing image (and also cache [if set]) to registry after build

    etc.

with the help of that you can have up to 6x faster builds (depends on conditions)

and I got better performance from buildx than kaniko

## Enough, How to use them?
---
### For [Buildkit](https://github.com/moby/buildkit)
You have two ways to activate it:

1. setting `DOCKER_BUILDKIT=1` as environment variable
2. set this configuration in `/etc/docker/daemon.json` and restart docker daemon
```json
{
  "features":
  {
    "buildkit": true
  }
}
```

> NOTE: You need active internet connection to get custom frontends and also docker 18.09 or higher

#### Sample Dockerfile to cache `apt` package manager
```Dockerfile
# syntax = docker/dockerfile:1.3
FROM debian
RUN rm -f /etc/apt/apt.conf.d/docker-clean; echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
RUN --mount=type=cache,target=/var/cache/apt --mount=type=cache,target=/var/lib/apt \
  apt update && apt-get --no-install-recommends install -y gcc
```
> Specify syntax to get all features that we need checkout `docker/dockerfile` in dockerhub (like shebang in bash :])

checkout [here](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/syntax.md) to see more info about syntax

### For [Buildx](https://github.com/docker/buildx)
1. if you are not using deb or rpm package get the latest release from [Release page on github](https://github.com/docker/buildx/releases/latest) & put in into ~/.docker/cli-plugins/docker-buildx
2. make it executable by running
```bash
chmod a+x ~/.docker/cli-plugins/docker-buildx
```
> NOTE: docker.io is not supported only CE & EE

Full document for buildx is [here](https://docs.docker.com/engine/reference/commandline/buildx/) there are tons of commands but syntax is as the same as Buildkit.

What I Love about these is the impact of speed of buildx & also a beautiful logging

Also Using k8s & docker-container builders you can monitor what is happening easily

thats it ;)
