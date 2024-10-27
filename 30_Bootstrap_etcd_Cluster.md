# Setup an External ETCD Cluster
In this tutorial we will configure a six-node TLS enabled `etcd` high availibility cluster, that can act as an external datastore for a Kubernetes H.A. Cluster 😉

|Role|FQDN|IP|OS|Kernel|RAM|vCPU|KVM Host|
|----|----|----|----|----|----|----|----|
|etcd Database|k8s0etcd1.kloud.lan|10.103.0.101|Ubuntu 24.04.1|6.11.0|4G|2|pve1|
|etcd Database|k8s0etcd2.kloud.lan|10.103.0.102|Ubuntu 24.04.1|6.11.0|4G|2|pve1|
|etcd Database|k8s0etcd3.kloud.lan|10.103.0.103|Ubuntu 24.04.1|6.11.0|4G|2|pve1|
|etcd Database|k8s0etcd4.kloud.lan|10.103.0.201|Ubuntu 24.04.1|6.11.0|4G|2|pve2|
|etcd Database|k8s0etcd5.kloud.lan|10.103.0.202|Ubuntu 24.04.1|6.11.0|4G|2|pve2|
|etcd Database|k8s0etcd6.kloud.lan|10.103.0.203|Ubuntu 24.04.1|6.11.0|4G|2|pve2|

![ETCD Cluster](images/k8s-cluster.jpg)

> [!NOTE]  
> Everything from here is done on `k8s1bastion1` in the directory `~/k8s0/`
> The number of recommended `etcd` nodes in a cluster for H.A., is three or five. I did six because I like symmetry.

## Prerequisite
- You must have DNS resolution. Some scripts in this tutorial use the DNS to find IP addresses of a host.
- You must have the "DNS Search" domain configured correctly. Some scripts use the command `dig +short +search <hostname>`, again to find IP addresses of hosts.
- You must have `openssl` or any other tool to generate TLS certificates.
- SSH to the `etcd` servers without password.

# SSH Key (Optional)
Push the public SSH key of your bastion host to every `etcd` VMs. This will give you the ability to SSH without needing a password. Run the commands from you bastion host. This script expect your SSH key to be named `id_ed25519.pub`.
```sh
DOMAIN="kloud.lan"
unset arrVMs
arrVMs=(
  "k8s0etcd1"
  "k8s0etcd2"
  "k8s0etcd3"
  "k8s0etcd4"
  "k8s0etcd5"
  "k8s0etcd6"
  "k8s0etcd1.${DOMAIN}"
  "k8s0etcd2.${DOMAIN}"
  "k8s0etcd3.${DOMAIN}"
  "k8s0etcd4.${DOMAIN}"
  "k8s0etcd5.${DOMAIN}"
  "k8s0etcd6.${DOMAIN}"
)
# This is the password to access the ETCD servers through SSH
export MYPASSWORD="PASSWORD"

for VM in "${arrVMs[@]}"
do
  sshpass -p ${MYPASSWORD} ssh-copy-id -o StrictHostKeyChecking=no -i ~/.ssh/id_ed25519.pub ${VM}
  ssh-keyscan ${VM} >> ~/.ssh/known_hosts
  printf "New SSH key for %s\n" "${VM}"
done
```

# Terminal Multiplexer
I'm using `tmux` to access all the `etcd` VMs at the same time. Create the following script to start a session on each VM:
```sh
FILE="k8s0-etcd.sh"
cat <<'EOF' > ${FILE}
#!/bin/bash

SERVERS=( \
  k8s0etcd1 \
  k8s0etcd2 \
  k8s0etcd3 \
  k8s0etcd4 \
  k8s0etcd5 \
  k8s0etcd6
)

split_list=()
for NODE in "${SERVERS[@]:1}"; do
    split_list+=( split-pane ssh "${NODE}" ';' )
done

tmux new-session ssh "${SERVERS[0]}" ';' \
    "${split_list[@]}" \
    select-layout tiled ';' \
    set-option -w synchronize-panes
EOF
chmod u=rwx,g=r,o=r ${FILE}
```

# Download the binaries
Install the latest version of the `etcd` binaries on **each** of the Linux `etcd` node. Use `tmux` to SSH on every `etcd` nodes simultaneously.

> [!NOTE]  
> `etcdctl` is a command line tool for interacting with the `etcd` database in a cluster.

## Install `ETCD` from binaries
Download the latest `etcd` binaries from GitHub. Feel free to specify the version if you want a specific version and not the latest.
```sh
ARCH=$(dpkg --print-architecture)
ETCD_VER=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest|grep tag_name | cut -d '"' -f 4)
printf "ETCD version is: [%s]\n" "${ETCD_VER}"
curl -LO https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-${ARCH}.tar.gz
tar xvf etcd-${ETCD_VER}-linux-${ARCH}.tar.gz
cd etcd-${ETCD_VER}-linux-${ARCH}
sudo install etcd etcdctl etcdutl -m 755 -o root -g adm /usr/local/bin/
# Cleanup
cd ..
rm -rf etcd-${ETCD_VER}-linux-${ARCH}*
unset ETCD_VER
```

> [!IMPORTANT]  
> This step above **MUST** be executed on every `etcd` server.
> later in this tutorial you will be installing the utilities `etcdctl` and `etcdutl` on your `bastion` host.

## Verify installation (Optional)
Check the binaries you just installed:
```sh 
etcd --version
etcdctl version
etcdutl version
# Close all terminals (tmux)
exit
```

# Generate and Distribute TLS Certificates
> [!NOTE]  
> The following can be done on any server. I used one of my `bastion` host.

We will use the `openssl` tool to generate our own CA and all the `etcd` server certificates and keys. You can use your own certificate manager.

## Define Variables
Define all the variables needed for the scripts below. Make sure you execute the scripts in the same window as the one you defined the variables.
```sh
unset ETCD_NODES
ETCD_NODES=(\
k8s0etcd1 \
k8s0etcd2 \
k8s0etcd3 \
k8s0etcd4 \
k8s0etcd5 \
k8s0etcd6
)
```

## Generate Private CA
This script will generate a CA root certificate with its private key. This CA will be used to sign the certificate for each of the `etcd` servers. You can use your own CA if you already have one.

```sh
dnsDomain="kloud.lan"
CA_CERT="etcd-ca"

printf "\nMaking ETCD Root CA...\n"
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:secp521r1 -pkeyopt ec_param_enc:named_curve -out ${CA_CERT}.key

openssl req -new -sha256 -x509 -key ${CA_CERT}.key -days 7305 \
-subj "/C=CA/ST=QC/L=Montreal/O=ETCDRootCA/OU=${CA_CERT}/CN=etcd.${dnsDomain}" \
-addext "subjectAltName = DNS:localhost,DNS:*.localhost,DNS:${CA_CERT}.${dnsDomain},IP:127.0.0.1" \
-addext "basicConstraints = critical,CA:TRUE" \
-addext "keyUsage = critical, digitalSignature, cRLSign, keyCertSign" \
-addext "subjectKeyIdentifier = hash" \
-addext "authorityKeyIdentifier = keyid:always, issuer" \
-addext "authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp" \
-addext "crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8080/crl/Intermediate-CA.crl" \
-out ${CA_CERT}.crt

# Print the certificate
openssl x509 -text -noout -in ${CA_CERT}.crt
```

This results in two files:
- The file `etcd-ca.crt` is the CA certificate
- The file `etcd-ca.key` is the CA private key

## Generate Node Certificates
You could use the same client certificate of every `etcd` node and it will work fine. I've decided to use different certificate for each node. This following will generate the certificate and key for every `etcd` node in the cluster. Delete the `csr` files as they are not needed anymore:

```sh
cat <<'EOF' > gen_cert.sh
#!/bin/bash

# Generate a new private/public key pair and a certificate for a server given a CA certificate.
#
# ***** This script assume the following: *****
#     Certificate filename: <name>.crt
#     CSR filename:         <name>.csr
#     Private key filename: <name>.key
#
# This is a bit dirty, but helps when you want to generate lots of certificates for testing.
#
# HOWTO use it: ./gen_cert.sh <HOSTNAME> <IP ADDRESS> <CA CERTIFICATE NAME>
#
# This will generate the following files:
#  -rw-r--r--   1 username  staff  1216 01 Jan 00:00 server.crt >> certificate
#  -rw-r--r--   1 username  staff   976 01 Jan 00:00 server.csr >> CSR
#  -rw-------   1 username  staff   302 01 Jan 00:00 server.key >> key pair
#
# Works with 'bash' and 'zsh' shell on macOS and Linux. Make sure you have OpenSSL *** in your PATH ***.
#
# *WARNING*
# This script was made for educational purposes ONLY.
# USE AT YOUR OWN RISK!"
DOMAIN='kloud.lan'
EXTRA_SAN=''

if [[ $# -lt 3 || $# -gt 4 ]]
then
   printf "\nUsage: %s <HOSTNAME> <IP ADDRESS> <CA CERTIFICATE NAME>\n" $(basename $0)
   printf "Ex.: %s k8s1etcd1 10.103.1.101 etcd-ca\n\n" $(basename $0)
   exit -1
fi

# Undocumented argument 😉
EXTRA_SAN=$4

# ---------------------------------------------------------------
### Generate the CSR with an RSA or ECC private key

printf "Generate the CSR with private key for [${1}] ...\n"
openssl ecparam -name prime256v1 -genkey -out ${1}.key
# Comment the line above and uncomment the line below if you want an RSA key
# openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out ${1}.key

openssl req -new -sha256 -key ${1}.key -subj "/C=CA/ST=QC/L=Montreal/O=${1}/OU=IT/CN=${1}.$DOMAIN" \
-addext "subjectAltName = DNS:localhost,DNS:*.localhost,DNS:${1},DNS:${1}.$DOMAIN,IP:127.0.0.1,IP:${2}${EXTRA_SAN}" \
-addext "basicConstraints = CA:FALSE" \
-addext "extendedKeyUsage = clientAuth,serverAuth" \
-addext "subjectKeyIdentifier = hash" \
-addext "keyUsage = nonRepudiation, digitalSignature, keyEncipherment, keyAgreement, dataEncipherment" \
-addext "authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp" \
-addext "crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8080/crl/Intermediate-CA.crl" \
-out ${1}.csr

# ---------------------------------------------------------------
### Sign the CSR with the CA and keep all the extension from the CSR
printf "Signing the CSR with the CA (generates the certificate) for [${1}] ...\n"
openssl x509 -req -sha256 -days 365 -in ${1}.csr -CA ${3}.crt -CAkey ${3}.key -CAcreateserial -copy_extensions copyall -out ${1}.crt &>/dev/null

# To verify that etcd node certificate ${1}.crt is the CA ${3}.crt:
# openssl verify -no-CAfile -no-CApath -partial_chain -trusted %{3}.crt ${1}.crt

# ---------------------------------------------------------------
### Verification of certificate and private key. The next 2 checksum must be identical to validate that the certificate and private key are related!

CRT_PUB=$(openssl x509 -pubkey -in ${1}.crt -noout | openssl sha256 | awk -F '= ' '{print $2}')
KEY_PUB=$(openssl pkey -pubout -in ${1}.key | openssl sha256 | awk -F '= ' '{print $2}')

if [[ "${CRT_PUB}" != "${KEY_PUB}" ]]
then
   printf "ERROR: Public Key for certificate [%s] doesn't match Public Key in [%s].\n\n" "${1}.crt" "${1}.key"
   exit -1
else
   printf "SUCCESS: Public Key for certificate [%s] does match Public Key in [%s].\n\n" "${1}.crt" "${1}.key"
   exit 0
fi
EOF
# Give rwx permission to owner, read permissions to all other users:
chmod u=rwx,g=r,o=r gen_cert.sh
```

```sh
for ETCD in "${ETCD_NODES[@]}"
do
  ./gen_cert.sh "${ETCD}" "$(dig +short +search ${ETCD})" "${CA_CERT}"
done
# Generate the certificate for the master node
API="k8s1api"
./gen_cert.sh ${API} $(dig +search +short ${API}) ${CA_CERT}
rm -f *.csr
```

I copied the master node certificate and key to `apiserver-etcd-client.{crt,key}` since they will be client certificates on each of the K8s Master node to authenticate on any `etcd` server.
```sh
cp ${API}.crt apiserver-etcd-client.crt
cp ${API}.key apiserver-etcd-client.key
```

> [!IMPORTANT]  
> Keep those files as they will be required on all K8s master nodes.

The last command is for our Kubernetes Control Plane nodes:

This results in two files per `etcd` node. The `.crt` is the certificate and the `.key` is the private key:
- k8s0etcd1.crt
- k8s0etcd1.key
- k8s0etcd2.crt
- k8s0etcd2.key
- k8s0etcd3.crt
- k8s0etcd3.key
- k8s0etcd4.crt
- k8s0etcd4.key
- k8s0etcd5.crt
- k8s0etcd5.key
- k8s0etcd6.crt
- k8s0etcd6.key
- k8s1api.crt
- k8s1api.key
- apiserver-etcd-client.crt
- apiserver-etcd-client.key

> [!IMPORTANT]  
> The private keys are not encrypted as `etcd` needs a non-encrypted `pem` file.

At this point, we have the certificates and keys generated for the CA and all `etcd` nodes. Every node certifcate has a SAN that includes:
- hostname
- FQDN
- IP address

# Distribute Certificates
We need to to distribute the CA certificate, the server certificate/key to each `etcd` node in the cluster.

The following commands must be executed **from** the server where the `etcd` certificates were generated. In my case, it's on my bastion host. The script below assume you have passwordless SSH access to every `etcd` server.
```sh
for ETCD in "${ETCD_NODES[@]}"
do
  TLS="${CA_CERT}.crt ${ETCD}.crt ${ETCD}.key"
  printf "Copying files [%s] on [%s]\n" "${TLS}" "${ETCD}"
  rsync --mkpath --rsync-path="sudo rsync" ${TLS} ${USER}@${ETCD}:/etc/etcd/pki/
done
```

We have generated and copied all the certificates/keys on each `etcd` node. In the next step, we will create the configuration file and the `systemd` unit file for each node.

# Create `etcd` configuration file
This is the configuration file `/etc/etcd/etcd.conf` that **MUST** be copied on each `etcd` node. The command can be pasted simultaneously on all the nodes. I will be using `tmux` again:
```sh
ETCD_CLUSTER_TOKEN="My_Secret_Token"
ETCD_IP=$(hostname -I | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
ETCD_NAME=$(hostname -s | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
# domain name to build FQDN of all ETCD server(s)
ETCD_CLUSTER_DOMAIN=$(hostname -d | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
ETCD_CA="etcd-ca.crt"

CLIENT_PORT="2379"
SERVER_PORT="2380"
HEALTH_PORT="2381"

# List of all ETCD hostname server(s)
unset ETCD_CLUSTER_NODES
ETCD_CLUSTER_NODES=( \
k8s0etcd1 \
k8s0etcd2 \
k8s0etcd3 \
k8s0etcd4 \
k8s0etcd5 \
k8s0etcd6
)

# Build string of all the ETCD nodes for the flag 'initial-cluster'
ETCD_CLUSTER_NODES_STR=""
for ETCD_NODE in "${ETCD_CLUSTER_NODES[@]}"
do
  # Str will have IP addresses only in URL
  ETCD_CLUSTER_NODES_STR+=",${ETCD_NODE}=https://$(dig +short +search ${ETCD_NODE}):${SERVER_PORT}"
  # String will have FQDN in URL. FQDN proved to be unreliable. ***To be verified***
  # ETCD_CLUSTER_NODES_STR+=",${ETCD_NODE}=https://${ETCD_NODE}.${ETCD_CLUSTER_DOMAIN}:${SERVER_PORT}"
done
# Remove the 1st ","
ETCD_CLUSTER_NODES_STR="${ETCD_CLUSTER_NODES_STR:1}"

cat <<EOF | sudo tee /etc/etcd/etcd.conf > /dev/null
# Configuration options: https://etcd.io/docs/v3.4/op-guide/configuration/
# ETCD config yaml sample: https://github.com/etcd-io/etcd/blob/main/etcd.conf.yml.sample
# The official ETCD ports are:
#   - 2379 for client requests (like the Kubernetes API)
#   - 2380 for peer communication (between ETCD members)
#   - 2381 for health communication status
# The ETCD ports can be set to accept TLS traffic, non-TLS traffic, or both TLS and non-TLS traffic.

# Human-readable name for this member.
# This value MUST be referenced in it's own entries listed in the "--initial-cluster" flag
name: "${ETCD_NAME}"

# Specifies the URLs on which ETCD listens on for client traffic. This flag tells the etcd to accept incoming
# requests from the clients on the specified scheme://IP:port combinations as long as "--listen-client-http-urls" is not specified.
# Domain name is invalid for binding
listen-client-urls: "https://${ETCD_IP}:${CLIENT_PORT},https://127.0.0.1:${CLIENT_PORT}"

# Specifies the URLs on which ETCD listens for communication with the other ETCD nodes that are member of the cluster.
# This flag tells the ETCD to accept incoming requests from its peers on the specified scheme://IP:port combinations.
# Domain name is invalid for binding
listen-peer-urls: "https://${ETCD_IP}:${SERVER_PORT}"

# Specifies the URLs on which ETCD listens for traffic on "/metrics" and "/health" endpoints
listen-metrics-urls: "http://${ETCD_IP}:${HEALTH_PORT},http://127.0.0.1:${HEALTH_PORT}"

# Initial cluster token for the etcd cluster during bootstrap. All members in the same cluster MUST have the same token.
initial-cluster-token: ${ETCD_CLUSTER_TOKEN}

# Comma separated string of initial cluster configuration used during cluster bootstrapping.
initial-cluster: "${ETCD_CLUSTER_NODES_STR}"

# List of this member's client URLs to advertise to the public.
# The URLs needed to be a comma-separated list.
advertise-client-urls: "https://${ETCD_NAME}.${ETCD_CLUSTER_DOMAIN}:${CLIENT_PORT}"

# Specifies the initial peer URLs advertised to the rest of the ETCD members when forming or joining a cluster.
# The URLs needed to be a comma-separated list and can contain domain names.
initial-advertise-peer-urls: "https://${ETCD_NAME}.${ETCD_CLUSTER_DOMAIN}:${SERVER_PORT}"

client-transport-security:
  # Path to the client server TLS cert file.
  cert-file: "/etc/etcd/pki/${ETCD_NAME}.crt"

  # Path to the client server TLS key file.
  key-file: "/etc/etcd/pki/${ETCD_NAME}.key"

  # Path to the client server TLS trusted CA cert file.
  trusted-ca-file: "/etc/etcd/pki/${ETCD_CA}"

  # Enable client cert authentication.
  client-cert-auth: true

peer-transport-security:
  # Path to the peer server TLS cert file.
  cert-file: "/etc/etcd/pki/${ETCD_NAME}.crt"

  # Path to the peer server TLS key file.
  key-file: "/etc/etcd/pki/${ETCD_NAME}.key"

  # Path to the peer server TLS trusted CA cert file.
  trusted-ca-file: "/etc/etcd/pki/${ETCD_CA}"

  # Enable peer client cert authentication.
  client-cert-auth: true

# Path to the data directory.
data-dir: "/var/lib/etcd"

# Initial cluster state ("new" or "existing").
initial-cluster-state: "new"

# Limit etcd to a specific set of tls cipher suites
# cipher-suites: [
#   TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
#   TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
#   TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256,
#   TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256,
#   TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
#   TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
#   TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
#   TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
#   TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
# ]

# Limit etcd to specific TLS protocol versions 
tls-min-version: 'TLS1.2'
tls-max-version: 'TLS1.3'

# Enable debug-level logging for etcd.
# log-level: debug
EOF
```

> [!IMPORTANT]  
> Any locations specified by `listen-metrics-urls` will respond to the `/metrics` and `/health` endpoints in `http` only.
> This can be useful if the standard endpoint is configured with mutual (client) TLS authentication (`listen-client-urls: "https://...`), but a load balancer or monitoring service still needs access to the health check with `http` only.
> The endpoint `listen-client-urls` still answers to `https://.../metrics`.

# Configuring and Starting the `etcd` Cluster
On every node, create the file `/lib/systemd/system/etcd.service` with the following contents. I will be using `tmux` to execute the command once aon all the nodes:
```sh
cat <<EOF | sudo tee /lib/systemd/system/etcd.service
[Unit]
Description=etcd is a strongly consistent, distributed key-value store database
Documentation=https://github.com/etcd-io/etcd
After=network.target
StartLimitIntervalSec=0s
 
[Service]
User=etcd
Type=notify
ExecStart=/usr/local/bin/etcd --config-file /etc/etcd/etcd.conf
Restart=on-failure
RestartSec=10s
LimitNOFILE=40000
 
[Install]
WantedBy=multi-user.target
EOF
```

## System account
Create a Linux system account to run the `etcd` service. This will create a system account with no home directory, no login shell, and no password.
```sh
sudo useradd --system --no-create-home --shell=/sbin/nologin etcd
```

## Setting permissions
```sh
# Common permission settings for etcd config files, certificates and database
sudo chown -R etcd:etcd /etc/etcd
sudo chown -R etcd:etcd /var/lib/etcd/
```

# Start `etcd` service
Start the `etcd` service on all VMs:
```sh
sudo systemctl daemon-reload
sudo systemctl enable --now etcd
sudo systemctl status --no-pager -l etcd
```

# Testing and Validating the Cluster (Optional but strongly suggested if not done earlier)
To interact with the `etcd` cluster, we will be using `etcdctl` and `etcdutl`. This utility as been installed in `/usr/local/bin` on every `etcd` nodes early in this tutorial. Let's also install it on our bastion host. Do NOT install the binary `etcd`.
```sh
ARCH=$(dpkg --print-architecture)
VER=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest|grep tag_name | cut -d '"' -f 4)
echo ${VER}
curl -LO https://github.com/etcd-io/etcd/releases/download/${VER}/etcd-${VER}-linux-${ARCH}.tar.gz
tar xvf etcd-${VER}-linux-${ARCH}.tar.gz
cd etcd-${VER}-linux-${ARCH}
sudo install etcdctl etcdutl -m 755 -o root -g adm /usr/local/bin/
# Cleanup
cd ..
rm -rf etcd-${VER}-linux-${ARCH}*
unset VER
```

You can export these environment variables and connect to the clutser without specifying the values each time:
```sh
ETCD_CLUSTER_PREFIX="k8s0"
ETCD_CLUSTER_DOMAIN=$(hostname -d | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
CA_CERT="etcd-ca"
# Pick any one of the etcd server
ETCD_NAME=${ETCD_CLUSTER_PREFIX}etcd1
export ETCDCTL_CACERT=./${CA_CERT}.crt
export ETCDCTL_CERT=./${ETCD_NAME}.crt
export ETCDCTL_KEY=./${ETCD_NAME}.key
# Build the string of endpoints with IP addresses from the hostnames using 'dig'
ETCD_URL_STR=""
for ETCD_NODE in "${ETCD_CLUSTER_NODES[@]}"
do
  # Use IP address for the endpoints
  ETCD_URL_STR+=",https://$(dig +short +search ${ETCD_NODE}):${CLIENT_PORT}"
  # Use FQDN for the endpoints. FQDN proved to be unreliable at some point. ***To be verified***
  # ETCD_URL_STR+=",https://${ETCD_NODE}.${ETCD_CLUSTER_DOMAIN}:${CLIENT_PORT}"
done
# Remove the 1st ","
ETCD_URL_STR="${ETCD_URL_STR:1}"
export ETCDCTL_ENDPOINTS=${ETCD_URL_STR}
```

> [!NOTE]  
> `export ETCDCTL_API=3` is not needed anymore with `etcd` version >3.4.x  
> You can use any of the three client certificate/key with `ETCDCTL_CERT` and `ETCDCTL_KEY` because they are all signed by the same CA.  
> You can generate another certificate/key pair for your bastion host, as long as the certificate is signed by the `etcd-ca`.  

## Check Cluster status
To execute the next command, you can be on any host that:
- can reach the `etcd` servers on port `TCP/2379`
- has the client certificate/private key and the CA certificate

And now its a lot easier
```sh
etcdctl --write-out=table member list
etcdctl --write-out=table endpoint status
etcdctl --write-out=table endpoint health
```

Output from `etcdctl --write-out=table member list`:
```
+------------------+---------+-----------+----------------------------------+----------------------------------+------------+
|        ID        | STATUS  |   NAME    |            PEER ADDRS            |           CLIENT ADDRS           | IS LEARNER |
+------------------+---------+-----------+----------------------------------+----------------------------------+------------+
|  6c037a33464d5dd | started | k8s0etcd6 | https://k8s0etcd6.kloud.lan:2380 | https://k8s0etcd6.kloud.lan:2379 |      false |
| 1754bcc952d3035f | started | k8s0etcd2 | https://k8s0etcd2.kloud.lan:2380 | https://k8s0etcd2.kloud.lan:2379 |      false |
| 21fea729ce57dfcd | started | k8s0etcd5 | https://k8s0etcd5.kloud.lan:2380 | https://k8s0etcd5.kloud.lan:2379 |      false |
| a4725abb07757413 | started | k8s0etcd3 | https://k8s0etcd3.kloud.lan:2380 | https://k8s0etcd3.kloud.lan:2379 |      false |
| b178165963edc5e0 | started | k8s0etcd4 | https://k8s0etcd4.kloud.lan:2380 | https://k8s0etcd4.kloud.lan:2379 |      false |
| fff1f2470836eb8f | started | k8s0etcd1 | https://k8s0etcd1.kloud.lan:2380 | https://k8s0etcd1.kloud.lan:2379 |      false |
+------------------+---------+-----------+----------------------------------+----------------------------------+------------+
```

Output from `etcdctl --write-out=table endpoint status`:
```
+----------------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|             ENDPOINT             |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://k8s0etcd1.kloud.lan:2379 | fff1f2470836eb8f |  3.5.16 |   20 kB |     false |      false |         2 |         14 |                 14 |        |
| https://k8s0etcd2.kloud.lan:2379 | 1754bcc952d3035f |  3.5.16 |   20 kB |     false |      false |         2 |         14 |                 14 |        |
| https://k8s0etcd3.kloud.lan:2379 | a4725abb07757413 |  3.5.16 |   20 kB |     false |      false |         2 |         14 |                 14 |        |
| https://k8s0etcd4.kloud.lan:2379 | b178165963edc5e0 |  3.5.16 |   20 kB |      true |      false |         2 |         14 |                 14 |        |
| https://k8s0etcd5.kloud.lan:2379 | 21fea729ce57dfcd |  3.5.16 |   20 kB |     false |      false |         2 |         14 |                 14 |        |
| https://k8s0etcd6.kloud.lan:2379 |  6c037a33464d5dd |  3.5.16 |   20 kB |     false |      false |         2 |         14 |                 14 |        |
+----------------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

Output from `etcdctl --write-out=table endpoint health`:
```
+----------------------------------+--------+-------------+-------+
|             ENDPOINT             | HEALTH |    TOOK     | ERROR |
+----------------------------------+--------+-------------+-------+
| https://k8s0etcd4.kloud.lan:2379 |   true | 24.934128ms |       |
| https://k8s0etcd6.kloud.lan:2379 |   true | 25.391913ms |       |
| https://k8s0etcd5.kloud.lan:2379 |   true | 24.959456ms |       |
| https://k8s0etcd3.kloud.lan:2379 |   true | 25.676353ms |       |
| https://k8s0etcd1.kloud.lan:2379 |   true | 33.335156ms |       |
| https://k8s0etcd2.kloud.lan:2379 |   true | 34.799911ms |       |
+----------------------------------+--------+-------------+-------+
```

## Check the logs
> [!NOTE]  
> You need to be on an `etcd` host to execute the following command:

Check for any warnings/errors on every nodes:
```sh
# journalctl -xeu etcd.service
journalctl -f -b -e -x --no-pager -u etcd.service
```

# Test Metrics
Check the `/health` endpoint for all your `etcd` nodes with the command below. You can replace the `/health` with `/metrics` but expect a lenghty output.
```sh
for NODE in "${ETCD_NODES[@]}"; do curl -L http://${NODE}:2381/health; echo ""; done
```

Output:
```
{"health":"true","reason":""}
{"health":"true","reason":""}
{"health":"true","reason":""}
{"health":"true","reason":""}
{"health":"true","reason":""}
{"health":"true","reason":""}
```

# Congratulation
You should have an `etcd` cluster in **High Availibility** mode 🍾🎉🥳

# `etcd` service obliteration (STOP - THINK - WAIT)
One of many recipes for `etcd` service obliteration on Ubuntu 24.04 LTS. This removed all `etcd` traces on a server, including the database, but leaves the server functional, ready to re-install `etcd` again 🤪
```sh
# Remove the service
SERVICE="etcd.service"
sudo systemctl disable --now ${SERVICE}
sudo rm /lib/systemd/system/${SERVICE}
sudo systemctl daemon-reload
sudo systemctl reset-failed
unset SERVICE
# Remove the binaries
sudo rm /usr/local/bin/etcd*
sudo rm -rf /etc/etcd
# Remove the database
sudo rm -rf /var/lib/etcd/
# Remove system account
sudo deluser etcd
```

# References
[etcd main website](https://etcd.io/)  
[Set up a High Availability etcd Cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/)  
[Options for Highly Available Topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)  
[Configuration file for etcd server](https://github.com/etcd-io/etcd/blob/main/etcd.conf.yml.sample)  

---
---

# Etcd Basic Commands (INCOMPLETE)
Here are some basic ETCD commands you can use:

|Description|Command|
|----|----|
|Set a key-value pair|etcdctl put \<key> \<value>|
|Get the value for a key|etcdctl get \<key>|
|Delete a key|etcdctl delete \<key>|
|Watch changes on a key|etcdctl watch \<key>|
|Set a key with a TTL (time-to-live)|etcdctl put \<key> \<value> --ttl seconds|
|List all keys in a directory (with prefix)|etcdctl get \<dir> --prefix|
|Delete all keys in a directory (with prefix)|etcdctl delete \<dir> --prefix|
|Create a new user (for authentication)|etcdctl user add \<username:password>|
|Create a role and grant permissions (RBAC)|etcdctl role add rolename|
||etcdctl role grant-permission rolename|
|Add a new Etcd cluster member|etcdctl member add --peer-urls=https://new-node:2380|
|Remove an Etcd cluster member|etcdctl member remove member-id|
|Take a snapshot of the Etcd data|etcdctl snapshot save \<snapshot.db>|
|Restore Etcd data from a snapshot|etcdctl snapshot restore \<snapshot.db>|
|Display cluster members|etcdctl --write-out=table member list|
|Display cluster health status|etcdctl --write-out=table endpoint status|
|Display cluster health|etcdctl --write-out=table endpoint health|

## Read example
Read examples
```sh
# List all the keys at the root
etcdctl get / --prefix --keys-only
# List the number of keys for a specific directory
etcdctl get "/root1/" --prefix --count-only --write-out=fields
```

## Delete key(s) (WAIT!)
Delete a specific key or all the keys:
```sh
# Delete the keys for a specific prefix (a.k.a., a specific K8s Cluster)
etcdctl del "/k8s1-cluster" --prefix

# Delete ALL the keys
etcdctl del "" --prefix
```

# Quorum
An `etcd` cluster operates so long as a member `quorum` can be established. If `quorum` is lost through transient network failures (e.g., partitions), `etcd` automatically and safely resumes once the network recovers and restores `quorum`. Raft enforces cluster consistency. For power loss, `etcd` persists the Raft log to disk. `etcd` replays the log to the point of failure and resumes cluster participation. For permanent hardware failure, the node may be removed from the cluster through runtime reconfiguration.

`etcd` is designed to withstand machine failures. An `etcd` cluster automatically recovers from temporary failures (e.g., machine reboots) and tolerates up to `(N-1)/2` permanent failures for a cluster of N members. When a member permanently fails, whether due to hardware failure or disk corruption, it loses access to the cluster.

If the cluster permanently loses more than `(N-1)/2` members then it disastrously fails, irrevocably losing quorum. Once quorum is lost, the cluster cannot reach consensus and therefore cannot continue accepting updates.

|Cluster Size|Majority|Failure Tolerance|
|:----:|:----:|:----:|
|1|1|0|
|2|2|0|
|3|2|1|
|4|3|1|
|5|3|2|
|6|4|2|
|7|4|3|
|8|5|3|
|9|5|4|
