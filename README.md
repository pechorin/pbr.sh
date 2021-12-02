# pbr.sh — provise / backup / restore
## Utility for small hosts with docker setup

### Fast bootstraping of ubuntu servers ready for Docker
- Linux users namespace remapping for docker users enabled by default (no host root inside docker containers)

### Supported backup targets
- docker volumes
- mysql in docker
- postgresql on host

## Usage

Generate config (or see example in `secrets.example`)
```bash
./provise config ./my-project/secrets/
```

Run provise script to bootstrap host and install backup and restore scripts
```bash
./provise
```

Run backup on remote machine
```bash
ssh my_user@my_host ./backup
```

Run restore on remote machine
```bash
ssh my_user@my_host ./backup restore
```
