## pbr.sh — backup & restore for small ubuntu hosts (with remote setup and namespace remapping for docker);

**⚠️This is just a learning research project and i think i'am accomplish my goals: i have a great time with shell and bash scripting and also i faced with many security problems in context of linux host and containers while trying to do this script really secured. Idea behind concept of this workflow is really about security and lazyness. But project is archived. Where are no support or developing of this.**

- setup remote server for docker, but in safe way, so containers root don't have host root previliege level (read below about docker namespace remmaping)
- do remote server backups (host folders, docker volumes, in-docker-mysql, in-host-postgresql)
- do remote server restores (plain in-folder restore or driver specific restoration mode like "really restore mysql database")
- do all work with bash, but provide high level of security, so there shouldn't be things like password in environment variables (only via restrictive secrets files)

## General safety notes 
- docker containers run with remaped user and group ids. host user id (1000) = docker root id (0);
- all credentials stored in files with restrictive permissions
  (only for specified user — `chmod 500`)
- no credentials are present in command line args or env variables,
  so where are no ability to see sensentive data in process tree

## Docker containers safety notes

`pbr.sh` tries to change remote host linux user namespace mapping on each `host-setup` command run to get this kind of result:

```bash
~ cat /etc/subuid
username:1000:1
username:100000:65536
```

via command:
```bash
usermod -v 1000-1000 -w 1000-1000 $remote_user
```

This mapping allows to map non-root host user uid (1000 in default ubuntu setup) to docker root uid (0). So root inside container cannot have root priviliges on host machine.

- About linux user namespaces: https://man7.org/linux/man-pages/man7/user_namespaces.7.html.
- Official docker documentation about user namespace mapping: https://docs.docker.com/engine/security/userns-remap/

### Containers security overview links:
- https://book.hacktricks.xyz/linux-unix/privilege-escalation/docker-breakout/docker-breakout-privilege-escalation
- https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Docker_Security_Cheat_Sheet.mdhttps://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Docker_Security_Cheat_Sheet.md


## Remote host setup for backup
- Bootstraping backup for ubuntu servers ready for Docker (with one command + config file) with `pbr.local host-setup` command
- Linux users namespace remapping for docker users enabled by default (no host root inside docker containers)
- Provisioning available only for ubuntu based hosts, for now
- Auto backups and cron setup for docker volumes, host folders, host postgresql and docker mysql
- Notifications to telegram on success or failed execution
- Logs stored in `pbr_log.txt`

## Local host requiremenets
- bash
- [restic](https://restic.net)
- configured [rclone](https://rclone.org) (~/.config/rclone/rclone.conf)
- local public key (~/.ssh/id_rsa.pub)
- remote host with ubuntu if you need remote backups

## Remote host requirements
Ubuntu server accessible via root ssh (keys should be already uploaded to server, only key based authentication allowed)

Root required only for setup (`host-setup`) phase, while this process regular user will be created with no sudo required setup. All secrets files will have only user access (500).

## Supported backup targets
- docker volumes
- host folders (also for localhost in case of local machine backup)
- mysql in docker
- postgresql on host

## Notifications & reporting & logging
- notify to telegran if configured (can be ommited)
- host logs stored in ./pbr_log.txt
- [ ] via email

### maybe V0.2:
- multiple backup repositories for each target
- ability to provie custom uid/guid for backup containers?
- configurable callbacks functions
- option to disable root login after setup
- ability to backup host folders to sepate repo? (and strictly without docker?)

## Idea and concept

bash is good for portability, I tried strict myself with simple bash language constructions and problem solution variants, the main point was readability and simplicity.

Your local machine is a controller for all operations. `pbr.local` is our primary tool. 

### Init config and using it
When you run `pbr.local init-secrets ~/my-project` script will generate new secrets directory with configuration file. After you're done configuration and remote (or local) host pre-setup, you can proceed with server setup command `pbr.local host-setup ~/my-project/pbr.secrets/`. The second argument to `host-setup` sub-command is secrets directory path, so you can have many secrets directories for different remote hosts or setups.

### Runnig backup

If server setup finished okay after `pbr.local host-setup <secrets-dir>` command, we can try to do our first backup: 

```bash
`pbr.local host-backup <secrets-dir>`
```

You will find all backup logs at `~/pbr_log.txt`, also you will receive telegram notification if this was configured.

As for the last step in this workflow, you can try to accomplish restoring with `pbr.local host-restore <secrets-dir>`.

`pbr.local` is control script stored in local machine, then `pbr.host` will be uploaded to remote host machine after `pbr.local provise` command.

### How backup script gets to the server

`pbr.host` holds logic responsible for backup and restore mechanics. When `pbr.local host-setup` run `pbr.host` is uploaded to remote host machine and script run execution added to user cron, result for host user will looks like:

```
crontab -l
30 0 * * * /home/username/pbr.host backup
```

In case of local git clone installation to local machine you get `pbr.host` script downloaded on your machine, so no `host-setup` should be run (and can be dangeroues), and you can use this script for local backups, but on this case you need to explicitly specify secrets folder (otherwise script will attempts to work in host-mode and lookup for secrets folder in preconfigured path — `~/.config/.secrets/settings.conf`)

### Backup script (remote)

So in case of all steps are done after login to remote host you will be able to run `pbr.host backup` or `pbr.host restore` scripts via command line. Configuration files are stored in home dir with restrictive permissions, and this is a host setup, so where is no need to specify secrets directory via command line argument, but you can! `pbr.host backup ~/pbr.secrets-full-archive/` or by specify concrete setting file (not directory) — `pbr.host backup ~/pbr.secrets/my-host-01.settings.conf`.

## Localhost backup example

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

Now you can add backup script with specified config to localhost cron, add this lines to via `crontab -e` command:

```bash
30 0 * * * /Users/username/pbr.sh/pbr.host backup /Users/username/my-local-backup
```

Crontab time can be configured via option `backup_cron_time="30 0 * * *"`

## Remote host backup example

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

Assume you rclone configured you can perform host configuration:
```
~/pbr.sh/pbr.local host-setup ~/my-remote-backup
```

This command will:

- update package versions (via apt-get update and upgrade)
- install docker, restic and rclone (via apt-get)
- create user specified in `remote_user` config (if `host_create_user=true`)
- upload all secrets to remote host from specified folder (in our example: `~/my-remote-backup`)
- upload sshd config with disabled password login to remote host
- recreate keys in ~/.ssh if `host_recreate_ssh_key=true` enabled in config (false by default)
- remap user namespace files `/etc/subuid` and `/etc/subgid/` (so username uid 1000 will equals root inside docker containers)
- move all configuration secrets to home user folder and set restrictive permissions on this files (chmod 500 or 700 in most cases)
- restart sshd and docker if `host_restart_services=true` option is setin config  (false by default)

-----

After `host-setup` success you can do perform `host-backup`:

```
~/pbr.sh/pbr.local host-backup ~/my-remote-backup
```

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

   ~/pbr.sh/pbr.local {init-secrets|host-setup|host-backup|host-restore|help}

1) Create secrets folder (for each provisioning or backup scenario)

   ~/pbr.sh/pbr.local init-secrets ~/my-project

                          # ~/.my-project/pbr.secrets/                 # secrets folder will be created
                          # ~/.my-project/pbr.secrets/settings.conf    # file containing configs and credentials

2) Do configured remote host setup and provisiong

   ~/pbr.sh/pbr.local host-setup ./my-project/pbr.secrets/

3) Backup remote host

   ~/pbr.sh/pbr.local host-backup ./my-project/pbr.secrets/

4) Restore remote host

   ~/pbr.sh/pbr.local host-restore ./my-project/pbr.secrets/

Other commands:

   ~/pbr.sh/pbr.local help      - display help message

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

## All configuration options

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
host_docker_nsremap=true
host_create_user=true
host_recreate_ssh_key=true
host_restart_services=false
```
