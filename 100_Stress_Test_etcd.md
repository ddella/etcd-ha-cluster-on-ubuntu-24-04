# Stress test etcd
```sh
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
```

```sh
CA_CERT="etcd-ca"
ETCD_CLUSTER_PREFIX="k8s0"
# Pick any one of the etcd server
ETCD_NAME=${ETCD_CLUSTER_PREFIX}etcd1
export ETCDCTL_CACERT=./${CA_CERT}.crt
export ETCDCTL_CERT=./${ETCD_NAME}.crt
export ETCDCTL_KEY=./${ETCD_NAME}.key
# Build the string of endpoints with IP addresses from the hostnames using 'dig'
ETCD_URL_STR=""
for ETCD_NODE in "${ETCD_CLUSTER_NODES[@]}"
do
  ETCD_URL_STR+=",https://$(dig +short +search ${ETCD_NODE}):${CLIENT_PORT}"
done
# Remove the 1st ","
ETCD_URL_STR="${ETCD_URL_STR:1}"
export ETCDCTL_ENDPOINTS=${ETCD_URL_STR}
```

Lots of timeout if I use the FQDN for the `etcd` server.
```sh
export ETCDCTL_ENDPOINTS=https://${ETCD_CLUSTER_PREFIX}etcd1.kloud.lan:${CLIENT_PORT},https://${ETCD_CLUSTER_PREFIX}etcd2.kloud.lan:${CLIENT_PORT},https://${ETCD_CLUSTER_PREFIX}etcd3.kloud.lan:${CLIENT_PORT},https://${ETCD_CLUSTER_PREFIX}etcd4.kloud.lan:${CLIENT_PORT},https://${ETCD_CLUSTER_PREFIX}etcd5.kloud.lan:${CLIENT_PORT},https://${ETCD_CLUSTER_PREFIX}etcd6.kloud.lan:${CLIENT_PORT}

export ETCDCTL_ENDPOINTS=${ETCD_CLUSTER_PREFIX}etcd1.kloud.lan:${CLIENT_PORT},${ETCD_CLUSTER_PREFIX}etcd2.kloud.lan:${CLIENT_PORT},${ETCD_CLUSTER_PREFIX}etcd3.kloud.lan:${CLIENT_PORT},${ETCD_CLUSTER_PREFIX}etcd4.kloud.lan:${CLIENT_PORT},${ETCD_CLUSTER_PREFIX}etcd5.kloud.lan:${CLIENT_PORT},${ETCD_CLUSTER_PREFIX}etcd6.kloud.lan:${CLIENT_PORT}

export ETCDCTL_ENDPOINTS=10.103.0.201:2379

```

## Status
Get the status of the cluster every second:
```sh
while true; do etcdctl --write-out=table endpoint health; sleep 1; done
```

## Write
Write data to the Cluster:
```sh
MAX_ROOT=100
MAX_KEY=100
for (( I=1; I<=${MAX_ROOT}; I++ ))
do
  for (( J=1; J<=${MAX_KEY}; J++ ))
  do
    etcdctl put /root${I}/key${J} "root${I} - key${J}"
  done
done
```

## Read
Read the entries from above
```sh
MAX_ROOT=100
MAX_KEY=100
for (( I=1; I<=${MAX_ROOT}; I++ ))
do
  for (( J=1; J<=${MAX_KEY}; J++ ))
  do
    etcdctl get --print-value-only /root${I}/key${J}
  done
done
```

## Delete
Delete all the entries in the database:
```sh
etcdctl del "" --prefix
```

## Check DNS response time
```sh
while true
do
  for i in {1..6}
  do
    T=$(dig +search k8s0etcd${i} | grep time | sed s/\;\;//)
    printf "%s for %s\n" "${T}" "k8s0etcd${i}"
    sleep 2
  done
done
```

