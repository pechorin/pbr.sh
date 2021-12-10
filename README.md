# pbr.sh — provise & backup & restore cycle for small hosts
## Utility for server provisioning and backuping with easy cmd interface

Modern devops stack are cool and big, but what if we have only bash? This is a little developer attempt to automate provisioning, backup and restore for small hosts machine (running ubuntu) with docker, postgresql and mysql. This tool suitable for docker volumes backup (and host folders too). Also pbr.sh can be used for localhost forlders backup. Latest bash version required.

## On Docker containers safety

pbr.sh trys to change host (not local) linux user namespace mapping to get result kind of:

```bash
~ cat /etc/subuid
username:1000:1
username:100000:65536
```

Read more about user namespaces: https://man7.org/linux/man-pages/man7/user_namespaces.7.html.

This mapping allow to map non-root host user uid (1000 in default ubuntu setup) to docker root uid (0). So root inside container cannot have root priviliges on host machine. https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Docker_Security_Cheat_Sheet.md

Please read official docker documentation about user namespace mapping: https://docs.docker.com/engine/security/userns-remap/

### Features
- Bootstraping (provisioning) ubuntu servers ready for Docker (with one command + config file)
- Linux users namespace remapping for docker users enabled by default
  (no host root inside docker containers)
- For Ubuntu based hosts servers only
- Auto backups cron setup

### Safety
- docker containers run without host root (user namespaces remmapped)
- all credentials stored in files with restrictive permissions
  (only for specified user — `chmod 500`)
- no credentials are present in command line args or env variables,
  so where are no ability to see sensentive data in process tree

### Requiremenets
- [pbr.local] bash
- [pbr.local] configured rclone (~/.config/rclone/rclone.conf)
- [pbr.local] local public key (~/.ssh/id_rsa.pub)
- [pbr.host] ubuntu accessible with ssh (keys should be already uploaded to server, only key based authentication allowed)

### Supported backup targets
- docker volumes
- host folders
- mysql in docker
- postgresql on host

## Usage

```
>
> pbr.local — local machine pbr.sh
>

Usage: (inside script dir)

   ./pbr.local {provise|init-secrets|host-backup|host-restore|help}

1) Create secrets folder in project directory

   pbr.local init-secrets ~/my-project

   # As result: ~/.my-project/pbr.secrets/ folder will be created

2) Do configured remote host setup and provisiong

   pbr.local provise ./my-project/pbr.secrets/

3) Backup remote host

   pbr.local host-backup ./my-project/pbr.secrets/

4) Restore remote host

   pbr.local host-restore ./my-project/pbr.secrets/

Other commands:

   pbr.local help                - display help message
   
```


## Configuration

TODO
