---
layout: post
title:  "Run Terraform In Docker"
date:   2020-04-21
image: /images/posts/terraform.jpg
tags: [terraform,hashicorp,devops,docker,aws,ecs]
---

In my previous post on [compiling go in docker](http://www.homeops.tech/2020/03/15/Compile-Go-In-Docker/) I noted that it had many advantages. This is also true for tools like terraform. One of the major advantages of say [Terraform Cloud](https://www.terraform.io/docs/cloud/index.html) is the fact that you can pin or change your terraform version. Terraform like many configuration languages is not backwards compatible. Upgrading code , while easier then say puppet , can be complex
when you have multiple state files. Being able to selectively upgrade one terraform configuration while leaving the others pinned allows for more flexibility. Using docker means you don't have to rely on your local machines copy or version compatibility across your team.

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
            <td>18.0.9</td>
            <td>Ubuntu</td>
        </tr>
        <tr>
            <td>terraform</td>
            <td>12,13,14</td>
            <td>Ubuntu</td>
        </tr>
    </tbody>
</table>


  
`Dockerfile`
  

{% gist 679c5fd0c32ac40c680d09d5b1aa5b63 Dockerfile %}

The main thing to notice here is the `ARG version` which allows you to pin your docker version. We do a multi-stage build here to copy terraform out of the upstream container.

> This dockerfile runs terraform after init. It can take any argument you pass to the regular cli. I have a couple other additions which allow terraform to itself use docker but thats left to the reader (or another future article!). 

  
`backend.tvars`
  

{% gist 679c5fd0c32ac40c680d09d5b1aa5b63 backend.tvars %}

In this example I'm using s3 storage for state.

  
`variables.sh`
  

{% gist 679c5fd0c32ac40c680d09d5b1aa5b63 variables.sh %}

You can send vars in anyway you would like, however in my case I might need some "active" vars like the current commit. This variables file works as a config file for the terraform stack. It is actually the only file you would need to update across multiple repos as well.

  
`functions.sh`
  

{% gist 679c5fd0c32ac40c680d09d5b1aa5b63 functions.sh %}

This script uses functions to override the local invocations of the terraform command. 

Lets break this down...

## AWS Credentials

```shell
     --network="credentials_network" \
     ...
     --build-arg aws_container_credentials_relative_uri="${AWS_CONTAINER_CREDENTIALS_RELATIVE_URI:?}"
     ...
     --env AWS_CONTAINER_CREDENTIALS_RELATIVE_URI \
```

When you first configure terraform you use `terraform init`. This downloads the plugins and potentially creates s3 objects. In the case you store your state remotely is say s3, you also need access to AWS credentials to access that s3 bucket. Because docker only has access to what you say you need to pass in any credentials. You could at this point simply pass the AWS credentials in as build args, however that's decently insecure if your not using short lived credentials.

Our solution is to use the `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` var. This is a standard variable that is used by most of the AWS SDKs. As terraform is simply a wrapper around the golang aws sdk it also supports this. This URL is used to query the link local address `169.254.170.2` for credentials. This way the creds are handed directly to the sdk and not saved in the container. The requirements for getting this up and running are to use `amazon-ecs-local-container-endpoints`
container. There are multiple articles that cover this far better then I would. You can find the two most relevent [here](https://github.com/awslabs/amazon-ecs-local-container-endpoints/blob/mainline/docs/features.md#vend-credentials-to-containers) and [here](https://aws.amazon.com/blogs/compute/a-guide-to-locally-testing-containers-with-amazon-ecs-local-endpoints-and-docker-compose/)

This variable is passed either through a build-arg directly but ends up simply being an environmental variable thats present at both `terraform init` and `terraform apply`

> You may note that I'm not using `DOCKER_BUILDKIT` in the build step of this container. Unfortunately at the time of this writing docker buildkit does not support attaching a network like the current implementation does. This is I believe scoped but not currently working. 

## Terraform variables

```shell
export TF_VAR_git_commit=$(git rev-parse --short HEAD)
...
for var in ${!TF_VAR*} ; do
  echo "${var}=${!var}" >>.env
done
...
--env-file .env \
...
unset -f ${!TF_VAR*}

```

These snippets allow you to declare any variable with `TF_VAR` and have it passed to the container.

> This file should be in your .gitignore if it contains sensitive values, consider instead using the secure AWS credentials to bootstrap access to secrets.

{% gist 679c5fd0c32ac40c680d09d5b1aa5b63 apply.sh %}

Due the function being named terraform you, can simply remove the source line for the functions and this is essentially just a generic wrapper around the terraform commands. While `init` is handled differently e.g. its a `build` vs a `run` all other terraform commands act identically to the command line. If a team member installed the latest version of terraform, they will not "accidentally" upgrade your state by doing an apply. The terraform version is pinned to the `TERRAFORM_VERSION` var we declared. This means this is likely a drop in replacement for any existing script you might have.

> Given the caching aspects of docker this can also make things slightly more consistent for plugins. However until build kit supports docker networks, we won't be able to use the `--mount` flag to prevent plugins from being downloaded on multiple containers.

## Terraform cloud

If you find this configuration useful then you likely will enjoy migrating to [Terraform Cloud](https://app.terraform.io/) as it supports pinned versions of terraform and shared states right out of the box. I normally try run most things in TFC when possible , however some builds need to run locally. E.g. Docker in Docker style builds.
