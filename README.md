# pbr.sh — provise & backup & restore cycle for small hosts
## Utility for small hosts provisioning and backuping with easy cmd interface

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

### Supported backup targets
- docker volumes
- mysql in docker
- postgresql on host

## Usage

Generate config (or see example in `secrets.example`)
```bash
./pbr.local init-secrets ~/my-project # ~/my-project/pbr.secrets/ folder will be creted
```

Run provise script to bootstrap host and install backup and restore scripts

```bash
./pbr.local provise ~/my-project/pbr.secrets/
```

Run backup on remote machine
```bash
ssh my_user@my_host ./pbr.host backup
```

Run restore on remote machine
```bash
ssh my_user@my_host ./pbr.host restore
```

## Configuration

TODO
