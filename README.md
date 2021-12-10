# pbr.sh — provise & backup & restore cycle for small hosts
**Utility for server provisioning and backups with easy cmd interface**

Modern devops stack are cool and big, but what if we have only bash? This is a little developer attempt to automate provisioning, backup and restore for small hosts machine (running ubuntu) with docker, postgresql and mysql. This tool suitable for docker volumes backup (and host folders too). Also pbr.sh can be used for localhost forlders backup. Latest bash version required.

**⚠️This is experimental toolset and used currently only on my pet home projects. Please be careful, this is not an alpha either.**

## On Docker containers safety

pbr.sh tries to change host (not local) linux user namespace mapping on each `provise` command run to get this kind of result:

```bash
~ cat /etc/subuid
username:1000:1
username:100000:65536
```

This mapping allows to map non-root host user uid (1000 in default ubuntu setup) to docker root uid (0). So root inside container cannot have root priviliges on host machine.

- About linux user namespaces: https://man7.org/linux/man-pages/man7/user_namespaces.7.html.
- Official docker documentation about user namespace mapping: https://docs.docker.com/engine/security/userns-remap/

Security part:
- https://book.hacktricks.xyz/linux-unix/privilege-escalation/docker-breakout/docker-breakout-privilege-escalation
- https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Docker_Security_Cheat_Sheet.mdhttps://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Docker_Security_Cheat_Sheet.md

## Features
- Bootstraping (provisioning) ubuntu servers ready for Docker (with one command + config file)
- Linux users namespace remapping for docker users enabled by default (no host root inside docker containers)
- Provisioning available only for ubuntu based hosts, for now
- Auto backups and cron setup for docker volumes, host folders, host postgresql and docker mysql
- Notifications to telegram

## Safety
- docker containers run with remaped user and group ids. host user id (1000) = docker root id (0);
  you should be
- all credentials stored in files with restrictive permissions
  (only for specified user — `chmod 500`)
- no credentials are present in command line args or env variables,
  so where are no ability to see sensentive data in process tree

## Local requiremenets
- bash
- [restic](https://restic.net)
- configured [rclone](https://rclone.org) (~/.config/rclone/rclone.conf)
- local public key (~/.ssh/id_rsa.pub)
- remote host with ubuntu if you need remote backups

## Host requirements
- ubuntu server accessible via root ssh (keys should be already uploaded to server, only key based authentication allowed)

Root required only for setup (provisioning) phase, while this process regular user will be created with no sudo required setup. All secrets files will have only user access (500).

## Supported backup targets
- docker volumes
- host folders (also for localhost in case of local machine backup)
- mysql in docker
- postgresql on host
-
## Notifications & reporting & loggin
- telegram notify to telegran if configured (can be ommited)
- host logs stored in ./pbr_log.txt
- [ ] via email

## TODO

### road 0.1
- [ ] rework forlders restore
- [ ] add cold restore
- [ ] ability to backup host folders to sepate repo? (and strictly without docker?)
- [ ] add configurable restrart for services after provide?

### maybe V0.2:
- multiple backup repositories for each target
- ability to provie custom uid/guid for backup containers?
- configurable callbacks functions
- restore snapshot name in notifications
- restory data only flag (just download from cloud, do not do restore)
- option to disable root login after setup
- configurable provise subscript

## How it works

There are two scripts which can be used independently (in various scenarious): `pbr.local` for local control and `pbr.host` which uses for backup and restore process, while first `pbr.local` uses for new hosts setup and provisioning. `pbr.local` also can control `pbr.host` runs.

TODO

## Usage

### Installation

```bash
# it's okay to install to ~
cd ~
git clone git@github.com:pechorin/pbr.sh.git
```

### Usage from local machine

```
>
> pbr.local — local machine pbr.sh
>

Usage: (lets asume pbr.sh installed into ~/)

   ~/pbr.local {provise|init-secrets|host-backup|host-restore|help}

1) Create secrets folder (for each provise or backup scenario)

   ~/pbr.local init-secrets ~/my-project

                          # ~/.my-project/pbr.secrets/                 # secrets folder will be created
                          # ~/.my-project/pbr.secrets/settings.conf    # file containing configs and credentials

2) Do configured remote host setup and provisiong

   ~/pbr.local provise ./my-project/pbr.secrets/

3) Backup remote host

   ~/pbr.local host-backup ./my-project/pbr.secrets/

4) Restore remote host

   ~/pbr.local host-restore ./my-project/pbr.secrets/

Other commands:

   ~/pbr.local help      - display help message

```

### Usage from host machine

```
> pbr.host — host machine pbr.sh

Usage: ./pbr.host {backup|restore|help}

Examples:

  pbr.host backup                        - do backup
  pbr.host restore                       - do restore
  pbr.host help                          - display help message
```

## Configuration

This is full configuration example, but you can comment unnecessary features:

```bash
remote_user=my_user
ssh_connection_string=root@my_server.example

# Optional: if not provided then ask for password prompt will be used
remote_user_password=my_password

# General backup settings
backup_name="my-server"
backup_storage_password="my_storage_password"
db_rclone_repo=rclone:yandex-s3:db-backup
data_rclone_repo=rclone:yandex-s3:data-backup

# You can provide any command line argument to restic backup command
restic_backup_args='--exclude="*.log" --exclude=".git"'

# [ Options with default vaules, but can be changed ]
# backup_cron_time="30 0 * * *"
# local_public_key_path=~/.ssh/id_rsa.pub
# local_rclone_config_path=~/.config/rclone/rclone.conf

# [ Option can be disabled ]
# Telegram notifications settings
telegram_chat_id=12345678
telegram_bot_token="telegram:bot_token"

# [ Option can be disabled ]
# MySQL settings
docker_mysql_user=root
docker_mysql_password="mysql_password"
docker_mysql_db_names=my_mysql_db1,my_mysql_db2

# [ Option can be disabled ]
# Postgresql settings
host_postgresql_user="my_user"
# Enable this to do pg backups
# host_postgresql_db_names="my_pg_db1,my_pg_db_2"

# [ Option can be disabled ]
# Docker volumes data backup settings
docker_backup_volumes=my_volume1,my_volume2

# [ Option can be disabled ]
# Host folders data backup settings
host_backup_folders=/home/vorobey/.config
```

## Local backup example

This is local backup configuration example:

```bash
remote_user=username
backup_name="macbook-work"
backup_storage_password="mystoragepassword"
data_rclone_repo=rclone:yandex-s3:local-macbook-data-backup
telegram_chat_id=12345678
telegram_bot_token="telegram:api-token"
restic_backup_args='--exclude="*.log" --exclude=".git" --exclude="node_modules"'
host_backup_folders=/Users/userame/sites,/Users/vorobey/vault
```
