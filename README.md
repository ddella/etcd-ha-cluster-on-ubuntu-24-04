# Set up a High Availability etcd Cluster
`etcd` is distributed, reliable key-value store for the most critical data of a distributed system. It's a strongly consistent, distributed key-value store that provides a reliable way to store data that needs to be accessed by a distributed system or cluster of machines. It gracefully handles leader elections during network partitions and can tolerate machine failure, even in the leader node.

This guide will cover the instructions for bootstrapping an `etcd` cluster on a six-node cluster from pre-built binaries. This tutorial has nothing to do with Kubernetes but it could eventually be used when installing an H.A Kubernetes Cluster with `External etcd` topology.

# Before you begin
We will be using six (6) Ubuntu Server 24.04 LTS with the most recent Linux Kernel. The most optimum number of servers is three or five for performance, reliability and quorum. I used six for symetry 😉

Prerequisites:
- Six Ubuntu 24.04 LTS with minimal install and SSH server active
- All the nodes must be able communicate with each other on TCP port `2380`.
- The clients of the `etcd` cluster (ex. Kubernetes API) can reach any of them on TCP port `2379`.
  - TCP port `2379` is the traffic for client requests
  - TCP port `2380` is the traffic for server-to-server communication
  - TCP port `2381` is the traffic for the endpoints `/metrics` and `/health` (Optional)
- Each host must have `systemd` and a `bash` compatible shell installed.
- Some infrastructure to copy files between hosts. For example, `scp` can satisfy this requirement.
- Passwordless SSH access to the servers

## TODO
Things to do and test before going to full production.

- Add a new member
- Remove an existing member
- Take a snapshot
- Restore a snapshot
- Update the TLS certificates before expiration

# Servers for this tutorial
Following is the table that describes every IP addresses of each servers needed in this tutorial.

> [!IMPORTANT]  
> You **must** have DNS entries, `A` and `PTR` records, for all the `etcd` servers. The scripts relies heavily on DNS being up to date.

## ETCD Cluster Nodes
Adjusts for your needs.

|Role|FQDN|IP|OS|Kernel|RAM|vCPU|KVM Host|
|----|----|----|----|----|----|----|----|
|etcd Database|k8s0etcd1.kloud.lan|10.103.0.101|Ubuntu 24.04.1|6.11.0|4G|2|pve1|
|etcd Database|k8s0etcd2.kloud.lan|10.103.0.102|Ubuntu 24.04.1|6.11.0|4G|2|pve1|
|etcd Database|k8s0etcd3.kloud.lan|10.103.0.103|Ubuntu 24.04.1|6.11.0|4G|2|pve1|
|etcd Database|k8s0etcd4.kloud.lan|10.103.0.201|Ubuntu 24.04.1|6.11.0|4G|2|pve2|
|etcd Database|k8s0etcd5.kloud.lan|10.103.0.202|Ubuntu 24.04.1|6.11.0|4G|2|pve2|
|etcd Database|k8s0etcd6.kloud.lan|10.103.0.203|Ubuntu 24.04.1|6.11.0|4G|2|pve2|

## ETCD version
The latest `etcd` version will be used:
```sh
ETCD_ARCH=$(dpkg --print-architecture)
ETCD_VER=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest|grep tag_name | cut -d '"' -f 4)
printf "ETCD latest version is [%s] on architecture [%s]\n" "${ETCD_VER}" "${ETCD_ARCH}"
```

## Support tiers
`etcd` on Linux with `AMD64` architecture is guaranted support tier 1.

- Tier 1: fully supported by `etcd` maintainers; `etcd` is guaranteed to pass all tests including functional and robustness tests.
- Tier 2: `etcd` is guaranteed to pass integration and end-to-end tests but not necessarily functional or robustness tests.
- Tier 3: `etcd` is guaranteed to build, may be lightly tested (or not), and so it should be considered *unstable*.

# Create an `etcd` Cluster
The high level steps to create an High Availability `etcd` Cluster.

> [!NOTE]  
> If you already have your VMs on your network, you can jump to step 3, Bootstrap the `etcd` cluster.

1. [Create a Linux Bridge to host the `etcd` servers (Optional)](10_Create_etcd_Network.md)
2. [Create the VMs (Optional if already have your VMs ready)](20_Create_etcd_VMs.md)
3. [Bootstrap the `etcd` cluster](30_Bootstrap_etcd_Cluster.md)

# References
[ETCD on GitHub](https://github.com/etcd-io/etcd)  
[Supported platforms](https://etcd.io/docs/v3.5/op-guide/supported-platform/)  
