---
author: "Alfero Chingono"
title: "Restore Database on Container Start Up"
date: 2021-04-21T10:20:50Z
draft: false
description: "In this post I share how I automatically restored a database backup every time a Docker container started up."
slug: restore-database-on-container-start-up
tags: [
"docker",
"sql-server"
]
categories: [
    "containers",
    "development"
]
image: "cover.jpg"
---

This is a continuation from my previous posts; [Dockerizing Blazor Wasm Application]({{< ref "/post/dockerizing-blazor-wasm-application/index.md" >}}) and [Waiting for Docker Service Container Port to Be Ready]({{< ref "/post/waiting-docker-service-container-port-ready/index.md" >}}).

One of the main reasons I needed my application container to wait for the database container to be ready was because I needed to initialize and seed the database before launching the application. This created a significant, and somewhat unacceptable, delay in container startup which impacted local development experience and automated UI tests.

I went on a quest to search for a solution to this problem and found out that Database engines such as MySQL support the ability to automatically restore database backups when creating a docker container. Surely, Microsoft SQL Server would have the same feature, right? Apparently, not. So I had no choice but to roll out my own solution, right? Right!

Now that we agree on the legitimacy of my quest, here are the changes I made to my project.

First, I created a `entrypoint.sh` bash script with the following contents:

```bash
#!/bin/bash

# Adapted from: github.com/microsoft/mssql-docker/issues/11

# Launch MSSQL and send to background
/opt/mssql/bin/sqlservr &
pid=$!
# Wait for it to be available
echo "Waiting for MS SQL to be available ⏳"
/opt/mssql-tools/bin/sqlcmd -l 30 -S localhost -h-1 -V1 -U sa -P $SA_PASSWORD -Q "SET NOCOUNT ON SELECT \"YAY WE ARE UP\" , @@servername"
is_up=$?
while [ $is_up -ne 0 ] ; do
    echo -e $(date)
    /opt/mssql-tools/bin/sqlcmd -l 30 -S localhost -h-1 -V1 -U sa -P $SA_PASSWORD -Q "SET NOCOUNT ON SELECT \"YAY WE ARE UP\" , @@servername"
    is_up=$?
    sleep 5
done

LOG_FILE=output.log
# check flag so that this is only done once on creation,
#      and not every time the container runs
if [ ! -f "${SCRIPTS_PATH}/${LOG_FILE}" ]; then
    # Run every bash script in /var/opt/mssql/scripts
    # https://stackoverflow.com/a/49383879
    for file in "${SCRIPTS_PATH}/"*.sh; do
        echo "Executing: ${file}" >> "${SCRIPTS_PATH}/${LOG_FILE}"
        if [ -x "$file" ]; then
            echo "Executing $file";
            "$file"
        else
            # warn on shell scripts without exec bit
            echo "Ignoring $file, not executable.";
        fi
    done

    # https://stackoverflow.com/a/49383879
    for file in "${SCRIPTS_PATH}/"*.sql; do
        echo "Executing: ${file}" >> "${SCRIPTS_PATH}/${LOG_FILE}"
        if test -f "$file"; then
            echo "Executing $file";
            /opt/mssql-tools/bin/sqlcmd -U sa -P $SA_PASSWORD -l 30 -e -i $file
        fi
    done
    echo "All scripts have been executed."
fi
echo "Waiting for MS SQL(pid $pid) to terminate."
# trap SIGTERM and send same to sqlservr process for clean shutdown
trap "kill -15 $pid" SIGTERM
# Wait on the sqlserver process
wait $pid
```

The script above does four main things:
- Start MSSQL engine in the background
- Wait for the engine to be ready for connections
- Execute scripts in a specified folder
- Cleanly shutdown the MSSQL process when the `SIGTERM` signal is received.

You may also notice that there are two loops through the files in the scripts folder. The first loop iterates over bash scripts while the second iterates over SQL scripts.

Next, I created a script file that restores the database:

```bash
#!/bin/bash

if [ "${DATABASE}" == "" ]; then 
    echo "'DATABASE' environment variable not set."
    exit 0
fi

BACKUP_FILE=/var/opt/mssql/backup/${DATABASE}.bak
DATA_FILE=/var/opt/mssql/data/${DATABASE}.mdf
LOG_FILE=/var/opt/mssql/data/${DATABASE}_log.ldf

if test -f "${BACKUP_FILE}"; then
    echo "${BACKUP_FILE} found."
else
    echo "${BACKUP_FILE} not found."
    exit 0
fi

echo "Restoring database ${DATABASE}"
/opt/mssql-tools/bin/sqlcmd \
   -S localhost -U sa -P ${SA_PASSWORD} \
   -Q 'RESTORE DATABASE '${DATABASE}' FROM DISK = "'${BACKUP_FILE}'" WITH MOVE "'${DATABASE}'" TO "'${DATA_FILE}'", MOVE "'${DATABASE}'_Log" TO "'${LOG_FILE}'"'
```

All this script does is check for a `DATABASE` environment variable and by convention, a backup file with a matching name in a specified folder. If both exist, then it executes a `RESTORE DATABASE` command.

Then, I created a `Dockerfile` with the following contents:

```Dockerfile
ARG DATABASE=Business
ARG SCRIPTS_PATH=/var/opt/mssql/scripts

FROM mcr.microsoft.com/mssql/server:2019-latest
ARG DATABASE
ARG SCRIPTS_PATH

COPY --chown=mssql:mssql entrypoint.sh /bin
RUN chmod +x /bin/entrypoint.sh
RUN mkdir -p ${SCRIPTS_PATH}
COPY --chown=mssql:mssql restore-database.sh ${SCRIPTS_PATH}
RUN chmod +x ${SCRIPTS_PATH}/restore-database.sh
RUN mkdir -p /var/opt/mssql/backup
COPY --chown=mssql:mssql database.bak /var/opt/mssql/backup
RUN mv /var/opt/mssql/backup/database.bak /var/opt/mssql/backup/${DATABASE}.bak

ENV DATABASE=${DATABASE}
ENV SCRIPTS_PATH=${SCRIPTS_PATH}
# https://docs.docker.com/compose/aspnet-mssql-compose/
ENTRYPOINT ["/bin/entrypoint.sh"]
```

This `Dockerfile` basically copies three files into the base MSSQL container:
- `entrypoint.sh`
- `restore-database.sh`
- `database.bak`

Once those three files are copied to the right folders and made executable, the magic happens when the container is created.

The last step was to update the `docker-compose.yml` file with the following changes:

```diff
  data:
-    image: mcr.microsoft.com/mssql/server:2019-latest 
+    build:
+      context: .
+      dockerfile: Dockerfile
+    image: business/data:latest
+    container_name: business-data
    environment:
      - SA_PASSWORD=${SQL_PASSWORD}
      - ACCEPT_EULA=Y
+      - DATABASE=${SQL_DATABASE}
    restart: unless-stopped
    ports:
      - 1433:1433
```

With those changes in place, the database container creation process was multiple times faster than before and my automated UI tests were happier for it.

References:  
[Restore a SQL Server database in a Linux Docker container](https://docs.microsoft.com/en-us/sql/linux/tutorial-restore-backup-in-sql-server-container)  
[BASH - FOR loop using LS and wildcard](https://stackoverflow.com/a/49383879)  