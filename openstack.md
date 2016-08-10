# Part III - Deploying on OpenStack

Welcome to the Stark & Wayne guide to deploying Cloud Foundry on OpenStack.

Before beginning with BOSH and different deployments, you need to set up your OpenStack environment. There are a few options you could use:

1. [DevStack On a VM][1] - This is a very simplified Openstack built into an Ubuntu image that gets set up and run with a single command in a VM. Note that this is for playing around with small environments only. 
2. [DevStack][2] - The full version of DevStack, fully configurable. DevStack is the developer version of OpenStack. It is what we will use for the documentation.
3. [OpenStack Marketplace][3] - If you don't mind sacrificing some low level configuration for a quicker 'up' time, then pick one of the many public clouds that are already hosted. Each has different 'flavors' and components of the main OpenStack.

Like stated, we will be using Option 2, the fully configurable DevStack. We will be running it on a single machine, but if you want a multi-node cluster, then you can follow [these documents][4].

## DevStack on a Machine
So to get started, you will need a dedicated Linux machine - I have an HP laptop with 16GB of RAM which runs it fine. To begin, you will need to be connected to an ethernet cord and know the network interface name for it, and you have to configure/know your IP address on your network. Ethernet cable I can't help you out much with, but to figure out the network interface, run the following command. Chances are it will either be `eth0` or `eno1`.

```bash
$ ifconfig
eno1      Link encap:Ethernet  HWaddr 50:65:f3:b9:af:93  
          inet addr:192.168.1.16  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fe80::b92:bc13:6620:18f2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:7751 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6381 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:7913321 (7.9 MB)  TX bytes:693764 (693.7 KB)
wlo1      Link encap:Ethernet  HWaddr d0:7e:35:a0:33:d6  
          inet addr:192.168.1.3  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fe80::6241:8b70:684d:464e/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:107 errors:0 dropped:0 overruns:0 frame:0
          TX packets:71 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:9339 (9.3 KB)  TX bytes:10114 (10.1 KB)
```

Running Ubuntu 16.04, my network interface name was `eno1`. Before 15, it was named `eth1` so its most likely one or the other. 

As for your IP address, I suggest setting it to a static IP so that your DHCP reassignments don't mess up your DevStack implementation, but using your DHCP IP is okay for development, those instructions are listed below. To do so, open up `/etc/network/inferfaces` and add these lines below whatever is there.

```
# Code from before changing anything
..................

# My IP description
# IPv4 address
iface eth0 inet static
	address	192.168.1.100    # Set this to whatever IP you have available/need, make sure its in your network range.
	netmask	255.255.255.0	   # Give yourself all 255 addresses
	network	192.168.1.0  	   # What network you want/need
	broadcast 192.168.1.255  
	gateway	192.168.1.1      # Gateway to your network, usually x.x.x.1
```

If you don't want to set a static IP, you can see your current IP with `ip addr show` and it will display your current IP address(es). Pick the one that matches your network interface.


## DevStack Setup



[1]: https://github.com/makelinux/devstack-install-on-iso
[2]: http://docs.openstack.org/developer/devstack/
[3]: http://www.openstack.org/marketplace/
[4]: http://docs.openstack.org/developer/devstack/guides/multinode-lab.html
