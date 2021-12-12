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

## Notifications & reporting & loggin
- telegram notify to telegran if configured (can be ommited)
- host logs stored in ./pbr_log.txt
- [ ] via email

## TODO

### road to 0.1
- [ ] rework forlders restore
- [ ] add cold restore
- [ ] ability to backup host folders to sepate repo? (and strictly without docker?)
- [ ] and finalizer to pbr.local

### maybe V0.2:
- multiple backup repositories for each target
- ability to provie custom uid/guid for backup containers?
- configurable callbacks functions
- restore snapshot name in notifications
- restory data only flag (just download from cloud, do not do restore)
- option to disable root login after setup
- configurable provise subscript

## Idea and concept

Your local machine is a controller for all operations, the scope of this script is small home projects with simple setups. Maybe if you work with some serious stuff, you should look at Ansible and other tools. In opposite to this caution main motivation of this script is to have a kind of strong fundament for not regular but painful operations like provisioning new hosts, backing it up and restoring data, also this script works well for local machines backup scenarios.

bash is good for portability, but perhaps in future i should rewrite this program to a higher-level language, but for now this is regular bash language. I tried strict myself with simple bash language constructions and problem solution variants, the main point was readability and simplicity.

`pbr.local` is our base tool for most operations. When you run `pbr.local init-secrets ~/my-project` script will generate new secrets directory with configuration file. After you're done configuration and remote (or local) host pre-setup, you can proceed with server setup command `pbr.local provise ~/my-project/pbr.secrets/`. The second argument to `provise` sub-command is secrets directory path, so you can have many secrets directories for different remote hosts or setups.

If server setup finished okay after `pbr.local provise <secrets-dir>` command, we can try to do first backup: `pbr.local host-backup <secrets-dir>`. You will find all backup logs at `~/pbr_log.txt`, also you will receive telegram notification if this was configured.

As for the last step in this workflow, you can try to accomplish restoring with `pbr.local host-restore <secrets-dir>`.

`pbr.local` is control script stored in local machine, then `pbr.host` will be uploaded to remote host machine after `pbr.local provise` command.

### How backup script gets to the server

`pbr.host` holds logic responsible for backup and restore mechanics. When `pbr.local provise` run `pbr.host` is uploaded to remote host machine and script run execution added to user cron via , result for host user looks like:

```
crontab -l
30 0 * * * /home/username/pbr.host backup
```

### Backup script (remote)

So in case of all steps are done after login to remote host you will be able to run `pbr.host backup` or `pbr.host restore` scripts via command line. Configuration files are stored in home dir with restrictive permissions, and this is a host setup, so where is no need to specify secrets directory via command line argument, but you can! `pbr.host backup ~/pbr.secrets-full-archive/` or by specify concrete setting file (not directory) — `pbr.host backup ~/pbr.secrets/my-host-01.settings.conf`.

### Localhost backup

Just clone pbr.sh repo to local machine (for this document i asssue we clone to ~ for command line examples readability),
then configure basic lcoalhost backup, then run by self or add to crontab:

```bash
# clone pbr.sh script to local home dir
cd ~ && git clone git@github.com:pechorin/pbr.sh.git

# create secrets folder and init blank configuration file
~/pbr.sh/pbr.local init-secrets . my-local-backup

# this will be enough for localhost backup
echo "# autogenerated config, this is bash script
      remote_user=username
      backup_name=macbook-work
      backup_storage_password=mystoragepassword
      data_rclone_repo=rclone:yandex-s3:local-macbook-data-backup
      telegram_chat_id=12345678
      telegram_bot_token=telegram:api-token
      restic_backup_args='--exclude=\"*.log\" --exclude=\".git\" --exclude=\"node_modules\"'
      host_backup_folders=/Users/userame/documents,/Users/username/work
" > ~/my-local-backup/settings.conf

# assume you rclone configured you can do:
~/pbr.sh/pbr.host ~/my-local-backup/settings.conf

# or just specify folder if settings file are default:
~/pbr.sh/pbr.host ~/my-local-backup

```

if all configured right you're will get result:

```text
cat ~/pbr_log.txt
--------------------
Starting backup at Sun 12 Dec 2021 12:30:02 AM UTC
running host-only folders backup
backup args -> --exclude="*.log" --exclude=".git" --exclude="node_modules"
open repository
lock repository
load index files
using parent snapshot e61ad066
start scan on [/Users/username/documents /Users/username/work]
start backup on [/Users/username/documents /Users/username/work]
scan finished in 6.568s: 241606 files, 32.380 GiB

Files:           0 new,     1 changed, 241605 unmodified
Dirs:            0 new,     3 changed, 30464 unmodified
Data Blobs:      1 new
Tree Blobs:      4 new
Added to the repo: 1.814 MiB

processed 241606 files, 32.380 GiB in 0:17
snapshot 4f2d6c4b saved
Skipping db backup
backup completed at Sun 12 Dec 2021 12:30:20 AM UTC
```

### Remote host backup

```bash
# clone pbr.sh script to local home dir
cd ~ && git clone git@github.com:pechorin/pbr.sh.git

# create secrets folder and init blank configuration file
~/pbr.sh/pbr.local init-secrets . my-remote-backup

# this will be enough for host backup
echo "# autogenerated config, this is bash script
      remote_user=username
      remote_user_password=remoteuserpassword # if none then password prompt will appear
      ssh_connection_string=root@my-remote-host-01
      backup_name=host-01
      backup_storage_password=mystoragepassword
      db_rclone_repo=rclone:yandex-s3:host-01-db-backup
      data_rclone_repo=rclone:yandex-s3:host-01-data-backup
      telegram_chat_id=12345678
      telegram_bot_token=telegram:api-token
      restic_backup_args='--exclude=\"*.log\" --exclude=\".git\" --exclude=\"node_modules\"'
      host_backup_folders=/Users/userame/documents,/Users/username/work

      # MySQL in docker backup settings
      docker_mysql_user=root
      docker_mysql_password=mysqlpassword
      docker_mysql_db_names=wordpress

      # Docker volumes data backup settings
      docker_backup_volumes=username_wp_data
" > ~/my-remote-backup/settings.conf
```

Assume you rclone configured you can perform host configuration, this command will:

- update package versions (via apt-get update and upgrade)
- install docker, restic and rclone (via apt-get)
- create user specified in `remote_user` config if none
- upload all secrets to remote host from specified folder (in our example: `~/my-remtote-backup`)
- upload sshd config with disabled password login to remote host 
- recreate keys in ~/.ssh if `provise_recreate_ssh_key=true` enabled in config (false by default)
- remap user namespace files `/etc/subuid` and `/etc/subgid/` (so username uid 1000 will equals root inside docker containers)
- move all configuration secrets to home user folder and set restrictive permissions on this files (chmod 500 or 700 in most cases)
- restart sshd and docker if `provise_restart_services=true` option is setin config  (false by default)

```
~/pbr.sh/pbr.local provise ~/my-remote-backup
```

After success you can do backup:

```
~/pbr.sh/pbr.local host-backup ~/my-remote-backup
```


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
docker_mysql_db_names="my_mysql_db1,my_mysql_db2"

# [ Option can be disabled ]
# Postgresql settings
host_postgresql_user="my_user"
# Enable this to do pg backups
# host_postgresql_db_names="my_pg_db1,my_pg_db_2"

# [ Option can be disabled ]
# Docker volumes data backup settings
docker_backup_volumes="my_volume1,my_volume2"

# [ Option can be disabled ]
# Host folders data backup settings
host_backup_folders=/home/username/.config

# [ Provise default values, can be changed ]
provise_create_user=true
provise_recreate_ssh_key=true
provise_host_docker_nsremap=true
provise_restart_services=false
```