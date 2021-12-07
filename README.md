# pbr.sh — provise & backup & restore cycle for small hosts
## Utility for server provisioning and backuping with easy cmd interface

### Features
- Fast bootstraping of ubuntu servers ready for Docker (with one command + config file)
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

### Prerequiremenets
- bash on local
- ubuntu with root access via ssh on host

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
