# Create DNS SRV records
```
_etcd-server-ssl._tcp.kloud.lan
_etcd-server._tcp.kloud.lan
```

> [!NOTE]  
> If `_etcd-server-ssl._tcp.kloud.lan` is found then etcd will attempt the bootstrapping process over TLS.

To help clients discover the etcd cluster, the following DNS SRV records are looked up in the listed order:
```
_etcd-client-ssl._tcp.kloud.lan
_etcd-client._tcp.kloud.lan
```

> [!NOTE]  
> If `_etcd-client-ssl._tcp.kloud.lan` is found, clients will attempt to communicate with the etcd cluster over SSL/TLS.

## Server entries
```
_etcd-server-ssl._tcp.kloud.lan. 300 IN SRV 0 0 2380 k8s0etcd1.kloud.lan.
_etcd-server-ssl._tcp.kloud.lan. 300 IN SRV 0 0 2380 k8s0etcd2.kloud.lan.
_etcd-server-ssl._tcp.kloud.lan. 300 IN SRV 0 0 2380 k8s0etcd3.kloud.lan.
_etcd-server-ssl._tcp.kloud.lan. 300 IN SRV 0 0 2380 k8s0etcd4.kloud.lan.
_etcd-server-ssl._tcp.kloud.lan. 300 IN SRV 0 0 2380 k8s0etcd5.kloud.lan.
_etcd-server-ssl._tcp.kloud.lan. 300 IN SRV 0 0 2380 k8s0etcd6.kloud.lan.
```

Test the entries:
```sh
dig +noall +answer SRV _etcd-server-ssl._tcp.kloud.lan
```

## Client entries
```
_etcd-client-ssl._tcp.kloud.lan. 300 IN SRV 0 0 2379 k8s0etcd1.kloud.lan.
_etcd-client-ssl._tcp.kloud.lan. 300 IN SRV 0 0 2379 k8s0etcd2.kloud.lan.
_etcd-client-ssl._tcp.kloud.lan. 300 IN SRV 0 0 2379 k8s0etcd3.kloud.lan.
_etcd-client-ssl._tcp.kloud.lan. 300 IN SRV 0 0 2379 k8s0etcd4.kloud.lan.
_etcd-client-ssl._tcp.kloud.lan. 300 IN SRV 0 0 2379 k8s0etcd5.kloud.lan.
_etcd-client-ssl._tcp.kloud.lan. 300 IN SRV 0 0 2379 k8s0etcd6.kloud.lan.
```

Test the entries:
```sh
dig +noall +answer SRV _etcd-client-ssl._tcp.kloud.lan
```

## A records
```sh
dig +noall +answer k8s0etcd1.kloud.lan k8s0etcd2.kloud.lan k8s0etcd3.kloud.lan k8s0etcd4.kloud.lan k8s0etcd5.kloud.lan k8s0etcd6.kloud.lan
```

```
k8s0etcd1.kloud.lan.	3600	IN	A	10.103.0.101
k8s0etcd2.kloud.lan.	3600	IN	A	10.103.0.102
k8s0etcd3.kloud.lan.	3600	IN	A	10.103.0.103
k8s0etcd4.kloud.lan.	3600	IN	A	10.103.0.201
k8s0etcd5.kloud.lan.	3600	IN	A	10.103.0.202
k8s0etcd6.kloud.lan.	3600	IN	A	10.103.0.203
```

# Parse output from `dig`

```sh
unset SERVER
readarray -t SERVER < <( dig +noall +answer +short SRV _etcd-server-ssl._tcp.kloud.lan | awk  '/ / {print $3','$4}' | sed 's/.$//' )
```

```sh
unset CLIENT
readarray -t CLIENT < <( dig +noall +answer +short SRV _etcd-client-ssl._tcp.kloud.lan | awk  '/ / {print $3','$4}' | sed 's/.$//' )
```

```
daniel@docker1 ~ $ for i in "${SERVER[@]}"; do echo "$i"; done
2380 k8s0etcd4.kloud.lan.
2380 k8s0etcd6.kloud.lan.
2380 k8s0etcd1.kloud.lan.
2380 k8s0etcd3.kloud.lan.
2380 k8s0etcd2.kloud.lan.
2380 k8s0etcd5.kloud.lan.
```

```
daniel@docker1 ~ $ printf -- "%s\n" "${SERVER[@]}"
2380 k8s0etcd3.kloud.lan.
2380 k8s0etcd1.kloud.lan.
2380 k8s0etcd5.kloud.lan.
2380 k8s0etcd6.kloud.lan.
2380 k8s0etcd2.kloud.lan.
2380 k8s0etcd4.kloud.lan.
```

```
daniel@docker1 ~ $ echo ${SERVER[1]}
2380 k8s0etcd6.kloud.lan.
```

https://www.cloudflare.com/en-ca/learning/dns/dns-records/dns-srv-record/
