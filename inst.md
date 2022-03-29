#!/bin/bash

# 720h = 30 дней
# 8760h = 1 год
# 43800h = 5 лет
# cert: 87600h = 10 лет
# int: 175200h = 20 лет
# root: 262800h = 30 лет

ca_street="Red Square 1"
ca_org="Horns and Hooves LLC"
ca_ou="IT"
ca_url="https://vault.example.com"

# enable Vault PKI secret
vault secrets enable \
  -path=pki_root_ca \
  -description="PKI Root CA" \
  -max-lease-ttl="262800h" \
  pki

# generate root CA
vault write -format=json pki_root_ca/root/generate/internal \
  common_name="Root Certificate Authority" \
  country="Russian Federation" \
  locality="Moscow" \
  street_address="$ca_street" \
  postal_code="101000" \
  organization="$ca_org" \
  ou="$ca_ou" \
  ttl="262800h" > pki-root-ca.json

# save the certificate
cat pki-root-ca.json | jq -r .data.certificate > rootCA.pem

# publish urls for the root ca
vault write pki_root_ca/config/urls \
  issuing_certificates="$ca_url/v1/pki_root_ca/ca" \
  crl_distribution_points="$ca_url/v1/pki_root_ca/crl"

# enable Vault PKI secret
vault secrets enable \
  -path=pki_int_ca \
  -description="PKI Intermediate CA" \
  -max-lease-ttl="175200h" \
  pki

# create intermediate CA with common name example.com and save the CSR
vault write -format=json pki_int_ca/intermediate/generate/internal \
  common_name="Intermediate Certificate Authority" \
  country="Russian Federation" \
  locality="Moscow" \
  street_address="$ca_street" \
  postal_code="101000" \
  organization="$ca_org" \
  ou="$ca_ou" \
  ttl="175200h" | jq -r '.data.csr' > pki_intermediate_ca.csr

# send the intermediate CA's CSR to the root CA
vault write -format=json pki_root_ca/root/sign-intermediate csr=@pki_intermediate_ca.csr \
  country="Russia Federation" \
  locality="Moscow" \
  street_address="$ca_street" \
  postal_code="101000" \
  organization="$ca_org" \
  ou="$ca_ou" \
  format=pem_bundle \
  ttl="175200h" | jq -r '.data.certificate' > intermediateCA.cert.pem

# publish the signed certificate back to the Intermediate CA
vault write pki_int_ca/intermediate/set-signed \
  certificate=@intermediateCA.cert.pem

# publish the intermediate CA urls ???
vault write pki_int_ca/config/urls \
  issuing_certificates="$ca_url/v1/pki_int_ca/ca" \
  crl_distribution_points="$ca_url/v1/pki_int_ca/crl"

# create a role example-dot-com-server
vault write pki_int_ca/roles/example-dot-com-server \
  country="Russia Federation" \
  locality="Moscow" \
  street_address="$ca_street" \
  postal_code="101000" \
  organization="$ca_org" \
  ou="$ca_ou" \
  allowed_domains="example.com" \
  allow_subdomains=true \
  max_ttl="87600h" \
  key_bits="2048" \
  key_type="rsa" \
  allow_any_name=false \
  allow_bare_domains=false \
  allow_glob_domain=false \
  allow_ip_sans=true \
  allow_localhost=false \
  client_flag=false \
  server_flag=true \
  enforce_hostnames=true \
  key_usage="DigitalSignature,KeyEncipherment" \
  ext_key_usage="ServerAuth" \
  require_cn=true

# create a role example-dot-com-client
vault write pki_int_ca/roles/example-dot-com-client \
  country="Russia Federation" \
  locality="Moscow" \
  street_address="$ca_street" \
  postal_code="101000" \
  organization="$ca_org" \
  ou="$ca_ou" \
  allow_subdomains=true \
  max_ttl="87600h" \
  key_bits="2048" \
  key_type="rsa" \
  allow_any_name=true \
  allow_bare_domains=false \
  allow_glob_domain=false \
  allow_ip_sans=false \
  allow_localhost=false \
  client_flag=true \
  server_flag=false \
  enforce_hostnames=false \
  key_usage="DigitalSignature" \
  ext_key_usage="ClientAuth" \
  require_cn=true

# Create cert, 5 years
vault write -format=json pki_int_ca/issue/example-dot-com-server \
  common_name="vault.example.com" \
  alt_names="vault.example.com" \
  ttl="43800h" > vault.example.com.crt

# save cert
cat vault.example.com.crt | jq -r .data.certificate > vault.example.com.crt.pem
cat vault.example.com.crt | jq -r .data.issuing_ca >> vault.example.com.crt.pem
cat vault.example.com.crt | jq -r .data.private_key > vault.example.com.crt.key
