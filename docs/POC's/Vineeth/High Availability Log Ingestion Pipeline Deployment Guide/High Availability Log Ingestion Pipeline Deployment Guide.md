# High Availability Log Ingestion Pipeline Deployment Guide

## Overview
This document provides a step-by-step deployment guide for implementing a secure, highly available log ingestion pipeline integrated with Wazuh. The architecture uses Filebeat, Logstash, HAProxy, rsyslog, and Wazuh Manager nodes to securely collect, process, load balance, and forward log events.

The solution implements mutual TLS (mTLS) communication using a private Public Key Infrastructure (PKI) consisting of a Root Certificate Authority (CA), an Intermediate CA, node certificates, certificate chains, and Certificate Revocation Lists (CRLs). Certificates are generated with appropriate server, client, or combined server/client roles based on each component's communication requirements. 

Filebeat collects logs from source systems and forwards them securely to the Logstash nodes. Logstash processes and enriches incoming events before forwarding them through HAProxy. HAProxy provides load balancing, TLS termination and re-encryption, certificate validation, backend health checking, and high availability across the Wazuh Manager nodes. rsyslog receives the forwarded events over TLS and reliably delivers them to the local Wazuh Manager using persistent disk queues. 

The document covers certificate generation and distribution, installation and configuration of each component, TLS and mTLS configuration, load balancing, backend health checks, persistent queue configuration, service management, and deployment across multiple nodes.

## Architecture diagram
![Logstash](<../../../assets/images/POC's/Vineeth/High Availability Log Ingestion Pipeline Deployment Guide/Logstash.png>)

## 1. SSL certs generation guide
### CA workstation setup

This is done on any one of the 8 VMs.
Choose one VM to act as the CA workstation and perform all root and intermediate CA operations there.

### 1.1 Create the correct directory structure
Run the following commands on the selected CA workstation. Use sudo if required and keep private CA directories restricted.

```
# Root CA directories
mkdir -p /opt/mtls-pki/ca/{certs,crl,newcerts,private}
chmod 700 /opt/mtls-pki/ca/private

# Intermediate CA directories
mkdir -p /opt/mtls-pki/intermediate/{certs,crl,newcerts,private}
chmod 700 /opt/mtls-pki/intermediate/private

# Issued node certs (one subdir per VM)
mkdir -p /opt/mtls-pki/issued/{vm1-filebeat,vm2-logstash1,vm3-logstash2,vm4-logstash3,vm5-haproxy,vm6-rsyslog1,vm7-rsyslog2,vm8-rsyslog3}

# Scripts
mkdir -p /opt/mtls-pki/scripts

# Root CA database
touch /opt/mtls-pki/ca/index.txt
echo 1000 > /opt/mtls-pki/ca/serial
echo 1000 > /opt/mtls-pki/ca/crlnumber

# Intermediate CA database
touch /opt/mtls-pki/intermediate/index.txt
echo 2000 > /opt/mtls-pki/intermediate/serial
echo 2000 > /opt/mtls-pki/intermediate/crlnumber

# Verify
find /opt/mtls-pki -type d | sort
```
Verify the directory layout and permissions before continuing.

### 1.2 — Root CA config
Create the root CA OpenSSL config file before generating any certificates. This file defines CA behavior and certificate constraints.
Save as /opt/mtls-pki/ca/openssl-root.cnf:

```
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = /opt/mtls-pki/ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand
private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30
default_md        = sha256
name_opt          = ca_default
cert_opt          = ca_default
default_days      = 3650
preserve          = no
policy            = policy_strict

[ policy_strict ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
default_bits        = 4096
distinguished_name  = req_distinguished_name
string_mask         = utf8only
default_md          = sha256
x509_extensions     = v3_ca
prompt              = no

[ req_distinguished_name ]
countryName             = IN
stateOrProvinceName     = Telangana
localityName            = Hyderabad
organizationName        = MyOrg
organizationalUnitName  = IT Security
commonName              = MyOrg Root CA

[ v3_ca ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:true, pathlen:1
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:true, pathlen:0
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign

[ crl_ext ]
authorityKeyIdentifier = keyid:always
```
Double-check the file path and contents before moving on.

### 1.3 — Generate Root CA (ECDSA P-256)

```
# Generate Root CA private key — ECDSA P-256
openssl ecparam \
  -name prime256v1 \
  -genkey \
  -noout \
  -out /opt/mtls-pki/ca/private/ca.key.pem

# Encrypt the key at rest (you will be prompted for a passphrase — keep it safe)
openssl pkcs8 \
  -topk8 \
  -v2 aes-256-cbc \
  -in  /opt/mtls-pki/ca/private/ca.key.pem \
  -out /opt/mtls-pki/ca/private/ca.key.enc.pem

mv /opt/mtls-pki/ca/private/ca.key.enc.pem /opt/mtls-pki/ca/private/ca.key.pem
chmod 400 /opt/mtls-pki/ca/private/ca.key.pem

# Self-sign the Root CA cert (10 years)
openssl req \
  -config  /opt/mtls-pki/ca/openssl-root.cnf \
  -key     /opt/mtls-pki/ca/private/ca.key.pem \
  -new -x509 \
  -days 3650 \
  -sha256 \
  -extensions v3_ca \
  -out /opt/mtls-pki/ca/certs/ca.cert.pem

chmod 444 /opt/mtls-pki/ca/certs/ca.cert.pem

# Verify — you must see CA:TRUE and pathlen:1
openssl x509 -noout -text -in /opt/mtls-pki/ca/certs/ca.cert.pem | \
  grep -A4 "Basic Constraints"

```
Expected output:
```
X509v3 Basic Constraints: critical
    CA:TRUE, pathlen:1
```
### 1.4 — Intermediate CA config

Save as /opt/mtls-pki/intermediate/openssl-intermediate.cnf:

```
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = /opt/mtls-pki/intermediate
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand
private_key       = $dir/private/intermediate.key.pem
certificate       = $dir/certs/intermediate.cert.pem
crlnumber         = $dir/crlnumber
crl               = $dir/crl/intermediate.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 15
default_md        = sha256
name_opt          = ca_default
cert_opt          = ca_default
default_days      = 825
preserve          = no
policy            = policy_loose

[ policy_loose ]
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
default_bits        = 4096
distinguished_name  = req_distinguished_name
string_mask         = utf8only
default_md          = sha256
prompt              = no

[ req_distinguished_name ]
countryName             = IN
stateOrProvinceName     = Telangana
localityName            = Hyderabad
organizationName        = MyOrg
organizationalUnitName  = IT Security
commonName              = MyOrg Intermediate CA

[ server_cert ]
basicConstraints       = critical, CA:FALSE
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer
keyUsage               = critical, digitalSignature, keyEncipherment
extendedKeyUsage       = serverAuth

[ client_cert ]
basicConstraints       = critical, CA:FALSE
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer
keyUsage               = critical, digitalSignature, keyEncipherment
extendedKeyUsage       = clientAuth

[ server_client_cert ]
basicConstraints       = critical, CA:FALSE
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer
keyUsage               = critical, digitalSignature, keyEncipherment
extendedKeyUsage       = serverAuth, clientAuth

[ crl_ext ]
authorityKeyIdentifier = keyid:always
```
### 1.5 — Generate and sign the Intermediate CA

```
# Generate Intermediate CA key — ECDSA P-256
openssl ecparam \
  -name prime256v1 \
  -genkey \
  -noout \
  -out /opt/mtls-pki/intermediate/private/intermediate.key.pem

# Encrypt it
openssl pkcs8 \
  -topk8 -v2 aes-256-cbc \
  -in  /opt/mtls-pki/intermediate/private/intermediate.key.pem \
  -out /opt/mtls-pki/intermediate/private/intermediate.key.enc.pem

mv /opt/mtls-pki/intermediate/private/intermediate.key.enc.pem \
   /opt/mtls-pki/intermediate/private/intermediate.key.pem
chmod 400 /opt/mtls-pki/intermediate/private/intermediate.key.pem

# Generate CSR for the Intermediate CA
openssl req \
  -config /opt/mtls-pki/intermediate/openssl-intermediate.cnf \
  -key    /opt/mtls-pki/intermediate/private/intermediate.key.pem \
  -new -sha256 \
  -out /opt/mtls-pki/intermediate/certs/intermediate.csr.pem

# Root CA signs the Intermediate CA cert (5 years, pathlen:0)
openssl ca \
  -config     /opt/mtls-pki/ca/openssl-root.cnf \
  -extensions v3_intermediate_ca \
  -days       1825 \
  -notext -md sha256 \
  -in  /opt/mtls-pki/intermediate/certs/intermediate.csr.pem \
  -out /opt/mtls-pki/intermediate/certs/intermediate.cert.pem

chmod 444 /opt/mtls-pki/intermediate/certs/intermediate.cert.pem

# Build the CA chain file (intermediate + root, ordered)
cat /opt/mtls-pki/intermediate/certs/intermediate.cert.pem \
    /opt/mtls-pki/ca/certs/ca.cert.pem \
  > /opt/mtls-pki/intermediate/certs/ca-chain.cert.pem
chmod 444 /opt/mtls-pki/intermediate/certs/ca-chain.cert.pem

# Verify the intermediate against root — must say OK
openssl verify \
  -CAfile /opt/mtls-pki/ca/certs/ca.cert.pem \
  /opt/mtls-pki/intermediate/certs/intermediate.cert.pem

# Verify pathlen:0
openssl x509 -noout -text \
  -in /opt/mtls-pki/intermediate/certs/intermediate.cert.pem | \
  grep -A4 "Basic Constraints"
```
Expected:
```
X509v3 Basic Constraints: critical
    CA:TRUE, pathlen:0
```
### 1.6 — Generate CRL for both CAs

```
# Root CA CRL
openssl ca \
  -config /opt/mtls-pki/ca/openssl-root.cnf \
  -gencrl \
  -out /opt/mtls-pki/ca/crl/ca.crl.pem

# Intermediate CA CRL
openssl ca \
  -config /opt/mtls-pki/intermediate/openssl-intermediate.cnf \
  -gencrl \
  -out /opt/mtls-pki/intermediate/crl/intermediate.crl.pem

# Combine for distribution (HAProxy and Logstash consume this)
cat /opt/mtls-pki/intermediate/crl/intermediate.crl.pem \
    /opt/mtls-pki/ca/crl/ca.crl.pem \
  > /opt/mtls-pki/intermediate/crl/ca-chain.crl.pem

# Verify both CRLs
openssl crl -noout -text -in /opt/mtls-pki/ca/crl/ca.crl.pem | head -10
openssl crl -noout -text -in /opt/mtls-pki/intermediate/crl/intermediate.crl.pem | head -10
```
### 1.7 — Node certificate generation script
Save as /opt/mtls-pki/scripts/gen-cert.sh:
```
#!/bin/bash
# Usage: ./gen-cert.sh <hostname> <ip> <role>
# role: server | client | server_client
set -euo pipefail

HOSTNAME="$1"
IP="$2"
ROLE="$3"
DAYS=825     # Max enforced by Chrome/Apple — never exceed

INTDIR="/opt/mtls-pki/intermediate"
OUTDIR="/opt/mtls-pki/issued/${HOSTNAME}"
mkdir -p "${OUTDIR}"

echo ">>> Generating ECDSA P-256 key for ${HOSTNAME}"
openssl ecparam \
  -name prime256v1 \
  -genkey -noout \
  -out "${OUTDIR}/${HOSTNAME}.key.pem"
chmod 400 "${OUTDIR}/${HOSTNAME}.key.pem"

# Build SAN extension file — this is what fixes the "san section" error
# We use -extfile with openssl x509, NOT openssl ca -extensions
EXT_FILE="$(mktemp /tmp/san-XXXXXX.cnf)"
cat > "${EXT_FILE}" <<EXTEOF
[ req ]
distinguished_name = req_distinguished_name
prompt = no

[ req_distinguished_name ]
C  = IN
ST = Telangana
L  = Hyderabad
O  = MyOrg
CN = ${HOSTNAME}

[ v3_req ]
basicConstraints       = critical, CA:FALSE
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer
subjectAltName         = @alt_names
keyUsage               = critical, digitalSignature, keyEncipherment
EXTEOF

case "${ROLE}" in
  server)        echo "extendedKeyUsage = serverAuth"             >> "${EXT_FILE}" ;;
  client)        echo "extendedKeyUsage = clientAuth"             >> "${EXT_FILE}" ;;
  server_client) echo "extendedKeyUsage = serverAuth, clientAuth" >> "${EXT_FILE}" ;;
  *) echo "Unknown role: ${ROLE}. Use server|client|server_client"; exit 1 ;;
esac

cat >> "${EXT_FILE}" <<ALTEOF

[ alt_names ]
DNS.1 = ${HOSTNAME}
IP.1  = ${IP}
ALTEOF

echo ">>> Generating CSR"
openssl req \
  -new -sha256 \
  -key    "${OUTDIR}/${HOSTNAME}.key.pem" \
  -config "${EXT_FILE}" \
  -out    "${OUTDIR}/${HOSTNAME}.csr.pem"

echo ">>> Signing with Intermediate CA"
# Use openssl x509 -req + -extfile to avoid the [san] section issue entirely
openssl x509 -req \
  -in      "${OUTDIR}/${HOSTNAME}.csr.pem" \
  -CA      "${INTDIR}/certs/intermediate.cert.pem" \
  -CAkey   "${INTDIR}/private/intermediate.key.pem" \
  -CAserial "${INTDIR}/serial" \
  -days    "${DAYS}" \
  -sha256 \
  -extfile "${EXT_FILE}" \
  -extensions v3_req \
  -out     "${OUTDIR}/${HOSTNAME}.cert.pem"

chmod 444 "${OUTDIR}/${HOSTNAME}.cert.pem"

# Full chain: node cert + intermediate cert + root cert
cat "${OUTDIR}/${HOSTNAME}.cert.pem" \
    "${INTDIR}/certs/intermediate.cert.pem" \
    "/opt/mtls-pki/ca/certs/ca.cert.pem" \
  > "${OUTDIR}/${HOSTNAME}.chain.pem"

# Verify the signed cert against the full chain
openssl verify \
  -CAfile "${INTDIR}/certs/ca-chain.cert.pem" \
  "${OUTDIR}/${HOSTNAME}.cert.pem"

# Print SANs and dates for confirmation
openssl x509 -noout -subject -issuer -dates \
  -ext subjectAltName,extendedKeyUsage \
  -in "${OUTDIR}/${HOSTNAME}.cert.pem"

rm -f "${EXT_FILE}"
echo ">>> Done: ${OUTDIR}/"
ls -lh "${OUTDIR}/"
```
Give the permissions to it

```
chmod +x /opt/mtls-pki/scripts/gen-cert.sh
```
### 1.8 — Generate all 8 node certificates

```
cd /opt/mtls-pki/scripts

# VM1 — Filebeat: connects OUT to HAProxy → client only
./gen-cert.sh vm1-filebeat   10.0.1.11  client

# VM5 — HAProxy: receives from Filebeat (server) + connects to Logstash (client)
./gen-cert.sh vm5-haproxy    10.0.1.15  server_client

# VM6 — rsyslog1: Logstash connects TO it → server only
./gen-cert.sh vm6-rsyslog1   10.0.1.16  server

# VM7 — rsyslog2
./gen-cert.sh vm7-rsyslog2   10.0.1.17  server

# VM8 — rsyslog3
./gen-cert.sh vm8-rsyslog3   10.0.1.18  server
```

To Generate the certs in logstash need to change few lines in gen-cert.sh as below

From
```
echo ">>> Generating ECDSA P-256 key for ${HOSTNAME}"
openssl ecparam \
  -name prime256v1 \
  -genkey -noout \
  -out "${OUTDIR}/${HOSTNAME}.key.pem"
```
To
```
openssl genrsa \
-out "${OUTDIR}/${HOSTNAME}.key.pem" \
4096
```
Then generate the certs for logstash
```
# VM2 — Logstash1: accepts connections (server) + connects to rsyslog (client)
./gen-cert.sh vm2-logstash1  10.0.1.12  server_client

# VM3 — Logstash2
./gen-cert.sh vm3-logstash2  10.0.1.13  server_client

# VM4 — Logstash3
./gen-cert.sh vm4-logstash3  10.0.1.14  server_client
```

### 1.9 — Distribute certificates securely
Distribute the certificates securely

## 2. Installation of Filebeat

### 2.1 Installation
Follow the below steps to install the filebeat
```
sudo apt update
sudo apt install filebeat -y
sudo systemctl enable filebeat
systemctl status filebeat
```
### 2.2 Filebeat config

Take the backup of original filebeat.yml
```
cd /etc/filebeat/
mv filebeat.yml filebeat.yml-backup
```
Add the below content in filebeat.yml file
```
# ============================================================
# INPUTS
# ============================================================
filebeat.inputs:
  - type: filestream          
    id: app-log-stream       
    enabled: true
    paths:
      - /var/log/applog/app.log
    encoding: utf-8
    close.inactive: 2m        
    clean_inactive: 72h       
    clean_removed: true       
    prospector.scanner.scan_frequency: 5s

    fields:
      log_type: linux-auth
      tenant: tenantB
      source_app: fake_app_generator
    fields_under_root: true

# ============================================================
# DISK QUEUE  (replaces in-memory queue; survives Filebeat restart)
# ============================================================
queue.disk:
  path: "${path.data}/diskqueue"
  max_size: 10GB
  segment_size: 512MB         
  read_ahead: 1024            

# ============================================================
# OUTPUT — point ONLY at HAProxy VIP, not individual Logstash nodes
# ============================================================
output.logstash:
  hosts:
    - logstash_IP:5044
    - logstash_IP:5044
    - logstash_IP:5044
  loadbalance: true           
  worker: 4                   
  bulk_max_size: 1024         
  pipelining: 2               
  compression_level: 3       
  timeout: 30s                
  backoff.init: 1s
  backoff.max: 60s

  ssl:
    enabled: true
    certificate_authorities:
      - /etc/filebeat/mcerts/ca-chain.cert.pem
    certificate: /etc/filebeat/mcerts/filebeat01.crt
    key:         /etc/filebeat/mcerts/filebeat01.key
    verification_mode: full  
    supported_protocols:      
      - TLSv1.2
      - TLSv1.3
    cipher_suites:
      - ECDHE-ECDSA-AES-128-GCM-SHA256
      - ECDHE-ECDSA-AES-256-GCM-SHA384
      - ECDHE-ECDSA-CHACHA20-POLY1305
    renegotiation: never
```
change the path to logs path, replace the logstash_IP with 3 logstash ip's and relace the certs at exact directory as generated in the initial stage.

Then restart the filebeat

## 3 Installation of Logstash
### 3.1 Installation
Follow the below steps to install the logstash

Download and install the Public Signing Key:

```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg
```
You may need to install the apt-transport-https package on Debian before proceeding:
```
sudo apt-get install apt-transport-https
```
Save the repository definition to /etc/apt/sources.list.d/elastic-9.x.list:
```
echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-9.x.list
```

Run sudo apt-get update and the repository is ready for use. You can install it with:
```
sudo apt-get update && sudo apt-get install logstash
```
### 3.2 Logstash config
Add Logstash pipeline configuration and SSL settings here. Ensure the input, filter, and output sections match the deployed certificate locations and that Logstash is configured to use the same CA chain as Filebeat.
```
cd /etc/logstash/conf.d/
vi beats.conf
```
add the below content here
```
input {
  beats {
    port => 5044
    ssl_enabled => true
    ssl_certificate => "/etc/logstash/mcerts/logstash.crt"
    ssl_key         => "/etc/logstash/mcerts/logstash.key"
    ssl_certificate_authorities => [
      "/etc/logstash/mcerts/ca-chain.cert.pem"
    ]
    ssl_client_authentication => "required"     # ✓ KEEP — Logstash is server here
    ssl_supported_protocols => ["TLSv1.2", "TLSv1.3"]
    ssl_cipher_suites => [
      "TLS_AES_256_GCM_SHA384",
      "TLS_AES_128_GCM_SHA256",
      "TLS_CHACHA20_POLY1305_SHA256",
      "TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384",
      "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256"
    ]
  }
}

filter {
  mutate {
    add_field => {
      "application" => "app_generator"
      "environment" => "lab"
    }
    # Remove internal tags that add noise
    remove_tag => ["beats_input_codec_plain_applied"]
  }
  mutate {
    add_field => { "logstash_node" => "logstash-2" }  # change on node 2
  }

}


output {
  tcp {
    host => "ha_proxy_ip_address"
    port => 9065

    ssl_enabled => true
    ssl_certificate => "/etc/logstash/mcerts/logstash.crt"
    ssl_key         => "/etc/logstash/mcerts/logstash.key"

    ssl_certificate_authorities => [
      "/etc/logstash/mcerts/ca-chain.cert.pem"
    ]

    ssl_verification_mode => "full"

    codec => json_lines
  }
}
```
Then restart the logstash

Repeat the same process for logstash 2 and 3

## 4 Installation of HAProxy
### 4.1 Installation
Follow the below steps to install the HAProxy

Run:
```
sudo apt update
sudo apt install haproxy -y
```
Verify the installation:
```
haproxy -v
```
Check the service:
```
sudo systemctl status haproxy
```
Enable HAProxy to start automatically after reboot:
```
sudo systemctl enable haproxy
```
Start HAProxy:
```
sudo systemctl start haproxy
```

### 4.1 HAProxy configuration

The main configuration file is:
```
/etc/haproxy/haproxy.cfg

cp -r haproxy.cfg haproxy.cfg-backup
```
Paste the below content and change the IP address 
```
global
    log /dev/log local0
    log /dev/log local1 notice

    external-check
    insecure-fork-wanted
    spread-checks 5

    daemon
    maxconn 100000
    tune.ssl.default-dh-param 4096
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305

    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

    user haproxy
    group haproxy

    stats socket /run/haproxy/admin.sock mode 660 level admin

defaults
    log global

    mode tcp

    option tcplog
    option dontlognull
    option tcpka

    retries 3

    timeout connect 10s
    timeout client 5m
    timeout server 5m
    timeout check 10s
#############################################
# Agent Registration
#############################################

frontend wazuh_registration_front
    bind *:1515
    mode tcp
    default_backend wazuh_registration_back

backend wazuh_registration_back
    mode tcp

    option tcp-check

    server wazuh-master  wazuh_master_IP:1515 check inter 5s fall 3 rise 2
    server wazuh-worker1 wazuh_worker1_IP:1515 check inter 5s fall 3 rise 2 backup
    server wazuh-worker2 wazuh_worker2_IP:1515 check inter 5s fall 3 rise 2 backup

#############################################
# Agent Events
#############################################

frontend wazuh_events_front
    bind *:1514
    mode tcp
    default_backend wazuh_events_back

backend wazuh_events_back
    mode tcp
    balance leastconn

    option tcp-check

    server wazuh-master  wazuh_master_IP:1514 check inter 5s fall 3 rise 2
    server wazuh-worker1 wazuh_worker1_IP:1514 check inter 5s fall 3 rise 2
    server wazuh-worker2 wazuh_worker2_IP:1514 check inter 5s fall 3 rise 2

#############################################
# Custom Syslog / Logstash -> Wazuh
#############################################
frontend rsyslog_front
    bind ha_proxy_ip_address:9065 ssl crt /etc/haproxy/mcerts/haproxy.pem ca-file /etc/haproxy/mcerts/ca-chain.cert.pem crl-file /etc/haproxy/mcerts/ca-chain.crl.pem verify required
    mode tcp
    default_backend rsyslog_back

backend rsyslog_back
    mode tcp
    balance leastconn
    option external-check
    external-check command /usr/local/bin/check_wazuh.sh
    default-server inter 5s fall 2 rise 1 on-marked-down shutdown-sessions

    server manager wazuh_master_IP:1549 ssl crt /etc/haproxy/mcerts/haproxy.pem ca-file /etc/haproxy/mcerts/ca-chain.cert.pem verify required check
    server worker1 wazuh_worker1_IP:1549 ssl crt /etc/haproxy/mcerts/haproxy.pem ca-file /etc/haproxy/mcerts/ca-chain.cert.pem verify required check
    server worker2 wazuh_worker2_IP:1549 ssl crt /etc/haproxy/mcerts/haproxy.pem ca-file /etc/haproxy/mcerts/ca-chain.cert.pem verify required check
    
```
In /usr/local/bin/check_wazuh.sh add the below content by replacing the IP address
```
#!/bin/bash
case "$HAPROXY_SERVER_NAME" in
  manager)
    HOST="wazuh_master_IP"
    ;;
  worker1)
    HOST="wazuh_worker1_IP"
    ;;
  worker2)
    HOST="wazuh_worker2_IP"
    ;;
  *)
    exit 1
    ;;
esac

/usr/bin/timeout 3 /bin/bash -c "</dev/tcp/${HOST}/9065" >/dev/null 2>&1
exit $?
```
Then restart the HAProxy service

## 5 Installation of Rsyslog 
### 5.1 Installation
Follow the below steps to install rsyslog
Install rsyslog:
```
sudo apt update
sudo apt install rsyslog rsyslog-gnutls -y
```
Enable and start the service:
```
sudo systemctl enable --now rsyslog
```
Verify the service:
```
sudo systemctl status rsyslog
```
### 5.2 Rsyslog configuration

Create a file at /etc/rsyslog.d

vi /etc/rsyslog.d/example.conf

Add the below content
```
module(load="imtcp" StreamDriver.Name="gtls"
       StreamDriver.Mode="1"
       StreamDriver.AuthMode="x509/certvalid")

global(
  DefaultNetstreamDriver="gtls"
  DefaultNetstreamDriverCAFile="/etc/rsyslog.d/certs/ca-chain.cert.pem"
  DefaultNetstreamDriverCertFile="/etc/rsyslog.d/certs/rsyslog.crt"
  DefaultNetstreamDriverKeyFile="/etc/rsyslog.d/certs/rsyslog.key"
)

input(type="imtcp" port="1549"
      StreamDriver.AuthMode="x509/certvalid"
      StreamDriver.Mode="1")

*.* /var/log/remote/remote.log

template(name="RawJson" type="string"
string="%msg%\n")

action(
  type="omfwd"

  target="127.0.0.1"
  port="9065"
  protocol="tcp"

  Template="RawJson"

  queue.type="LinkedList"
  queue.filename="wazuh_forward"
  queue.maxdiskspace="10g"
  queue.saveonshutdown="on"
  queue.size="100000"

  action.resumeRetryCount="-1"
  action.resumeInterval="10"
)
```
Then restart the rsyslog service

Repeat the same process in worker1 vm and worker2 vm