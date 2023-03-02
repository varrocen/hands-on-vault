# PKI Secrets Engine

 * [Scenario](#scenario)
 * [Generate root CA](#root-ca)
 * [Generate intermediate CA](#intermediate-ca)
 * [Request certificates](#request-certificates)
 * [Revoke certificates](#revoke-certificates)
 * [Rotate Root CA](#rotate-root-ca)

<a name="scenario"/>

## Scenario

In this tutorial, you are going to first generate a self-signed root certificate.

Then you are going to generate an intermediate certificate that is signed by the root. You can then create a role and generate a certificate for the `dev.example.com` domain.

```mermaid
  flowchart TD;
      A[Root Certificate - example.com]-- sign -->B;
      B-- reference -->A
      B[Intermediate Certificate - example.com]-- sign -->C;
      C-- reference -->B;
      C[Identy Certificate - dev.example.com];
```

<a name="root-ca"/>

## Generate root CA

Enable the pki secrets engine at the pki path:

```
vault secrets enable pki
```

Tune the pki secrets engine to issue certificates with a maximum time-to-live (TTL) of 8760h hours (1 year).

```
vault secrets tune -max-lease-ttl=8760h pki
```

Generate the root CA, give it an issuer name, and save its certificate in the file `root-ca-2023.crt`:

```
vault write -field=certificate pki/root/generate/internal \
     common_name="Root CA" \
     issuer_name="root-ca-2023" \
     ttl=8760h > root-ca-2023.crt
```

Read the root CA certificate:

```
openssl x509 -in root-ca-2023.crt -noout -text
```

List the issuer information for the root CA:

```
vault list pki/issuers/
```

You can read the issuer with its ID to get the certificates and other metadata about the issuer:

```
vault read pki/issuer/<id>
```

A role is a logical name that maps to a policy used to generate those credentials:

```
vault write pki/roles/root-ca-2023 \
    allowed_domains=example.com \
    allow_subdomains=true
```

Configure the chain CA and CRL URLs:

```
vault write pki/config/urls \
     issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
     crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
```

Check insuing certificates url:

```
curl $VAULT_ADDR/v1/pki/ca --output ca.der
openssl x509 -inform DER -in ca.der -outform PEM -out ca.pem
openssl x509 -in ca.pem -noout -text
```

Check CRL distribution points:

```
curl $VAULT_ADDR/v1/pki/crl --output crl.der
openssl crl -inform DER -in crl.der -outform PEM -out crl.pem
openssl crl -in crl.pem -noout -text
```

<a name="intermediate-ca"/>

## Generate intermediate CA


Enable the pki secrets engine at the pki_int path:

```
vault secrets enable -path=pki_int pki
```

Tune the pki_int secrets engine to issue certificates with a maximum time-to-live (TTL) of 8760h hours (1 year).

```
vault secrets tune -max-lease-ttl=8760h pki_int
```

Execute the following command to generate an intermediate and save the CSR as `intermediate-ca.csr`:

```
vault write -format=json pki_int/intermediate/generate/internal \
     common_name="Intermediate CA" \
     issuer_name="intermediate-ca" \
     | jq -r '.data.csr' > intermediate-ca.csr
```

Read the intermediate CA certificate signing request:

```
openssl req -in intermediate-ca.csr -noout -text
```

Sign the intermediate certificate with the root CA private key, and save the generated certificate as `intermediate-ca.pem`.

```
vault write -format=json pki/root/sign-intermediate \
     issuer_ref="root-ca-2023" \
     csr=@intermediate-ca.csr \
     format=pem_bundle \
     ttl="8760h" \
     | jq -r '.data.certificate' > intermediate-ca.pem
```

Once the CSR is signed and the root CA returns a certificate, it can be imported back into Vault:

```
vault write pki_int/intermediate/set-signed certificate=@intermediate-ca.pem
```

Read the intermediate CA certificate:

```
openssl x509 -in intermediate-ca.pem -noout -text
```

Create a role for the intermediate CA:

```
vault write pki_int/roles/intermediate-ca \
     issuer_ref="$(vault read -field=default pki_int/config/issuers)" \
     allowed_domains="example.com" \
     allow_subdomains=true \
     max_ttl="72h"
```

Configure the chain CA and CRL URLs:

```
vault write pki_int/config/urls \
     issuing_certificates="$VAULT_ADDR/v1/pki_int/ca" \
     crl_distribution_points="$VAULT_ADDR/v1/pki_int/crl"
```

<a name="request-certificates"/>

## Request certificates

Execute the following command to request a new certificate for the `dev.example.com` domain:

```
vault write pki_int/issue/intermediate-ca common_name="dev.example.com" ttl="24h"
```

Execute the following command to request a new certificate for the `dev.wrong-example.com` domain (dont't work):

```
vault write pki_int/issue/intermediate-ca common_name="dev.wrong-example.com" ttl="24h"
```

<a name="revoke-certificates"/>

## Revoke certificates

<a name="rotate-root-ca"/>

## Rotate Root CA
