# Snapshotting the keyspace
Recovering a cluster first needs a snapshot of the keyspace from an `etcd` member. A snapshot may either be taken from a live member with the `etcdctl snapshot save` command or by copying the `/var/lib/etcd/member/snap/db` file from an `etcd` data directory. For example, the following command snapshots the keyspace served by `$ETCDCTL_ENDPOINTS` to the file `./snapshot-${SUFFIX}.db`:

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
# Take the snapshot
etcdctl snapshot save ./snapshot-${SUFFIX}.db
```

> [!IMPORTANT]  
> Snapshot can only be requested from one `etcd` node, so `ETCDCTL_ENDPOINTS` flag should contain only one endpoint.

Output should look like:
```
{"level":"info","ts":"2024-10-27T10:45:31.082046-0400","caller":"snapshot/v3_snapshot.go:65","msg":"created temporary db file","path":"./snapshot-20241027-104531.db.part"}
{"level":"info","ts":"2024-10-27T10:45:31.098377-0400","logger":"client","caller":"v3@v3.5.16/maintenance.go:212","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":"2024-10-27T10:45:31.098583-0400","caller":"snapshot/v3_snapshot.go:73","msg":"fetching snapshot","endpoint":"https://10.103.0.203:2379"}
{"level":"info","ts":"2024-10-27T10:45:31.129590-0400","logger":"client","caller":"v3@v3.5.16/maintenance.go:220","msg":"completed snapshot read; closing"}
{"level":"info","ts":"2024-10-27T10:45:31.139483-0400","caller":"snapshot/v3_snapshot.go:88","msg":"fetched snapshot","endpoint":"https://10.103.0.203:2379","size":"2.5 MB","took":"now"}
{"level":"info","ts":"2024-10-27T10:45:31.139594-0400","caller":"snapshot/v3_snapshot.go:97","msg":"saved","path":"./snapshot-20241027-104531.db"}
```

> [!WARNING]  
> Note that taking the snapshot from the `/var/lib/etcd/member/snap/db` file might lose data that has not been written yet, but is included in the wal (write-ahead-log) folder.

## Status of a snapshot
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