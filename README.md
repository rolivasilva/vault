# Managing your MySQL Database Secrets with Vault over docker

This project allows to manage the secrets of database (users, passwords, roles) with `vault` and `consul` as [secret backend](https://www.vaultproject.io/docs/secrets/consul/).


---------

- [Version Vault](#version-vault)
- [Creating the docker-compose file](#creating-the-docker-compose-file)
- [Configure environment](#Configure-environment)
- [initializing vault](#initializing-vault)
- [Unsealing Vault](#Unsealing-Vault)
- [Vault Tokens](#Vault-Tokens)
- [The mysql secret backend](#The-mysql-secret-backend)

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
$ docker-compose up -d
```

## Configure environment

This command will create an alias and the vault address to the Docker container.

```bash
$ alias vault='docker exec -it vault-dev vault "$@"'

$ export VAULT_ADDR=http://127.0.0.1:8200
```

## Initializing vault


```bash
$ vault operator init -Initializing vaultVAULT_ADDR} -key-shares=5 -key-threshold=3
Unseal Key 1: ZEVedUL9zP+oLGuMLHv8l+OMMPyLfiv4y983SA+WUl0O
Unseal Key 2: IKHlewPE3ZKSJlDEC0L7akM/rflMzGFZVmj6wmcoQ+LN
Unseal Key 3: LEo9Ya/UHhFcY1Y1mkHxMS1xZHKxkpT2/EuGlirDrjOp
Unseal Key 4: GWGmL7wpWS/A6QFyAvoUz5Bqcli/O8PdOJxeMmDGxHTc
Unseal Key 5: uazr6Gag5CzoMZkf/YnBwlaALhtwgyXmA0dHvKjtu1TC

Initial Root Token: c6a220d1-d3d3-13b2-980a-d713f4968e70

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault rekey" for more information.
```

## Unsealing Vault

```bash
$ vault operator unseal -address=${VAULT_ADDR} $(grep 'Key 1:' keys.txt | awk '{print $NF}')
Key (will be hidden):
Sealed: true
Key Shares: 5
Key Threshold: 3
Unseal Progress: 1

$ vault operator unseal -address=${VAULT_ADDR} $(grep 'Key 2:' keys.txt | awk '{print $NF}')
Key (will be hidden):
Sealed: true
Key Shares: 5
Key Threshold: 3
Unseal Progress: 2

$ vault operator unseal -address=${VAULT_ADDR} $(grep 'Key 3:' keys.txt | awk '{print $NF}')
Key (will be hidden):
Sealed: false
Key Shares: 5
Key Threshold: 3
Unseal Progress: 0
```

## Vault Tokens

```bash
$ export VAULT_TOKEN=$(grep 'Initial Root Token:' keys.txt | awk '{print substr($NF, 1, length($NF)-1)}')
$ vault login -address=${VAULT_ADDR} ${VAULT_TOKEN}
```

## The mysql secret backend

```bash
vault secrets enable -address=${VAULT_ADDR} mysql
```

## MySQL Conection Database

```bash
$ vault write -address=${VAULT_ADDR} mysql/config/connection\
connection_url="root:zIeYwEM4F1J8t0t@tcp(192.168.1.56:3306)/"
```

Write user readonly

```bash
$ vault write -address=${VAULT_ADDR} mysql/roles/readonly\
sql="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT\
  ON *.* TO '{{name}}'@'%';"
```

Read user readonly credentials

```bash
$ vault read -address=${VAULT_ADDR} mysql/creds/readonly
```  
