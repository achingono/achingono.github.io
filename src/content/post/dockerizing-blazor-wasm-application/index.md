---
author: "Alfero Chingono"
title: "Dockerizing Blazor Wasm Application"
date: 2021-04-07T18:39:49Z
draft: false
description: "In this post, I share how to run a Blazor WASM application in a docker container."
slug: dockerizing-blazor-wasm-application
tags: [
    "docker",
    "blazor-wasm",
    "bash",
    "nginx"
]
categories: [
    "containers",
    "development"
]
image: "cover.jpg"
---

This is a continuation of my previous post; [Waiting for Docker Service Container Port to Be Ready]({{< ref "/post/waiting-docker-service-container-port-ready/index.md" >}}). After reading [this article](https://www.c-sharpcorner.com/article/dockerizing-blazor-wasm-application/), I decided to build up on that codebase as a learning opportunity and add a Blazor WASM front-end to the .NET Core API already built.

I will not repeat the process details as that is nicely outlined in the referenced article. What I will do here is share my version of the `Dockerfile` and the associated script and configuration file.

So, here's my `Dockerfile`:

```dockerfile
# [Choice] .NET Core version: 5.0, 3.1, 2.1
ARG VARIANT=5.0
ARG PROJECT=Business.Spa
ARG CONFIG=Release

# create build image from base SDK image
FROM mcr.microsoft.com/dotnet/sdk:${VARIANT} as build
# https://github.com/moby/moby/issues/37345#issuecomment-400245466
ARG PROJECT

# Copy all the csproj files and restore to cache the layer for faster builds
# https://github.com/dotnet/dotnet-docker/issues/1697#issuecomment-589420446
COPY src/Business.Spa/Business.Spa.csproj ./src/Business.Spa/

# restore nuget packages
WORKDIR /src
RUN dotnet restore

# run publish command
FROM build as publish
# https://github.com/moby/moby/issues/37345#issuecomment-400245466
ARG PROJECT
ARG CONFIG

# copy source files
COPY src/ /src/

# run the publish command
WORKDIR /src/${PROJECT}
RUN dotnet publish -c ${CONFIG} -o /${CONFIG} --no-restore

# create release image from base runtime image
FROM nginx:alpine AS runtime
ARG CONFIG

RUN apk update \
    && apk add --no-cache openssh

# copy startup commands
COPY ./configure-environment.sh /docker-entrypoint.d/
RUN chmod +x /docker-entrypoint.d/configure-environment.sh

# copy the nginx configuration file
# https://www.c-sharpcorner.com/article/dockerizing-blazor-wasm-application/
COPY ./nginx.conf /etc/nginx/nginx.conf

ENV BLAZOR_ENVIRONMENT=Staging 
ENV REST_URL=http://localhost:8081
ENV PORT=8080

EXPOSE 8080

WORKDIR /home/site/wwwroot
COPY --from=publish /${CONFIG}/wwwroot .
```

You will notice that my `Dockerfile` does not have a `CMD` or `ENTRYPOINT` declaration. That is because the base image I'm using to host my application `nginx:alpine` already has a very nice feature where it automatically runs all scripts in the `/docker-entrypoint.d/` folder and executes them before starting up nginx.

This allowed me to copy my `configure-environment.sh` to the `/docker-entrypoint.d/` folder, make it executable and let the magic happen.

Here's my `configure-environment.sh`:

```bash
#!/bin/sh  
  
# The script replaces the Api.BaseAddress in the configuration file  
# with environment values set for the container  
  
# replace the placeholder in the blazor congfiguration file  
# https://stackoverflow.com/a/23134318
sed -i -e "s|REST_URL|${REST_URL}|g" "/home/site/wwwroot/appsettings.${BLAZOR_ENVIRONMENT}.json"

# replace the placeholders in the nginx congfiguration file 
# https://github.com/dotnet/aspnetcore/issues/25152 
sed -i -e 's/BLAZOR_ENVIRONMENT/'"${BLAZOR_ENVIRONMENT}"'/g' /etc/nginx/nginx.conf
sed -i -e 's/PORT/'"${PORT}"'/g' /etc/nginx/nginx.conf

```

And finally, here's my nginx configuration file:

```conf
events { }
http {
    include mime.types;
    types {
        application/wasm wasm;
    }
    server {
        # this will be replaced by the sed command in the configure-environment.sh script
        listen PORT;
        index index.html;
        
        # This will add the environment http header to the responses
        # https://github.com/dotnet/aspnetcore/issues/25152 
        add_header Blazor-Environment BLAZOR_ENVIRONMENT;

        location / {
            root /home/site/wwwroot;
            try_files $uri $uri/ /index.html =404;
        }
    }
}
```

Credits:  
[Dockerizing Blazor WASM Application](https://www.c-sharpcorner.com/article/dockerizing-blazor-wasm-application/)  
[docker-nginx/docker-entrypoint.sh](https://github.com/nginxinc/docker-nginx/blob/master/mainline/alpine/docker-entrypoint.sh)  
[linux - Environment variable substitution in sed](https://stackoverflow.com/a/23134318)  
[ASP.NET Core Blazor environments](https://docs.microsoft.com/en-us/aspnet/core/blazor/fundamentals/environments?view=aspnetcore-5.0)  
[Blazor WASM not loading appsettings.{environment}.json in Azure App Services](https://github.com/dotnet/aspnetcore/issues/25152)
