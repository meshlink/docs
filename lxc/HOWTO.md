Meshlink Simulation Environment Setup
=====================================

# Requirements

Install required packages:

```sudo apt-get install lxc lxctl dnsmasq bridge-utils debootstrap```

Check that all requirements are met. Everthing should be green.

```sudo lxc-checkconfig```

If you're running debian and not Ubuntu, you'll have to set mount cgroups manually.

Add the following to ```/etc/fstab```:

```none		/sys/fs/cgroup	cgroup	defaults	0	0```

Then mount:
```mount /sys/fs/cgroup```

Run ```sudo lxc-checkconfig``` again to confirm.
Reference material for this can be found on the [Debian Wiki](https://wiki.debian.org/LXC).

# Set Up Virtual Network

## Create Bridge Adapter

Add the following to you ```/etc/network/interfaces```:

```
auto lxcbr0
iface lxcbr0 inet static
        pre-up brctl addbr lxcbr0 && brctl addif lxcbr0 eth0
        bridge_fd 0
        bridge_maxwait 0
        address 192.168.100.1
        netmask 255.255.255.0
        post-up iptables -A FORWARD -i lxcbr0 -s 192.168.100.1/24 -j ACCEPT
        post-up iptables -A POSTROUTING -t nat -s 192.168.100.1/24 -j MASQUERADE
        # add checksum so that dhclient does not complain.
        # udp packets staying on the same host never have a checksum filled else
        post-up iptables -A POSTROUTING -t mangle -p udp --dport bootpc -s 192.168.100.1/24 -j CHECKSUM --checksum-fill
```

This creates a birdge interface ```lxcbr0``` that all the containers will connect to.

Restart your networking: ```sudo service networking restart```.

Make sure to enable IP forwarding with ```sudo sysctl -w net.ipv4.ip_forward=1```

## Start DHCP on lxcbr0

Next, configure dnsmasq to run on ```lxcbr0```. Add the following to ```/etc/dnsmasq.conf```:

```
# Bind it to the LXC interface
interface=lxcbr0
# ip range with lease time
domain=192.168.100.0/24
dhcp-range=192.168.100.100,192.168.100.200,1h
# DHCP options
dhcp-option=40
```

Restart DHCP: ```sudo service dnsmasq restart```

# Create First Container

In the following command replace ```debian``` with ```ubuntu``` if you are on ubuntu. This is the template for the container.

```sudo lxc-create -n node0 -t debian```

## Configure First Container

Add the following to ```/var/lib/lxc/node0/config```:

```
lxc.network.type = veth
lxc.network.flags = up
lxc.network.link = lxcbr0
```

Confirm that ```/var/lib/lxc/node0/rootfs/etc/network/interfaces``` looks like this:

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
```

## Start First Container

Start the container with: ```sudo lxc-start -n node0 -d```

Connect to it with ```sudo lxc-console -n node0```

Confirm that the container has been assigned an IP in the 192.168.100.100-200 range.

Detach from the console with ```ctrl+a``` ```q```

## Install Meshlink on the first container
Stop the container again:

```sudo lxc-stop -n node0```


Then copy the meshlink sources from the parent to the container (from outside the container console).

Replace the paths as necessary to point to the meshlink sources.

```sudo cp -r /home/user/foo/meshlink /var/lib/lxc/node0/rootfs/root/```


Start to and connect to the contianer and run:

```apt-get install --assume-yes build-essential autoconf libssl-dev libtool ncurses-dev libreadline-dev liblzo2-dev ctags cscope texinfo```

Then install meshlink

# Create More Containers
Clone the first container as many times as required with:

```sudo lxc-clone -o node0 -n nodeX```
