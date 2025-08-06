## Prerequisites
---
1. Install dependencies
```bash
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager virtinst
```

2. Verify that CPU supports KVM virtualization:
```bash
kvm-ok
```
3. (Optional) add user to libvirt and kvm groups to allow managing VM without using sudo
```
sudo adduser $(whoami) libvirt
sudo adduser $(whoami) kvm
```


## Network Bridging
---
### Why?
Network bridging is needed so that the VM can have its own IP. It works by adding a virtual network switch that acts like a bridge between the host+VM to the NIC

1. Identify the interface of the IP address (e.g. eno1, eth0) using `ip a`
2. Configure netplan for bridging
	- Go to /etc/netplan to find the netplan configuration plan (.yaml)
	- (optional) make a backup of the current netplan
		`sudo cp /etc/netplan/your-config-file.yaml /etc/netplan/your-config-file.yaml.bak`
	- Edit the configuration plan. Example:
```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    enp2s0:  # <--- !! REPLACE with your network interface !!
      dhcp4: no
      dhcp6: no
  bridges:
    br0:
      interfaces: [enp2s0]  # <--- !! REPLACE with your network interface !!
      dhcp4: yes
      dhcp6: no
      parameters:
        stp: true
        forward-delay: 4
```

3. Apply the new network configuration using `sudo netplan apply`. 
4. Verify that bridge interface has IP address with `ip a show br0`

# Creating The Headless Debian 12 Virtual Machine
---
1. Download ISO file
2. Create the QEMU bridge configuration file
	- Add "allow" rule for `br0` interface by adding `allow br0` to `/etc/qemu/bridge.conf` 
	- Set permission to root for better security
```
sudo chown root:root /etc/qemu/bridge.conf sudo chmod 0644 /etc/qemu/bridge.conf
sudo chmod u+s /usr/lib/qemu/qemu-bridge-helper
```
3. Create a virtual disk
```bash
qemu-img create -f qcow2 debian12-cli.qcow2 20G
```
4. Start the installation (adjust disk and iso path as needed)
```bash
qemu-system-x86_64 -m 2048 -cpu host -smp 2 \
-hda debian12-cli.qcow2 \
-cdrom debian-12.11.0-amd64-netinst.iso \
-boot d \
-netdev bridge,id=net0,br=br0 \
-device virtio-net-pci,netdev=net0 \
-display sdl --enable-kvm
```
5. Follow along with the installation
	- Uncheck "Debian desktop environment" for CLI-only installation
	- Uncheck "GNOME"
	- Check "SSH server and standard system utilities"
6. After the process is finished, we can enter the VM with the same command as installation, but without the cdrom argument
```bash
qemu-system-x86_64 -m 2048 -cpu host -smp 2 \
-hda debian12-cli.qcow2 \
-boot d \
-netdev bridge,id=net0,br=br0 \
-device virtio-net-pci,netdev=net0 \
-display sdl --enable-kvm
```
7. After installation is complete, install Sudo package and add user to the appropriate group
```bash
# Switch to root user first
su -
# Then install sudo
apt update
apt install sudo

usermod -aG sudo username
```
8. Relog or restart shell session, then verify sudo installation with `sudo whoami` 