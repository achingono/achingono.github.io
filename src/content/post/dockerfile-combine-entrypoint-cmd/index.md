---
author: "Alfero Chingono"
title: "Combining ENTRYPOINT and CMD in a Dockerfile"
date: 2021-03-15T17:51:18Z
draft: false
description: "In this post, I show how to combine CMD and ENTRYPOINT in the same Dockerfile."
slug: dockerfile-combine-entrypoint-cmd
tags: [
    "docker",
    "dotnet-core",
    "sql-server",
    "bash",
    "tcp"
]
categories: [
    "containers",
    "development"
]
image: "cover.png"
---
In my previous post; [Reference Build Arguments in Docker Startup Script]({{< ref "/post/reference-environment-variables-docker-startup-script/index.md" >}}), I showed how I added reusability to my `Dockerfile` by adding build arguments.

The docker builder [CMD](https://docs.docker.com/engine/reference/builder/#cmd) reference document states:

> If you would like your container to run the same executable every time, then you should consider using `ENTRYPOINT` in combination with `CMD`. See [ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#entrypoint).
>
> If the user specifies arguments to docker run then they will override the default specified in CMD.

So, after some fiddling around, I ended up with the following final version of my `Dockerfile`:

```Dockerfile
# [Choice] .NET Core version: 5.0, 3.1, 2.1
ARG VARIANT=3.1
ARG DB_SERVICE=db
ARG DB_SERVICE_PORT=1433
ARG PROJECT=Business.Web

# create a base runtime image with node
FROM mcr.microsoft.com/dotnet/core/aspnet:${VARIANT} AS runtime
EXPOSE 80
EXPOSE 443
RUN curl -sL https://deb.nodesource.com/setup_10.x |  bash -
RUN apt-get install -y nodejs

# create a base SDK image with node
FROM mcr.microsoft.com/dotnet/core/sdk:${VARIANT} as sdk
RUN curl -sL https://deb.nodesource.com/setup_10.x |  bash -
RUN apt-get install -y nodejs

# copy source files
FROM sdk as build
COPY src/ ./src

# restore nuget packages
WORKDIR /src
RUN dotnet restore

# run publish command
FROM build as publish
RUN dotnet publish -c Release -o /release --no-restore

# create release image from base runtime image
FROM runtime AS release
# https://github.com/moby/moby/issues/37345#issuecomment-400245466
ARG PROJECT
ENV ASSEMBLY=${PROJECT}.dll
COPY --from=publish /release .
COPY entrypoint.sh .
COPY testconnection.sh .
RUN chmod +x ./entrypoint.sh
RUN chmod +x ./testconnection.sh

ENTRYPOINT [ "./entrypoint.sh" ]
CMD ["sh", "-c", "$DB_SERVICE $DB_SERVICE_PORT dotnet $ASSEMBLY"]
```

Configuring it this way allows me to supply different arguments when running the container without having to specify `entrypoint.sh` every time. Small change, and I hope this proves helpful to you, dear reader. All comments and feedback will be greatly appreciated.
