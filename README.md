# Managing your MySQL Database Secrets with Vault over docker

This project allows to manage the secrets of database (users, passwords, roles) with `vault` and `consul` as [secret backend](https://www.vaultproject.io/docs/secrets/consul/).


---------

- [Version Vault](#version-vault)
- [Creating the docker-compose file](#creating-the-docker-compose-file)



## Version Vault

The versions used for the implementation of the project were

- `vault v:0.9.5`  [repos docker](https://hub.docker.com/_/vault/).
- `consult v:1.0.6` [repos docker](https://hub.docker.com/_/consul/).

## Creating the docker-compose file

docker.compose.yml:

```bash
version: '2.0'
services:

  config:
      container_name: config
      build: ./
      volumes:
        - /config

  vault:
      container_name: vault-dev
      image: vault:0.9.5
      links:
        - consul:consul
      ports:
        - 8200:8200
      volumes_from:
        - config
      command: vault server -config=/config/vault.conf

  consul:
      container_name: consul
      image: consul:1.0.6
      ports:
        - 8500:8500
```

run docker compose:

```bash
docker-compose up -d
```
