# Create `etcd` Network
This section will show how to create a Linux Bridge to host the VMs for the `etcd` Cluster. If you already have your network, just skip this tutorial.

Example of my naming convention and IP numbering:
```
10.103.0.0/24
    │  │
    │  │
    │  └----> Kubernetes Cluster #
    │
    └-----> Type of network ("100" for bastion, "101" for master, "102" for worker, "103" for etcd, ...)
```

> [!NOTE]  
> I have two KVM hosts, named `pve1.kloud.lan` and `pve2.kloud.lan`. The VMs can be spread between those KVM nodes since the bridge will be link via VxLAN.

# Create `etcd` bridge on `pve1.kloud.lan`
> [!IMPORTANT]  
> This section applies to node `pve1.kloud.lan`. You should have an SSH session on it.

## Define Variables
Define all the variables needed for the scripts below. Make sure you execute the scripts in the same window as the one you defined the variables.
```sh
# Cluster prefix for naming convention
K8S_CLUSTER_INDEX="10"    # Index for VxLAN and file naming
K8S_CLUSTER_NAME="k8s1"   # K8s Cluster name (k8s1, k8s2, ...)
K8S_BRIDGE_TYPE="etcd"    # Type of servers in that bridge (bastion, master, worker, etcd, ...)

# The name of the bridge
BRIDGE="${K8S_CLUSTER_NAME}-${K8S_BRIDGE_TYPE}"
BRIDGE_IPv4="10.103.0.1/24"

# VxLAN variables
VXLAN_ID="${K8S_CLUSTER_INDEX}103"
VXLAN_INTF="${K8S_CLUSTER_NAME}-${VXLAN_ID}"
VTEP1_ADDR="192.168.11.1"
VTEP2_ADDR="192.168.11.2"
VTEP_DEV="eth1"
```

## Creating a new virtual network
Create a Linux bridge on each node with an IP address. The default gateway for all the VMs connected to it will be this IP address. After the creation of the bridge on each node, we need to join them by VxLAN.

If you use the command line to create the bridge and the VxLAN they won't persist a reboot. We'll use `netplan` so that all the configurations will be persistent.

### Bridge Creation
Add the following files in your `/etc/netplan/` directory and use the command `sudo netplan apply` to apply the configurations. This will persist when the node reboots:

Linux Bridge configuration:
```sh
CONFIG_FILE="${K8S_CLUSTER_INDEX}-${K8S_CLUSTER_NAME}-${K8S_BRIDGE_TYPE}.yaml"
cat <<EOF | sudo tee /etc/netplan/${CONFIG_FILE} > /dev/null
# Linux Bridge for Kubernetes Cluster etcd hosts
# ${CONFIG_FILE}
network:
  version: 2
  renderer: networkd
  bridges:
    ${BRIDGE}:
      addresses: [ ${BRIDGE_IPv4} ]
      dhcp4: false
      dhcp6: false
      mtu: 9000
EOF
sudo chmod 600 /etc/netplan/${CONFIG_FILE}
```

> [!IMPORTANT]  
> Adjust the address of the bridge. It will be my default gateway for the external world.

### VxLAN Creation
Linux VxLAN between the bridges on each node:
```sh
CONFIG_FILE="${K8S_CLUSTER_INDEX}-${K8S_CLUSTER_NAME}-${K8S_BRIDGE_TYPE}-vxlan-${VXLAN_ID}.yaml"
cat <<EOF | sudo tee /etc/netplan/${CONFIG_FILE} > /dev/null
# ${CONFIG_FILE}
# https://github.com/canonical/netplan/blob/main/examples/vxlan.yaml
# https://netplan.readthedocs.io/en/latest/netplan-yaml/#properties-for-device-type-tunnels
network:
  renderer: networkd
  tunnels:
    ${VXLAN_INTF}:
      mode: vxlan
      id: ${VXLAN_ID}
      link: ${VTEP_DEV}
      mtu: 8950
      port: 4789
      local: ${VTEP1_ADDR}
      remote: ${VTEP2_ADDR}
  bridges:
    ${BRIDGE}:
      interfaces:
        - ${VXLAN_INTF}
      mtu: 8950
EOF
sudo chmod 600 /etc/netplan/${CONFIG_FILE}
```

## Apply the configuration
Apply the new configuration to create the bridge and VxLAN. If it's good, hit enter. If somethinbg went wrong it will revert in 120 seconds:
```sh
sudo netplan try
```

----------
----------

# Create `etcd` bridge on `pve2.kloud.lan`
> [!IMPORTANT]  
> This section applies to node `pve2.kloud.lan`. You should have an SSH session on it.

## Define Variables
Define all the variables needed for the scripts below. Make sure you execute the scripts in the same window as the one you defined the variables.
```sh
# Cluster prefix for naming convention
K8S_CLUSTER_INDEX="10"    # Index for VxLAN and file naming
K8S_CLUSTER_NAME="k8s1"   # K8s Cluster name (k8s1, k8s2, ...)
K8S_BRIDGE_TYPE="etcd"    # Type of servers in that bridge (bastion, master, worker, etcd, ...)

# The name of the bridge
BRIDGE="${K8S_CLUSTER_NAME}-${K8S_BRIDGE_TYPE}"
BRIDGE_IPv4="10.103.0.2/24"

# VxLAN variables
VXLAN_ID="${K8S_CLUSTER_INDEX}103"
VXLAN_INTF="${K8S_CLUSTER_NAME}-${VXLAN_ID}"
VTEP1_ADDR="192.168.11.2"
VTEP2_ADDR="192.168.11.1"
VTEP_DEV="eth1"
```

## Creating a new virtual network
Create a Linux bridge on each node with an IP address. The default gateway for all the VMs connected to it will be this IP address. After the creation of the bridge on each node, we need to join them by VxLAN.

If you use the command line to create the bridge and the VxLAN they won't persist a reboot. We'll use `netplan` so that all the configurations will be persistent.

### Bridge Creation
Add the following files in your `/etc/netplan/` directory and use the command `sudo netplan apply` to apply the configurations. This will persist when the node reboots:

Linux Bridge configuration:
```sh
CONFIG_FILE="${K8S_CLUSTER_INDEX}-${K8S_CLUSTER_NAME}-${K8S_BRIDGE_TYPE}.yaml"
cat <<EOF | sudo tee /etc/netplan/${CONFIG_FILE} > /dev/null
# Linux Bridge for Kubernetes Cluster etcd hosts
# ${CONFIG_FILE}
network:
  version: 2
  renderer: networkd
  bridges:
    ${BRIDGE}:
      addresses: [ ${BRIDGE_IPv4} ]
      dhcp4: false
      dhcp6: false
      mtu: 9000
EOF
sudo chmod 600 /etc/netplan/${CONFIG_FILE}
```

> [!IMPORTANT]  
> Adjust the address of the bridge. It will be my default gateway for the external world.

### VxLAN Creation
Linux VxLAN between the bridges on each node:
```sh
CONFIG_FILE="${K8S_CLUSTER_INDEX}-${K8S_CLUSTER_NAME}-${K8S_BRIDGE_TYPE}-vxlan-${VXLAN_ID}.yaml"
cat <<EOF | sudo tee /etc/netplan/${CONFIG_FILE} > /dev/null
# ${CONFIG_FILE}
# https://github.com/canonical/netplan/blob/main/examples/vxlan.yaml
# https://netplan.readthedocs.io/en/latest/netplan-yaml/#properties-for-device-type-tunnels
network:
  renderer: networkd
  tunnels:
    ${VXLAN_INTF}:
      mode: vxlan
      id: ${VXLAN_ID}
      link: ${VTEP_DEV}
      mtu: 8950
      port: 4789
      local: ${VTEP1_ADDR}
      remote: ${VTEP2_ADDR}
  bridges:
    ${BRIDGE}:
      interfaces:
        - ${VXLAN_INTF}
      mtu: 8950
EOF
sudo chmod 600 /etc/netplan/${CONFIG_FILE}
```

## Apply the configuration
Apply the new configuration to create the bridge and VxLAN. If it's good, hit enter. If somethinbg went wrong it will revert in 120 seconds:
```sh
sudo netplan try
```

## Verification (Optional)
```sh
ip add show ${BRIDGE}
ip add show ${VXLAN_INTF}
ip link show master ${BRIDGE}
ip -j -p -d link show ${BRIDGE}
ip -j -p -d link show ${VXLAN_INTF}
ip -br a
```

# References
[ip-link - network device configuration](https://man7.org/linux/man-pages/man8/ip-link.8.html)  
[netplan yaml configuration](https://netplan.readthedocs.io/en/latest/netplan-yaml/)  

# Test (Optional)
If you want to test connectivity between the new bridge accros the KVM hosts, you can create virtual interface on each bridge and ping the remote one.

## This section applies to KVM host `pve1.kloud.lan`
```sh
NS="ns"
VETH="veth"
VPEER="vpeer" # interface inside the network ns
VPEER_ADDR_LOCAL="10.10.0.11"
VPEER_ADDR_REMOTE="10.10.0.21"

# create namespace
sudo ip netns add $NS
# create veth link
sudo ip link add ${VETH} type veth peer name ${VPEER}
# bring up devices
sudo ip link set ${VETH} up
sudo ip netns exec ${NS} ip link set lo up
# add peers to ns
sudo ip link set ${VPEER} netns ${NS}
# setup peer ns interface
sudo ip netns exec ${NS} ip link set ${VPEER} up
# assign ip address to ns interface
sudo ip netns exec ${NS} ip addr add ${VPEER_ADDR_LOCAL}/24 dev ${VPEER}
# assign veth pairs to bridge
sudo ip link set ${VETH} master ${BRIDGE}
```

> [!NOTE]  
> It may take 20 seconds to ping the remote.
```sh
# ping local and remote IP in the ns
sudo ip netns exec ${NS} ping -c 4 ${VPEER_ADDR_LOCAL}
sudo ip netns exec ${NS} ping ${VPEER_ADDR_REMOTE}
sudo ip netns exec ${NS} ip neighbor
```

Clean up:
```sh
# delete namespace
sudo ip netns delete ${NS}
```

---
---

## This section applies to KVM host `pve2.kloud.lan`
```sh
NS="ns"
VETH="veth"
VPEER="vpeer" # interface inside the network ns
VPEER_ADDR_LOCAL="10.10.0.21"
VPEER_ADDR_REMOTE="10.10.0.11"

# create namespace
sudo ip netns add $NS
# create veth link
sudo ip link add ${VETH} type veth peer name ${VPEER}
# bring up devices
sudo ip link set ${VETH} up
sudo ip netns exec ${NS} ip link set lo up
# add peers to ns
sudo ip link set ${VPEER} netns ${NS}
# setup peer ns interface
sudo ip netns exec ${NS} ip link set ${VPEER} up
# assign ip address to ns interface
sudo ip netns exec ${NS} ip addr add ${VPEER_ADDR_LOCAL}/24 dev ${VPEER}
# assign veth pairs to bridge
sudo ip link set ${VETH} master ${BRIDGE}
```

> [!NOTE]  
> It may take 20 seconds to ping the remote.

```sh
# ping local and remote IP in the ns
sudo ip netns exec ${NS} ping -c 4 ${VPEER_ADDR_LOCAL}
sudo ip netns exec ${NS} ping ${VPEER_ADDR_REMOTE}
sudo ip netns exec ${NS} ip neighbor
```

Clean up:
```sh
# delete namespace
sudo ip netns delete ${NS}
```
