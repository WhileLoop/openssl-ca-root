### https://jamielinux.com/docs/openssl-certificate-authority/index.html

# Setup root CA
mkdir -p root/ca/certs
mkdir -p root/ca/crl
mkdir -p root/ca/newcerts
mkdir -p root/ca/private

chmod 700 root/ca/private
touch root/ca/index.txt
echo 1000 > root/ca/serial

# Create the root key
openssl genrsa -aes256 -out root/ca/private/ca.key.pem 4096
chmod 400 root/ca/private/ca.key.pem

# Create the root certificate
openssl req -config root/ca/openssl.cnf \
  -key root/ca/private/ca.key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca \
  -out root/ca/certs/ca.cert.pem

# Verify the root certificate
openssl x509 -noout -text -in root/ca/certs/ca.cert.pem


# Setup intermediate CA
mkdir -p root/ca/intermediate/certs
mkdir -p root/ca/intermediate/crl
mkdir -p root/ca/intermediate/newcerts
mkdir -p root/ca/intermediate/private

chmod 700 root/ca/intermediate/private
touch root/ca/intermediate/index.txt
echo 1000 > root/ca/intermediate/serial
echo 1000 > root/ca/intermediate/crlnumber

# Create the intermediate key
openssl genrsa -aes256 -out root/ca/intermediate/private/intermediate.key.pem 4096
chmod 400 root/ca/intermediate/private/intermediate.key.pem

# Create the intermediate certificate
openssl req -config root/ca/intermediate/openssl.cnf -new -sha256 \
      -key root/ca/intermediate/private/intermediate.key.pem \
      -out root/ca/intermediate/csr/intermediate.csr.pem

openssl ca -config root/ca/openssl.cnf -extensions v3_intermediate_ca \
      -days 3650 -notext -md sha256 \
      -in root/ca/intermediate/csr/intermediate.csr.pem \
      -out root/ca/intermediate/certs/intermediate.cert.pem

chmod 444 root/ca/intermediate/certs/intermediate.cert.pem


# Verify the intermediate certificate
openssl x509 -noout -text -in root/ca/intermediate/certs/intermediate.cert.pem
openssl verify -CAfile root/ca/certs/ca.cert.pem root/ca/intermediate/certs/intermediate.cert.pem

# Create the certificate chain file
cat root/ca/intermediate/certs/intermediate.cert.pem root/ca/certs/ca.cert.pem > root/ca/intermediate/certs/ca-chain.cert.pem

# Create a user private key
openssl genrsa -out root/ca/intermediate/private/domain.com.key.pem 2048
chmod 400 root/ca/intermediate/private/domain.com.key.pem

# Create a user certificate signing request
openssl req -config root/ca/intermediate/openssl.cnf \
      -key root/ca/intermediate/private/domain.com.key.pem \
      -new -sha256 -out root/ca/intermediate/csr/domain.com.csr.pem

# Create a user certificate
openssl ca -config /root/ca/intermediate/openssl.cnf \
      -extensions server_cert -days 375 -notext -md sha256 \
      -in /root/ca/intermediate/csr/domain.com.csr.pem \
      -out /root/ca/intermediate/certs/domain.com.cert.pem

# Verify the certificate
openssl x509 -noout -text -in root/ca/intermediate/certs/domain.com.cert.pem

# Verify chain of trust
openssl verify -CAfile root/ca/intermediate/certs/ca-chain.cert.pem \
      root/ca/intermediate/certs/domain.com.cert.pem

# Create PFX with chain
openssl pkcs12 -export -out root/certificate.pfx -inkey root/ca/intermediate/private/domain.com.key.pem \
      -in root/ca/intermediate/certs/domain.com.cert.pem \
      -certfile root/ca/intermediate/certs/ca-chain.cert.pem


# Extract encrypted key from PFX
openssl pkcs12 -in root/certificate.pfx -nocerts -out root/certificate-key-encrypted.pem

# Decrypt key
openssl rsa -in root/certificate-key-encrypted.pem -out root/certificate-key-decrypted.pem


# Extract server cert
openssl pkcs12 -in root/certificate.pfx -clcerts -nokeys -out root/server.crt

# Extract chain cert
openssl pkcs12 -in root/certificate.pfx -cacerts -nokeys -out root/server-ca.crt

# Extract server and chain cert
openssl pkcs12 -in root/certificate.pfx -nokeys -out root/bundle.crt
