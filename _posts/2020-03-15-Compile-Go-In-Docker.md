---
layout: post
title:  "Compile Go In Docker"
date:   2020-03-15
image: /images/posts/golang.jpg
tags: [docker,golang,go,make,ssh]
---

I have recently been moving almost all of my workflows to docker. This makes allot of sense over time as things like your development environment can change from laptop to laptop or os to os. While I think something like `https://nixos.org/` is a great step forward in the direction of a common development and delivery platform. I still tend to fallback to docker when building simple shell scripts that I need to share with members of my team. Running things in docker can mean that you can also
pin the version of the  given software or libraries which has some huge stability and repeatability implications over time. Somthing many people miss when sharing assets among each other.


<!--more-->


<table>
    <caption>Versions tested</caption>
    <tbody>
        <tr>
            <th>Software</th>
            <th>Version</th>
            <th>OS</th>
        </tr>
        <tr>
            <td>docker</td>
            <td>20.10.2</td>
            <td>Ubuntu</td>
        </tr>
    </tbody>
</table>

In an upcoming artible I will cover how I use this same trick to selectivly upgrade terraform code across a suite of lambdas. In the case of Golang, using docker has the major benefits e.g. its likely a container is my final output as well as my compile time environment. Most of the code I write ends up in one way or another on a serverless platform like AWS Lambda. Lambdas recently started support docker containers in liue of the standard zip file we had all grown accustom to. This means I can use the
environment to compile my code and then the resultant container to deploy it.


# Dockerfile

```docker
 FROM golang:latest
 
 ARG target
 
 # Go build configuration
 ENV GO111MODULE="on"
 ENV GOPRIVATE="github.com/homeops-tech/slackbot"
 ENV GOOS="linux"
 ENV CGO_ENABLED="0"
 # Setup our container
 RUN mkdir /src
 COPY . /src/
 WORKDIR /src
 
 # Create the module (refresh with rm)
 RUN test -f go.mod || go mod init "github.com/homeops-tech/slackbot"
 
 # Setup SSH
 RUN git config --global url."git@github.com:".insteadOf "https://github.com/"
 RUN mkdir -p /root/.ssh/
 RUN ssh-keyscan github.com >> /root/.ssh/known_hosts
 
 # Download go modules
 RUN  --mount=type=ssh go mod download -x
 COPY . .
 
 # Build the module
 RUN --mount=type=cache,target=/root/.cache/go-build go build -ldflags="-s -w" ${target}
```

Lets break this down, the `GOPRIVATE` directive here is due to the fact that the repo/module we are using is private on Github.

```shell
RUN git config --global url."git@github.com:".insteadOf "https://github.com/"
RUN mkdir -p /root/.ssh/
RUN ssh-keyscan github.com >> /root/.ssh/known_hosts
```

This allows use to rewrite the http url to ssh and download the known host keys during build.

> This can get quite abstract but you need to authenticate inside the container to these repos. This done normally by ssh agents. see the makefile example below.

## Using Buildkit

```shell
RUN  --mount=type=ssh go mod download -x
COPY . .
```

Dockers layer system is great for build processes as it will cache things in each layer and reuse them across builds. In the case of common libraries, especially external we dont want that. Its just downloading the same cache over and over. Given a language like go has a compliation , this can slow down your development process. The `--mount` flag uses the new expiermental BuildKit function of docker to cache just that layer across invocations.

This requires a special environmental variable you can see in this Makefile example

  
`Makefile`



```make
PROJECT="slackbot"

build:
    # Build go docker image with dependencies 
    @SSH_AUTH_SOCK=`launchctl getenv SSH_AUTH_SOCK` DOCKER_BUILDKIT=1 docker build \
        --rm \
        --force-rm \
        --ssh=default \
        --build-arg target='./...' \
        --tag \
        $(PROJECT):latest \
        .
    # Copy updated go.mod and go.sum out of container
    @docker cp $(shell docker create $(PROJECT):latest):/src/go.mod go.mod
    @docker cp $(shell docker create $(PROJECT):latest):/src/go.sum go.sum

.DEFAULT_GOAL := build

.PHONY: build
```

## Dependencies

When using golang with other imported modules, you may create a go.mod file to pin versions of those libs. When using docker I find it useful to have that process happen inside the docker container. 

```shell
docker cp $(shell docker create $(PROJECT):latest):/src/go.mod go.mod
docker cp $(shell docker create $(PROJECT):latest):/src/go.sum go.sum
```

This trick means you are getting the "internal" versions that were latest when you did the download. Paired with `RUN test -f go.mod || go mod init "github.com/homeops-tech/slackbot"` in your Dockerfile allows you pin version between builds. The simple example of this is on the first build, your container runs and downloads the modules and generates the file. The `docker cp` commands then copies the files out of the container. On the next run, your docker container is pinned to these libraries. You can manually update them by editing the file or if you want to get the latest of all libs, simply delete the files and they will be regenerated.


> I find this incredibily helpful when testing newer go versions via the Docker tag `FROM golang:latest` by editing these two files you can check compatibilty without changing your local go installation.


## SSH Agents

You may notice the `SSH_AUTH_SOCK` var above is pointed towards this command `launchctl getenv SSH_AUTH_SOCK`. This is essentially a supported hack on docker for mac. Unlike other platforms docker for mac is running virtually and thus it has to actually mount the ssh socket for the system inside that VM. This special path allows you to use this "hack" they built in to pass your ssh credentials via your agent to the docker container. This is very important when using golang with private repos as otherwise you would not be able to download them
