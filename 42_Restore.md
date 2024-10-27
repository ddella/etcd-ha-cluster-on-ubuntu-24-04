# Restore from Snapshot
Recovering a cluster first needs a snapshot of the keyspace from an `etcd` member.

> [!NOTE]  
> You need to have access to client certificate/private key and CA certificate.

```sh
ETCD_NAME="k8s0etcd1" # Pick any one of the etcd server
CLIENT_PORT="2379"  # Client TCP port
CA_CERT="etcd-ca"
SUFFIX=$(date +"%Y%m%d-%H%M%S")
export ETCDCTL_CACERT=./${CA_CERT}.crt  # ETCD CA certificate used to signed the client certificates
export ETCDCTL_CERT=./${ETCD_NAME}.crt  # Client cert
export ETCDCTL_KEY=./${ETCD_NAME}.key   # Client private key
export ETCDCTL_ENDPOINTS="https://$(dig +short +search ${ETCD_NODE}):${CLIENT_PORT}"
```

## Snapshot status
To understand which revision and hash a given snapshot contains, you can use the `etcdutl snapshot status` command:

> [!NOTE]  
> You should be in the same terminal as the step above to have all the variables

```sh
etcdutl snapshot status ./snapshot-${SUFFIX}.db -w table
```

Output:
```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 26eb70b2 |    30090 |      30115 |     2.5 MB |
+----------+----------+------------+------------+
```

# Restore

## Copy Snapshot
Copy the snapshot to **every** `etcd` nodes. If the location of the snapshot is available from the `etcd` nodes, you can skip this section.
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

for ETCD in "${ETCD_NODES[@]}"
do
  SNAPSHOT="snapshot-20241027-104531.db"
  printf "Copying file [%s] on [%s]\n" "${SNAPSHOT}" "${ETCD}"
  rsync --mkpath --rsync-path="rsync" ${SNAPSHOT} ${USER}@${ETCD}:/home/${USER}
done
```

## Restore
On **each** `etcd` node, restore the snapshot to a different directory from the production database. I'm using `tmux` to execute the commands on all `etcd` nodes simultaneously.
```sh
NEW_DIR="/var/lib/etcd1"
sudo etcdutl snapshot restore ./snapshot-20241027-104531.db --data-dir ${NEW_DIR} --bump-revision 100 --mark-compacted
sudo chown -R etcd:etcd ${NEW_DIR}
```

## etcd.conf
We need to modify the configuration file, on **every** `etcd` node, for them to use the restored database . I'm using `tmux` to execute the commands on all `etcd` nodes simultaneously.

The steps to be executed on **every** `etcd` node are:
- stop the `etcd` service
- change the configuration of the data directory in the configuration file for the one where you did the `etcdutl snapshot restore`
- start the `etcd` service
- check the status of the `etcd` service

```sh
# Stop the service
sudo systemctl stop etcd.service
# Path to the data directory in the configuration file '/etc/etcd/etcd.conf'
sudo sed -i s#/var/lib/etcd#/var/lib/etcd1# /etc/etcd/etcd.conf
# Start the service
sudo systemctl start etcd.service
# Check the status
sudo systemctl status --no-pager -l etcd.service
```
