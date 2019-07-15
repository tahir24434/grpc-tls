# Setting up Vault and Certify

## Vault

1. Vault comes as a binary you can place on any path in your `$PATH`. . Follow instructions from [Vault](https://www.vaultproject.io/downloads.html) website.

2. Start the server.

```bash
vault server -config=vault_config.hcl
```

3. Initialize the server.

ca.cert: "This file is used to verify the Vault server's SSL certificate" [VAULT](https://www.vaultproject.io/docs/commands/index.html#vault_addr)

```bash
export VAULT_ADDR=https://localhost:8200
export VAULT_CACERT=ca.cert
vault operator init
```

4. Unseal the Vault.

```bash
export uKey1="..."
export uKey2="..."
export uKey3="..."
vault operator unseal ${uKey1}
vault operator unseal ${uKey2}
vault operator unseal ${uKey3}
```

5. Test Vault.

```bash
$ curl \
    --cacert ca.cert \
    -i https://localhost:8200/v1/sys/health
HTTP/1.1 200 OK
Cache-Control: no-store
Content-Type: application/json
Date: Mon, 15 Jul 2019 01:42:29 GMT
Content-Length: 294

{"initialized":true,"sealed":false,"standby":false,"performance_standby":false,"replication_performance_mode":"disabled","replication_dr_mode":"disabled","server_time_utc":1563154949,"version":"1.1.3","cluster_name":"vault-cluster-d6f1a7ef","cluster_id":"50b7cade-fd03-c05c-9b19-05467bd285e7"}
```

6. Enable Vault PKI Secrets Engine backend. Follow instructions from [Vault](https://www.vaultproject.io/docs/secrets/pki/index.html) website.

```bash
vault secrets enable pki
```

Generate CA certificate and private key.

```bash
vault write pki/root/generate/internal \
    common_name=localhost \
    ttl=8760h
```

Certificate location.

```bash
vault write pki/config/urls \
    issuing_certificates="https://localhost:8200/v1/pki/ca" \
    crl_distribution_points="https://localhost:8200/v1/pki/crl"
```

Create a role (`my-role`).

```bash
vault write pki/roles/my-role \
    allowed_domains=localhost \
    allow_subdomains=true \
    max_ttl=72h
```

7. Test Vault PKI (with `vault_cn.json`).

```json
{
  "common_name": "localhost"
}
```

```bash
export TOKEN="..."
curl \
    --cacert ca.cert \
    --header "X-Vault-Token: ${TOKEN}" \
    --request POST \
    --data @vault_cn.json \
    https://localhost:8200/v1/pki/issue/my-role
{"request_id":"b1248933-c113-291d-61ac-59487ff8c27c","lease_id":"","renewable":false,"lease_duration":0,"data":{"certificate":"-----BEGIN CERTIFICATE-----\nMIIDsTCCApmgAwIBAgIUKyqoOpSkEgptLE3LOyrn/oE1MoUwDQYJKoZIhvcNAQEL\nBQAwFDESMB...-----END CERTIFICATE-----", ... }
```

## Certify

- Run the server with `make run-server-vault`

```bash
$ make run-server-vault
go run server/main.go -self=false -cefy=true
level=info time=2019-07-15T21:20:17.362421Z caller=main.go:203 msg="Server listening" port=50051
level=info time=2019-07-15T21:20:17.362684Z caller=main.go:206 msg="Starting gRPC services"
level=info time=2019-07-15T21:20:17.362737Z caller=main.go:209 msg="Listening for incoming connections"
level=debug time=2019-07-15T21:20:26.121926Z caller=logger.go:36 server_name=localhost remote_addr=[::1]:49268 msg="Getting server certificate"
level=debug time=2019-07-15T21:20:26.122415Z caller=logger.go:36 msg="Requesting new certificate from issuer"
level=debug time=2019-07-15T21:20:26.240165Z caller=logger.go:36 serial=80514697307960587646287223417136054196693349002 expiry=2019-07-18T21:20:26Z msg="New certificate issued"
level=debug time=2019-07-15T21:20:26.24021Z caller=logger.go:36 serial=80514697307960587646287223417136054196693349002 took=118.286421ms msg="Certificate found"
```

- Run the client with `make run-client-ca`

You need to get Vault's CA certificate first. Let's make an API call to get it and save it as `ca-vault.cert`. Do not confuse this file with `ca-cert`, which is the CA certificate of the issuer of the certificates in Vault's config; `tls_cert_file` and `tls_key_file`)

```bash
$ curl \
    --cacert ca.cert \
    https://localhost:8200/v1/pki/ca/pem \
    -o ca-vault.cert
```

- Validate Vault CA

Let's now run the client, we need to export the name of the Vault's CA certificate file as `CAFILE`.

$ export CAFILE="ca-vault.cert"
$ make run-client-ca
go run client/main.go -id 1 -file ca-vault.cert -mode 3
User found:  Nicolas
```