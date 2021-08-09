# Vault Migrate Example

## Initialization

### Start source server

```shell
docker-compose up source
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' vault operator init > /data/tmp/init.output"
```

the unseal keys and the root token will be stored in `tmp/init.output`.

Please keep running this command (you need to fill the unseal key) until you see the output says `sealed` is `false`.

```shell
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' vault operator unseal"
```

### Add some data into source server

> Enable `generic` secret engine for `secret/` path

```shell
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/init.output | grep -i token | sed 's|.*: *||') vault secrets enable -path=secret/ generic"
```

> Write secrets

```shell
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/init.output | grep -i token | sed 's|.*: *||') vault write secret/demo hello=world"
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/init.output | grep -i token | sed 's|.*: *||') vault read secret/demo"
```

> Add policies

```shell
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/init.output | grep -i token | sed 's|.*: *||') vault policy write ro /data/policies/ro.hcl"
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/init.output | grep -i token | sed 's|.*: *||') vault policy write rw /data/policies/rw.hcl"
```

> Generate tokens

```shell
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/init.output | grep -i token | sed 's|.*: *||') vault token create -policy=ro > /data/tmp/token.ro"
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/init.output | grep -i token | sed 's|.*: *||') vault token create -policy=rw > /data/tmp/token.rw"
```

> Verify tokens

```shell
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/token.ro | grep token | head -n1 | awk '{print $2}') vault read secret/demo -format=json"
# You should see `"data": { "hello": "world" }`
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/token.ro | grep token | head -n1 | awk '{print $2}') vault write secret/demo foo=bar"
# You should see `permission denied`
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/token.rw | grep token | head -n1 | awk '{print $2}') vault write secret/demo foo=bar"
# You should see `Success! Data written to: secret/demo`
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/token.ro | grep token | head -n1 | awk '{print $2}') vault read secret/demo -format=json"
# You should see `"data": { "foo": "bar" }`
```

## Migration

Document: https://www.vaultproject.io/docs/commands/operator/migrate

> Copy storage level data from source storage to target storage

```shell
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/init.output | grep -i token | sed 's|.*: *||') vault operator migrate -config /data/migrate.hcl"
```

> Start target vault server

```shell
docker-compose up target
```

Please keep running this command (you need to fill the unseal key) until you see the output says `sealed` is `false`.
Here, we should able to keep using the unseal keys that generated by source vault.

```shell
docker-compose exec target sh -c "VAULT_ADDR='http://0.0.0.0:8200' vault operator unseal"
```

> List the secrets in source storage

```shell
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/init.output | grep -i token | sed 's|.*: *||') vault list secret/"
# You should only see `demo` in the list
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/init.output | grep -i token | sed 's|.*: *||') vault write secret/demo-post-migrate hashi=corp"
# We add a new secret record in source vault and you should see `Success! Data written to: secret/demo-post-migrate`
docker-compose exec source sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/init.output | grep -i token | sed 's|.*: *||') vault list secret/"
# You should only see both `demo` and `demo-post-migrate` in the list
```

> List the secrets in target storage

```shell
docker-compose exec target sh -c "VAULT_ADDR='http://0.0.0.0:8200' VAULT_TOKEN=$(cat tmp/init.output | grep -i token | sed 's|.*: *||') vault list secret/"
# You should only see `demo` in the list
```
