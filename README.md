[![Docker Pulls](https://img.shields.io/docker/pulls/itzg/mc-backup.svg)](https://hub.docker.com/r/itzg/mc-backup)
[![release](https://github.com/itzg/docker-mc-backup/workflows/release/badge.svg?branch=multiarch)](https://github.com/itzg/docker-mc-backup/actions?query=workflow%3Arelease)
[![Discord](https://img.shields.io/discord/660567679458869252?label=Discord&logo=discord)](https://discord.gg/DXfKpjB)

Provides a side-car container to backup itzg/minecraft-server world data.

## Environment variables

##### Common variables:

- `SRC_DIR`=/data
- `BACKUP_NAME`=world
- `INITIAL_DELAY`=2m
- `BACKUP_INTERVAL`=24h
- `PAUSE_IF_NO_PLAYERS`=false
- `PRUNE_BACKUPS_DAYS`=7
- `PRUNE_RESTIC_RETENTION`=--keep-within 7d
- `SERVER_PORT`=25565
- `RCON_HOST`=localhost
- `RCON_PORT`=25575
- `RCON_PASSWORD`=minecraft
- `RCON_RETRIES`=5 : Set to a negative value to retry indefinitely
- `RCON_RETRY_INTERVAL`=10s
- `EXCLUDES`=\*.jar,cache,logs
- `BACKUP_METHOD`=tar
- `RESTIC_ADDITIONAL_TAGS`=mc_backups
- `TZ` : Can be set to the timezone to use for logging

If `PRUNE_BACKUPS_DAYS` is set to a positive number, it'll delete old `.tgz` backup files from `DEST_DIR`. By default deletes backups older than a week.

If `BACKUP_INTERVAL` is set to 0 or smaller, script will run once and exit.

Both `INITIAL_DELAY` and `BACKUP_INTERVAL` accept times in `sleep` format: `NUMBER[SUFFIX] NUMBER[SUFFIX] ...`.
SUFFIX may be 's' for seconds (the default), 'm' for minutes, 'h' for hours or 'd' for days.

Examples:
- `BACKUP_INTERVAL`="1.5d" -> backup every one and a half days (36 hours)
- `BACKUP_INTERVAL`="2h 30m" -> backup every two and a half hours
- `INITIAL_DELAY`="120" -> wait 2 minutes before starting

The `PAUSE_IF_NO_PLAYERS` option lets you pause backups if no players are online.

If `PAUSE_IF_NO_PLAYERS`="true" and there are no players online after a backup is made, then instead of immediately scheduling the next backup, the script will start checking the server's player count every minute. Once a player joins the server, the next backup will be scheduled in `BACKUP_INTERVAL`.

`EXCLUDES` is a comma-separated list of glob(3) patterns to exclude from backups. By default excludes all jar files (plugins, server files), logs folder and cache (used by i.e. PaperMC server).

##### `tar` backup method

- `DEST_DIR`=/backups
- `LINK_LATEST`=false
- `TAR_COMPRESS_METHOD`=gzip

`LINK_LATEST` is a true/false flag that creates a symbolic link to the latest backup.

`TAR_COMPRESS_METHOD` is the compression method used by tar. Valid value: gzip bzip2 

##### `restic` backup method

See [restic documentation](https://restic.readthedocs.io/en/latest/030_preparing_a_new_repo.html) on what variables are needed to be defined.
At least one of `RESTIC_PASSWORD*` variables need to be defined, along with `RESTIC_REPOSITORY`.

Use the `RESTIC_ADDITIONAL_TAGS` variable to define a space separated list of additional restic tags. The backup will always be tagged with the value of `BACKUP_NAME`. e.g.: `RESTIC_ADDITIONAL_TAGS=mc_backups foo bar` will tag your backup with `foo`, `bar`, `mc_backups` and the value of `BACKUP_NAME`.

You can finetune the retention cycle of the restic backups using the `PRUNE_RESTIC_RETENTION` variable. Take a look at the [restic documentation](https://restic.readthedocs.io/en/latest/060_forget.html) for details.

> **_EXAMPLE_**  
> Setting `PRUNE_RESTIC_RETENTION` to `--keep-daily 7 --keep-weekly 5 --keep-monthly 12 --keep-yearly 75` will keep the most recent 7 daily snapshots, then 4 (remember, 7 dailies already include a week!) last-day-of-the-weeks and 11 or 12 last-day-of-the-months (11 or 12 depends if the 5 weeklies cross a month). And finally 75 last-day-of-the-year snapshots. All other snapshots are removed.

:warning: | When using restic as your backup method, make sure that you fix your container hostname to a constant value! Otherwise, each time a container restarts it'll use a different, random hostname which will cause it not to rotate your backups created by previous instances!
---|---

:warning: | When using restic, at least one of `HOSTNAME` or `BACKUP_NAME` must be unique, when sharing a repository. Otherwise other instances using the same repository might prune your backups prematurely.
---|---

:warning: | SFTP restic backend is not directly supported. Please use RCLONE backend with SFTP support.
---|---

## Volumes

- `/data` :
  Should be attached read-only to the same volume as the `/data` of the `itzg/minecraft-server` container
- `/backups` :
  The volume where incremental tgz files will be created, if using tar backup method.

## On-demand backups

If you would like to kick off a backup prior to the next backup interval, you can `exec` the command `backup now` within the running backup container. For example, using the [Docker Compose example](examples/docker-compose.yml) where the service name is `backups`, the exec command becomes:

```shell
docker-compose exec backups backup now
```

This mechanism can also be used to avoid a long running container completely by running a temporary container, such as:

```shell
docker run --rm ...data and backup -v args... itzg/mc-backup backup now
```

## Example

### Kubernetes

An example StatefulSet deployment is provided [in this repository](test-deploy.yaml).

The important part is the containers definition of the deployment:

```yaml
containers:
  - name: mc
    image: itzg/minecraft-server
    env:
      - name: EULA
        value: "TRUE"
    volumeMounts:
      - mountPath: /data
        name: data
  - name: backup
    image: mc-backup
    imagePullPolicy: Never
    securityContext:
      runAsUser: 1000
    env:
      - name: BACKUP_INTERVAL
        value: "2h 30m"
    volumeMounts:
      - mountPath: /data
        name: data
        readOnly: true
      - mountPath: /backups
        name: backups
```

### Docker Compose

```yaml
version: '3.7'

services:
  mc:
    image: itzg/minecraft-server
    ports:
    - 25565:25565
    environment:
      EULA: "TRUE"
    volumes:
    - mc:/data
  backups:
    image: itzg/mc-backup
    environment:
      BACKUP_INTERVAL: "2h"
      # instead of network_mode below, could declare RCON_HOST
      # RCON_HOST: mc
    volumes:
    # mount the same volume used by server, but read-only
    - mc:/data:ro
    # use a host attached directory so that it in turn can be backed up
    # to external/cloud storage
    - ./mc-backups:/backups
    # share network namespace with server to simplify rcon access
    network_mode: "service:mc"

volumes:
  mc: {}
```

### Restic with rclone

Setup the rclone configuration for the desired remote location
```shell
docker run -it --rm -v rclone-config:/config/rclone rclone/rclone config
```

Setup the `itzg/mc-backup` container with the following specifics
- Set `BACKUP_METHOD` to `restic`
- Set `RESTIC_PASSWORD` to a restic backup repository password to use
- Use `rclone:` as the prefix on the `RESTIC_REPOSITORY`
- Append the rclone config name, colon (`:`), and specific sub-path for the config type

In the following example `CFG_NAME` and `BUCKET_NAME` need to be changed to specifics for the rclone configuration you created:
```yaml
version: "3"

services:
  mc:
    image: itzg/minecraft-server
    environment:
      EULA: "TRUE"
    ports:
      - 25565:25565
    volumes:
      - mc:/data
  backup:
    image: itzg/mc-backup
    environment:
      RCON_HOST: mc
      BACKUP_METHOD: restic
      RESTIC_PASSWORD: password
      RESTIC_REPOSITORY: rclone:CFG_NAME:BUCKET_NAME
    volumes:
      # mount volume pre-configured using
      # docker run -v rclone-config:/config/rclone rclone/rclone config
      - rclone-config:/config/rclone
      - mc:/data:ro
      - backups:/backup

volumes:
  rclone-config:
    # declared external since it is created by one-off docker run usage
    external: true
  mc: {}
  backups: {}
```
