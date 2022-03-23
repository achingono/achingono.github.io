---
author: "Alfero Chingono"
title: "Optimizing Dockerfile Startup Script With Environment Variables"
date: 2021-03-23T18:08:25Z
draft: false
description: "In this post, I will show how I optimized my Dockerfile startup script to leverage environment variables more elegantly."
slug: optimizing-dockerfile-startup-script-with-environment-variables
tags: [
    "docker",
    "bash",
]
categories: [
    "containers",
    "development"
]
image: "cover.png"
---

In my previous post; [Combining ENTRYPOINT and CMD in a Dockerfile]({{< ref "/post/dockerfile-combine-entrypoint-cmd/index.md" >}}), I showed how I added reusability to my `Dockerfile` by combining `ENTRYPOINT` and `CMD`.

To take things further, I didn't like how I still had to supply arguments to `entrypoint.sh` when those arguments were already available in the container as environment variables, so I eventually modified my `Dockerfile` this way:

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
COPY entrypoint.sh /bin/
COPY testconnection.sh /bin/
RUN chmod +x /bin/entrypoint.sh
RUN chmod +x /bin/testconnection.sh

ENTRYPOINT [ "/bin/entrypoint.sh" ]
CMD ["sh", "-c", "dotnet $ASSEMBLY"]
```

Subtle change; the location of the `entrypoint.sh` changed to `/bin/entrypoint.sh` and the `$DB_SERVICE` as well as the `$DB_SERVICE_PORT` arguments are missing.
So, how does my `entrypoint.sh` know which server to check for connections before running our startup command? Thanks for asking:

```bash
#!/bin/bash
set -e

# check if the startup command has been provided
if [ "$1" == "" ]; then
 echo "Startup command not set. Exiting"
 exit;
fi

# check if the $DB_SERVICE environment variable has been set
if [ "$DB_SERVICE" == "" ]; then
 echo "Environment variable 'SQL_SERVICE_NAME' not set. Exiting."
 exit;
fi

# check if the $DB_SERVICE_PORT environment variable has been set
if [ "$DB_SERVICE_PORT" == "" ]; then
 echo "Environment variable 'SQL_SERVICE_PORT' not set. Exiting."
 exit;
fi

echo "Testing connection to ${DB_SERVICE}:${DB_SERVICE_PORT}"
until /bin/test-connection.sh $DB_SERVICE_NAME $DB_SERVICE_PORT; do
>&2 echo "DB Service is starting up"
sleep 1
done

>&2 echo "DB Service is up - executing command: '$@'"
# https://stackoverflow.com/a/3816747
exec "$@"
exit 0
```

As you can see, I am reading environment variables in the bash script and I use all arguments to this script as the startup command. You may prefer it the other way, but I like it this way, and I hope you find this post valuable.

As always, all comments and feedback are greatly appreciated.
