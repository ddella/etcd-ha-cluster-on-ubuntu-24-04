# Start Linux `etcd` VMs
This tutorial will show how to create Linux Ubuntu 24.04 LTS VMs. The VMs will serve as the external `etcd` database that could be used, for example, for a Kubernetes Cluster.

|Role|FQDN|IP|OS|Kernel|RAM|vCPU|KVM Host|
|----|----|----|----|----|----|----|----|
|etcd Database|k8s0etcd1.kloud.lan|10.103.0.101|Ubuntu 24.04.1|6.11.0|4G|2|pve1|
|etcd Database|k8s0etcd2.kloud.lan|10.103.0.102|Ubuntu 24.04.1|6.11.0|4G|2|pve1|
|etcd Database|k8s0etcd3.kloud.lan|10.103.0.103|Ubuntu 24.04.1|6.11.0|4G|2|pve1|
|etcd Database|k8s0etcd4.kloud.lan|10.103.0.201|Ubuntu 24.04.1|6.11.0|4G|2|pve2|
|etcd Database|k8s0etcd5.kloud.lan|10.103.0.202|Ubuntu 24.04.1|6.11.0|4G|2|pve2|
|etcd Database|k8s0etcd6.kloud.lan|10.103.0.203|Ubuntu 24.04.1|6.11.0|4G|2|pve2|

# This section applies to KVM host `pve1.kloud.lan`
> [!IMPORTANT]  
> This section applies to KVM host `pve1.kloud.lan` **ONLY**.

## Define Variables
Define all the variables needed for the scripts below. Make sure you execute the scripts in the same window as the one you defined the variables.
```sh
# VM template. This VM must be shutdown
ORIGINAL="k8s1-template"
SUBNET="10.103.0"
# Declare a "kindOf" dictionnary
unset newVM
declare -A newVM
newVM[k8s0etcd1]="${SUBNET}.101"
newVM[k8s0etcd2]="${SUBNET}.102"
newVM[k8s0etcd3]="${SUBNET}.103"

# Subnet info for the VMs
localSubnet="${SUBNET}.0"
subnetMask="24"
defaultGateway="${SUBNET}.1"
localBroadcast="${SUBNET}.255"

# The bridge for the new VMs
BRIDGE="k8s0-etcd"
# The bridge where the template VM is connected
OLD_BRIDGE="k8s1-bastion"
TEMPLATE_IPv4="10.100.1.81"
TEMPLATE_GATEWAY="10.100.1.1"

# Local username on the clone VM
USERNAME="daniel"

# Libvirt images directory
LIBVIRT_DIR="/var/lib/libvirt/images"
```

## Create the VMs
### Clone VM
Start by cloning a valid VM. Make sure the template VM is shutdown:
```sh
# Make sure the template VM is shutdown
for NEW in "${!newVM[@]}"
do
  # Create the new directory
  sudo install -v -d -g libvirt -o libvirt-qemu ${LIBVIRT_DIR}/${NEW}

  # Clone the VM
  sudo virt-clone --connect qemu:///system \
  --original ${ORIGINAL} --name ${NEW} \
  --file ${LIBVIRT_DIR}/${NEW}/${NEW}.qcow2

  # Change permission
  sudo chown libvirt-qemu:libvirt ${LIBVIRT_DIR}/${NEW}/${NEW}.qcow2
done
```

### Customize the VMs
Once you have cloned the VM, you can customize the new virtual machines with `virt-sysprep` utility. Make sure the utility is installed with the command `sudo apt install guestfs-tools`
```sh
OPERATIONS=$(virt-sysprep --list-operations | egrep -v 'lvm-uuids|fs-uuids|ssh-hostkeys|ssh-userdir' | awk '{ printf "%s,", $1}' | sed 's/,$//')

for NEW in "${!newVM[@]}"
do
  sudo virt-sysprep -d ${NEW} \
  --hostname ${NEW}.kloud.lan \
  --enable ${OPERATIONS} \
  --keep-user-accounts ${USERNAME} \
  --run-command "ssh-keygen -q -t ed25519 -N '' -f /home/${USERNAME}/.ssh/id_ed25519" \
  --ssh-inject ${USERNAME}:file:/home/${USERNAME}/.ssh/id_ecdsa.pub \
  --run-command "sed -i \"s/127.0.1.1.*/127.0.1.1 ${NEW}/\" /etc/hosts" \
  --run-command "sed -i \"s/127.0.0.1 localhost/127.0.0.1 localhost ${NEW}/\" /etc/hosts" \
  --run-command "sed -i \"s/${TEMPLATE_IPv4}/${newVM[${NEW}]}/\" /etc/netplan/50-cloud-init.yaml" \
  --run-command "sed -i \"s/${TEMPLATE_GATEWAY}/${defaultGateway}/\" /etc/netplan/50-cloud-init.yaml"
done
```

### Attach VM to bridge
> [!NOTE]  
> You can skip this section if your VM template has the correct bridge network configured.

Make it permanent:
```sh
for NEW in "${!newVM[@]}"
do
  virsh dumpxml ${NEW} > ${NEW}.xml
  sed -i "s/${OLD_BRIDGE}/${BRIDGE}/g" ${NEW}.xml
  virsh define --file ${NEW}.xml
  virsh autostart ${NEW}
  rm -f ${NEW}.xml
done
```

## Start the VMs
Start the VMs.

> [!NOTE]  
> By default the VMs won't auto start. I added the line `virsh autostart ${NEW}` for them to start automaticaly.

```sh
for NEW in "${!newVM[@]}"
do
  printf "Starting VM %s\n" "${NEW}"
  virsh start ${NEW}
done
```

## Add SNAT (Optional)
You can add a source NAT rule for the VMs to access the Internet via the node's default gateway. This command configure the NAT.

IF THE FILE `/etc/iptables/rules.v4` EXISTS use this command:
```sh
sudo sed -i -e '/# Forward traffic.*/a\' -e "-A POSTROUTING -s ${localSubnet}/${subnetMask} -o eth0 -j MASQUERADE" /etc/iptables/rules.v4
```

If the file `/etc/iptables/rules.v4` doesn't exist, use the following:
```sh
cat <<EOF | sudo tee -a /etc/iptables/rules.v4
# nat Table rules
*nat
:POSTROUTING ACCEPT [0:0]

# Forward traffic from VM subnets through eth0.
-A POSTROUTING -s ${localSubnet}/${subnetMask} -o eth0 -j MASQUERADE

# don't delete the 'COMMIT' line or these nat table rules won't be processed
COMMIT
EOF
```

You can apply the rules with the command:
```sh
sudo iptables-restore /etc/iptables/rules.v4
```
----------
----------
----------
# Node PVE2
> [!IMPORTANT]  
> This section applies to KVM host `pve1` **ONLY**

## Define Variables
Define all the variables needed for the scripts below. Make sure you execute the scripts in the same window as the one you defined the variables.
```sh
# VM template. This VM must be shutdown
ORIGINAL="k8s1-template"
SUBNET="10.103.0"
# Declare a "kindOf" dictionnary
unset newVM
declare -A newVM
newVM[k8s0etcd4]="${SUBNET}.201"
newVM[k8s0etcd5]="${SUBNET}.202"
newVM[k8s0etcd6]="${SUBNET}.203"

# Subnet info for the VMs
localSubnet="${SUBNET}.0"
subnetMask="24"
defaultGateway="${SUBNET}.2"
localBroadcast="${SUBNET}.255"

# The bridge for the new VMs
BRIDGE="k8s0-etcd"
# The bridge where the template VM is connected
OLD_BRIDGE="k8s1-bastion"
TEMPLATE_IPv4="10.100.1.82"
TEMPLATE_GATEWAY="10.100.1.2"

# Local username on the clone VM
USERNAME="daniel"

LIBVIRT_DIR="/var/lib/libvirt/images"
```

## Create the VMs
### Clone VM
Start by cloning a valid VM. Make sure the template VM is shutdown:
```sh
# Make sure the template VM is shutdown
for NEW in "${!newVM[@]}"
do
  # Create the new directory
  sudo install -v -d -g libvirt -o libvirt-qemu ${LIBVIRT_DIR}/${NEW}

  # Clone the VM
  sudo virt-clone --connect qemu:///system \
  --original ${ORIGINAL} --name ${NEW} \
  --file ${LIBVIRT_DIR}/${NEW}/${NEW}.qcow2

  # Change permission
  sudo chown libvirt-qemu:libvirt ${LIBVIRT_DIR}/${NEW}/${NEW}.qcow2
done
```

### Customize the VMs
Once you have cloned the VM, you can customize the new virtual machines with `virt-sysprep` utility. Make sure the utility is installed with the command `sudo apt install guestfs-tools`
```sh
OPERATIONS=$(virt-sysprep --list-operations | egrep -v 'lvm-uuids|fs-uuids|ssh-hostkeys|ssh-userdir' | awk '{ printf "%s,", $1}' | sed 's/,$//')

for NEW in "${!newVM[@]}"
do
  sudo virt-sysprep -d ${NEW} \
  --hostname ${NEW}.kloud.lan \
  --enable ${OPERATIONS} \
  --keep-user-accounts ${USERNAME} \
  --run-command "ssh-keygen -q -t ed25519 -N '' -f /home/${USERNAME}/.ssh/id_ed25519" \
  --ssh-inject ${USERNAME}:file:/home/${USERNAME}/.ssh/id_ecdsa.pub \
  --run-command "sed -i \"s/127.0.1.1.*/127.0.1.1 ${NEW}/\" /etc/hosts" \
  --run-command "sed -i \"s/127.0.0.1 localhost/127.0.0.1 localhost ${NEW}/\" /etc/hosts" \
  --run-command "sed -i \"s/${TEMPLATE_IPv4}/${newVM[${NEW}]}/\" /etc/netplan/50-cloud-init.yaml" \
  --run-command "sed -i \"s/${TEMPLATE_GATEWAY}/${defaultGateway}/\" /etc/netplan/50-cloud-init.yaml"
done
```

### Attach VM to bridge
> [!NOTE]  
> You can skip this section if your VM template has the correct bridge network configured.

Make it permanent:
```sh
for NEW in "${!newVM[@]}"
do
  virsh dumpxml ${NEW} > ${NEW}.xml
  sed -i "s/${OLD_BRIDGE}/${BRIDGE}/g" ${NEW}.xml
  virsh define --file ${NEW}.xml
  virsh autostart ${NEW}
  rm -f ${NEW}.xml
done
```

## Start the VMs
Start the VMs.

> [!NOTE]  
> By default the VMs won't auto start. I added the line `virsh autostart ${NEW}` for them to start automaticaly.

```sh
for NEW in "${!newVM[@]}"
do
  printf "Starting VM %s\n" "${NEW}"
  virsh start ${NEW}
done
```

## Add SNAT (Optional)
You can add a source NAT rule for the VMs to access the Internet via the node's default gateway. This command configure the NAT.

IF THE FILE `/etc/iptables/rules.v4` EXISTS use this command:
```sh
sudo sed -i -e '/# Forward traffic.*/a\' -e "-A POSTROUTING -s ${localSubnet}/${subnetMask} -o eth0 -j MASQUERADE" /etc/iptables/rules.v4
```

If the file `/etc/iptables/rules.v4` doesn't exist, use the following:
```sh
cat <<EOF | sudo tee -a /etc/iptables/rules.v4
# nat Table rules
*nat
:POSTROUTING ACCEPT [0:0]

# Forward traffic from VM subnets through eth0.
-A POSTROUTING -s ${localSubnet}/${subnetMask} -o eth0 -j MASQUERADE

# don't delete the 'COMMIT' line or these nat table rules won't be processed
COMMIT
EOF
```

You can apply the rules with the command:
```sh
sudo iptables-restore /etc/iptables/rules.v4
```

# Cleanup
Cleanup stuff.
```sh
unset ORIGINAL
unset newVM
unset localSubnet
unset subnetMask
unset defaultGateway
unset localBroadcast
unset BRIDGE
unset VXLAN_ID
unset VXLAN_NAME
unset VTEP1_ADDR
unset VTEP2_ADDR
unset VTEP_DEV
```

# Destroy the VMs (Optional)
If you want to completly destroy the VMs and remove the disk image, use the script below:
```sh
LIBVIRT_DIR="/var/lib/libvirt/images"
for VM in "${!newVM[@]}"
do
  virsh --connect qemu:///system shutdown ${VM}
  STATUS=$(virsh --connect qemu:///system domstate ${VM})
  while ([ "${STATUS}" != "shut off" ] )
  do
    STATUS=$(virsh --connect qemu:///system domstate ${VM})
    printf "   Status of VM [%s] is: %s\n" "${VM}" "${STATUS}"
    sleep 2
  done
  printf "[%s] is shutdown: %s\n" "${VM}" "${STATUS}"
  sudo virsh --connect qemu:///system undefine ${VM} --remove-all-storage --wipe-storage
  # Make sure both vars exist before deleting directory
  if ! [[ -z "${LIBVIRT_DIR}" || -z "${VM}" ]] ; then
    sudo rm -rf ${LIBVIRT_DIR}/${VM}
    printf "Success: Deleted directory %s/%s.\n" "${LIBVIRT_DIR}" "${VM}"
  else
    printf "Error: Can't delete directory %s/%s.\n" "${LIBVIRT_DIR}" "${VM}"
  fi
  virsh --connect qemu:///system pool-destroy ${VM}
  virsh --connect qemu:///system pool-undefine ${VM}
done
```

# Troubleshoot
Show bridge details:
```sh
ip -d link show br_etcd_1
```

Show port in bridge:
```sh
sudo brctl show br_etcd_1
```

```sh
ip -brief link
```

# BUGS
On Ubuntu 22.04 LTS, there's an annoying bug but with an easy fix.

If you get this message `Cannot call Open vSwitch: ovsdb-server.service is not running.` after executing the command `sudo netplan apply`, then apply this patch at the end of the file `/usr/share/netplan/netplan/cli/commands/apply.py`

The patch just checks if **Open vSwitch** is active before throwing the warning.
```
             if exit_on_error:
                 sys.exit(1)
         except OvsDbServerNotRunning as e:
- logging.warning('Cannot call Open vSwitch: {}.'.format(e))
+ if utils.systemctl_is_active('ovsdb-server.service'):
+   logging.warning('Cannot call Open vSwitch: {}.'.format(e))
```
