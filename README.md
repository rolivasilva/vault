# Managing your MySQL Database Secrets with Vault over docker

Vault is a powerful tool for managing secrets, allows to protect, store and control any tokens, passwords, certificates and other important secrets in our technological age. This project is a guide to manage the secrets database (conections, users, passwords, roles) with `vault` and `consul` as [secret backend](https://www.vaultproject.io/docs/secrets/consul/).

---------

# Summary


- [Version Vault](#version-vault)
- [File Configuration Vault](#file-onfiguration-vault)
- [Docker Containers](#docker-containers)
  - [Creating the docker-compose file](#creating-the-docker-compose-file)
- [Configure environment](#configure-environment)
- [initializing vault](#initializing-vault)
- [Unsealing Vault](#unsealing-vault)
- [Vault Tokens](#vault-tokens)
- [The mysql secret backend](#the-mysql-secret-backend)

---------

## Version Vault

Versions of `Vault` and `Consul` used from the `docker` repositories, (would have to certify for some other version if some command is not deprecated)

- `vault v:0.9.5`  [repos docker](https://hub.docker.com/_/vault/).
- `consult v:1.0.6` [repos docker](https://hub.docker.com/_/consul/).

---

## File Configuration Vault

- The configuration file `vault.hcl` is located in `/config`, the file extension is .hcl, where the listerner and backend are configured

```bash
backend "consul" {
  address = "consul:8500"
  advertise_addr = "http://consul:8300"
  path = "vault"
  scheme = "http"
}

listener "tcp" {
  address = "0.0.0.0:8200"
  tls_disable = 1
}
disable_mlock = true

````

## Creating the docker-compose file

The `docker.compose.yml` is located in the root of project, same contains the configuration of the containers `vault` and `consul`

```bash
version: '3.0'
services:

  vault:
      container_name: vault-dev
      hostname: vault
      image: vault:0.9.5
      restart: always
      networks:
        - vault
      links:
        - consul:consul
      depends_on:
        - consul
      ports:
        - 8200:8200
      environment:
        VAULT_ADDR: http://127.0.0.1:8200
      volumes:
        - ./config:/config
        - ./policies:/policies
      entrypoint: /config/init-vault.sh -t 30 -h consul -p 8500 -s -- vault server -config=/config/vault.hcl

  consul:
      container_name: consul
      image: consul:1.0.6
      restart: always
      hostname: consul
      networks:
        - vault
      command: "agent -dev -client 0.0.0.0"
      ports:
        - 8500:8500
        - 8400:8400
        - 8600:53/udp

networks:
  vault:
    driver: bridge
    ipam:
      driver: default
      config:
      - subnet: 172.16.230.0/24
        gateway: 172.16.230.1

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
$ vault operator init -address=${VAULT_ADDR} -key-shares=5 -key-threshold=2 > keys.txt
Unseal Key 1: KAs0LdwQyWD0N9dYrFF92ASt9T7Rn0MCTfELSW5X768G
Unseal Key 2: Fhb1n5qTguI2KNgHtzmBZiOp0OgBykKNtP6Mpe/Nu2Bu
Unseal Key 3: 4ANzyEcUF1LpU/4TEE+oThx1gbbtE3V1dtjX0QaDVfcY
Unseal Key 4: iSfajt0GLYMe6sOMiDg2a3BUcOQDU7YECuXky4mU1qBv
Unseal Key 5: 7BzliGGW5lHnVTAYJK2qWD1HuUIyZfShxxZnG50soi76

Initial Root Token: 27126ef9-0b1c-5296-e99c-51cdb948a12b

Vault initialized with 5 key shares and a key threshold of 2. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 2 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 2 key to
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

$ vault status -address=${VAULT_ADDR}
Key             Value
---             -----
Seal Type       shamir
Sealed          false
Total Shares    5
Threshold       2
Version         0.9.4
Cluster Name    vault-cluster-534b8145
Cluster ID      7fcab974-7176-7041-334c-b2e589141531
HA Enabled      true
HA Cluster      https://127.0.0.1:8201
HA Mode         active

```

## Vault Tokens

```bash
$ export VAULT_TOKEN=$(grep 'Initial Root Token:' keys.txt | awk '{print substr($NF, 1, length($NF)-1)}')
$ vault login -address=${VAULT_ADDR} ${VAULT_TOKEN}}
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                Value
---                -----
token              27126ef9-0b1c-5296-e99c-51cdb948a12b
token_accessor     35fd3d3b-1f5a-cf64-9d2e-d15f245bf4bf
token_duration     ∞
token_renewable    false
token_policies     [root]

```

## The mysql secret backend

```bash
$ vault secrets enable -address=${VAULT_ADDR} mysql
Success! Enabled the mysql secrets engine at: mysql/
```

## MySQL Conection Database

```bash
$ vault write -address=${VAULT_ADDR} mysql/config/connection connection_url="root:rootpw@tcp(127.0.0.1:3306)/"
WARNING! The following warnings were returned from Vault:

  * Read access to this endpoint should be controlled via ACLs as it will
  return the connection URL as it is, including passwords, if any.

```

Write user readonly

```bash
$ vault write -address=${VAULT_ADDR} mysql/roles/readonly sql="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';"
  Success! Data written to: mysql/roles/readonly

```

Read user readonly credentials

```bash
$ vault read -address=${VAULT_ADDR} mysql/creds/readonly
Key                Value
---                -----
lease_id           mysql/creds/readonly/d689549e-c899-48dc-9d0c-0b577d77b839
lease_duration     768h
lease_renewable    true
password           92f634a3-8a9e-245a-722e-f486b364f424
username           read-root-02f989

```  

```bash
$ curl -s -H  "X-Vault-Token:$VAULT_TOKEN"  -XGET http://127.0.0.1:8200/v1/mysql/creds/readonly
{"request_id":"fb48f794-6e88-e523-93a0-47075f3141f4","lease_id":"mysql/creds/readonly/15d331a5-9158-e876-2230-fe2de04e0456","renewable":true,"lease_duration":2764800,"data":{"password":"db6c08f8-36af-8954-a807-66cb6cb1c9d4","username":"read-root-b98d82"},"wrap_info":null,"warnings":null,"auth":null}

```
