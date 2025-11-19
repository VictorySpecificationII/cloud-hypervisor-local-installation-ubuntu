# Cloud-hypervisor Configuration on Ubuntu

 - If you've not installed it yet, follow the section `First time installation`
 - If you've installed it, every time you boot into your OS, follow `On fresh host boot after installation`

## Reference

That's how my directory tree looks like - if you run into any trouble, this is what mine looks like.

```
antreas@sanctuary:~/Desktop/koyeb-stack$ pwd
/home/antreas/Desktop/koyeb-stack
antreas@sanctuary:~/Desktop/koyeb-stack$ tree . -L 1
.
├── cloud-hypervisor
├── noble-server-cloudimg-amd64.img
├── noble-server-cloudimg-amd64.raw
├── ref.txt
├── vm internet setup
└── vmlinux

```

## On fresh host boot after installation

Navigate to the cloud hypervisor folder and create a cloud-init

```
cd ~/Desktop/koyeb-stack/cloud-hypervisor
./scripts/create-cloud-init.sh
cd ..
```


Perform routing changes
```
# Create a tap interface on the host for VMs
sudo ip tuntap add dev tap0 mode tap

# Bring the tap interface up
sudo ip link set tap0 up

# Assign an IP to the tap interface to act as the gateway for VM subnet
sudo ip addr add 192.168.249.1/24 dev tap0   # host IP for the subnet

# Enable IP forwarding in the kernel so host can route packets between interfaces
sudo sysctl -w net.ipv4.ip_forward=1

#General NAT for any outgoing traffic on wlp7s0 (host Internet interface)
sudo iptables -t nat -A POSTROUTING -o wlp7s0 -j MASQUERADE

# NAT specifically for VM subnet (192.168.249.0/24) going out through host's Internet
sudo iptables -t nat -A POSTROUTING -s 192.168.249.0/24 -o wlp7s0 -j MASQUERADE

# Ensure IP forwarding is still enabled (redundant but safe)
sudo sysctl -w net.ipv4.ip_forward=1

# Verify that IP forwarding is active
cat /proc/sys/net/ipv4/ip_forward

# Make sure NAT for VM subnet is applied (redundant, safe to reapply)
sudo iptables -t nat -A POSTROUTING -s 192.168.249.0/24 -o wlp7s0 -j MASQUERADE

# List NAT rules to verify
sudo iptables -t nat -L -n -v

# Allow VM subnet (tap0) to forward packets out to host Internet (wlp7s0)
sudo iptables -A FORWARD -i tap0 -o wlp7s0 -j ACCEPT

# Allow responses from Internet back to VM subnet
sudo iptables -A FORWARD -i wlp7s0 -o tap0 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Fire up the VM

```
cloud-hypervisor \
  --kernel vmlinux \
  --disk path=noble-server-cloudimg-amd64.raw path=/tmp/ubuntu-cloudinit.img \
  --cmdline "console=hvc0 root=/dev/vda1 rw" \
  --cpus boot=1 \
  --memory size=512M \
  --net "tap=tap0,mac=12:34:56:78:90:ab"
```


# First time installation

Create a directory in your Desktop, name it `koyeb-stack` and cd into it - all the steps take part there

## Virtualization support

To check the status of KVM, you can run

```
kvm-ok
# Should return something like:
# INFO: /dev/kvm exists
# KVM acceleration can be used
```

## Dependencies

Many apt packages that are necessary. Install the following:

```
sudo apt-get install -y \
  build-essential mtools cloud-image-utils libssl-dev whois
```

## Cloud-hypervisor installation

We need to install the cloud-hypervisor binary on the machine

```
wget https://github.com/cloud-hypervisor/cloud-hypervisor/releases/latest/download/cloud-hypervisor
chmod +x cloud-hypervisor
sudo setcap cap_net_admin+ep ./cloud-hypervisor
sudo mv cloud-hypervisor /usr/local/bin/
```

Test with

```
cloud-hypervisor --version
```


## Cloud init template creation

We need to create a cloud-init ISO with a default user and password for the Ubuntu image. It also setups up some basic networking.

Clone the cloud-hypervisor repo and cd into it.

```
git clone https://github.com/cloud-hypervisor/cloud-hypervisor.git
cd cloud-hypervisor
```

Run the script to create the cloud init image ./scripts/create-cloud-init.sh

```
./scripts/create-cloud-init.sh
```

You now have a /tmp/ubuntu-cloudinit.img file.

Note: You can change the Ubuntu password by modifying the test_data/cloud-init/ubuntu/local/user-data file. The password is generated with mkpasswd --method=SHA-512 --rounds=4096 "your_desired_password"

## Download Kernel

Run 
```
cd ..
``` 

to go one directory back and not pollute the cloud-hypervisor installation.


We need a kernel to use for the VM. We can build one from source. Or we can use a pre-built one.

To avoid building from source, we can use grab the built kernel from the cloud-hypervisor linux repo. This is using the v6.2 release.


```
wget https://github.com/cloud-hypervisor/linux/releases/download/ch-release-v6.2-20240908/vmlinux
```


## Disk image

Get a pre-built Ubuntu image and convert it to a raw disk image. This is using a Ubuntu 24.04 image.

```
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
qemu-img convert -p -f qcow2 -O raw noble-server-cloudimg-amd64.img noble-server-cloudimg-amd64.raw
```

## Boot the VM

```
cloud-hypervisor \
	--kernel vmlinux \
	--disk path=noble-server-cloudimg-amd64.raw path=/tmp/ubuntu-cloudinit.img \
	--cmdline "console=hvc0 root=/dev/vda1 rw" \
	--cpus boot=1 \
	--memory size=512M \
	--net "tap=,mac=,ip=,mask="
```

The VM boot up and be prompted for the Ubuntu username and password. With the default cloud-init image, the username is cloud and the password is cloud123.

You should now have a shell into a fresh Ubuntu 24.04 instance.

```
uname -a
```

You can shutdown the VM with

```
sudo shutdown -h now
```

## Cloud-init regeneration

If you make any changes to networking, run


```
pushd cloud-hypervisor && ./scripts/create-cloud-init.sh && popd
```

## Internet access

On my ubuntu laptop, my internet interface is wlp7s0 - need to route and enable nat for internet access - using default networking settings on the vm - commands explained with comments

```
# Create a tap interface on the host for VMs
sudo ip tuntap add dev tap0 mode tap

# Bring the tap interface up
sudo ip link set tap0 up

# Assign an IP to the tap interface to act as the gateway for VM subnet
sudo ip addr add 192.168.249.1/24 dev tap0   # host IP for the subnet

# Enable IP forwarding in the kernel so host can route packets between interfaces
sudo sysctl -w net.ipv4.ip_forward=1

#General NAT for any outgoing traffic on wlp7s0 (host Internet interface)
sudo iptables -t nat -A POSTROUTING -o wlp7s0 -j MASQUERADE

# NAT specifically for VM subnet (192.168.249.0/24) going out through host's Internet
sudo iptables -t nat -A POSTROUTING -s 192.168.249.0/24 -o wlp7s0 -j MASQUERADE

# Ensure IP forwarding is still enabled (redundant but safe)
sudo sysctl -w net.ipv4.ip_forward=1

# Verify that IP forwarding is active
cat /proc/sys/net/ipv4/ip_forward

# Make sure NAT for VM subnet is applied (redundant, safe to reapply)
sudo iptables -t nat -A POSTROUTING -s 192.168.249.0/24 -o wlp7s0 -j MASQUERADE

# List NAT rules to verify
sudo iptables -t nat -L -n -v

# Allow VM subnet (tap0) to forward packets out to host Internet (wlp7s0)
sudo iptables -A FORWARD -i tap0 -o wlp7s0 -j ACCEPT

# Allow responses from Internet back to VM subnet
sudo iptables -A FORWARD -i wlp7s0 -o tap0 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

## VM Boot

Then we fire it up

```
cloud-hypervisor \
  --kernel vmlinux \
  --disk path=noble-server-cloudimg-amd64.raw path=/tmp/ubuntu-cloudinit.img \
  --cmdline "console=hvc0 root=/dev/vda1 rw" \
  --cpus boot=1 \
  --memory size=512M \
  --net "tap=tap0,mac=12:34:56:78:90:ab"
```

`ping 1.1.1.1` from inside the vm should pong.



Reference: https://jakerunzer.com/posts/getting-started-with-cloud-hypervisor
