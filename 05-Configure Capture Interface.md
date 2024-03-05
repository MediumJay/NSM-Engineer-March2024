
# Step 1 - SSH into the Sensor
- `ssh sensor`

# Step 2 - Install Ethtool
- `sudo yum install ethtool -y`


# Step 3 - Examine interfaces on VM and Download Script
- Sensor is only VM with 2 interfaces
  - view with `ip a`
- eth1 is the one we will configure for capturing
- use ethtool to look at the interface
  - `sudo ethtool -k eth1`
- notice checksumming is on; we want it off so CPU isnt overloaded
  - curl down a script from browser
  - `sudo curl -LO https://repo/fileshare/interface.sh` L link O output




# Step 4 - Run the script to turn off checksumming/offloading
- make it executable
  - `sudo chmod +x interface.sh`
- run the script with eth1 parameter
  - `sudo ./interface.sh eth1`
  - "Operation not supported" is fine

# Step 5 - Create a script to make sure changes are persistent through power cycle
- sudo vi `/sbin/ifup-local`
  - insert these changes: 
```
#!/bin/bash
if [[ "$1" == "eth1" ]]
then
for i in rx tx sg tso ufo gso gro lro rxvlan txvlan
do
/usr/sbin/ethtool -K $1 $i off
done
/usr/sbin/ethtool -N $1 rx-flow-hash udp4 sdfn
/usr/sbin/ethtool -N $1 rx-flow-hash udp6 sdfn
/usr/sbin/ethtool -n $1 rx-flow-hash udp6
/usr/sbin/ethtool -n $1 rx-flow-hash udp4
/usr/sbin/ethtool -C $1 rx-usecs 10
/usr/sbin/ethtool -C $1 adaptive-rx off
/usr/sbin/ethtool -G $1 rx 4096

/usr/sbin/ip link set dev $1 promisc on

fi
```
- this script is very similar to interface.sh, additional does promisc on 
- make it executable
  - `sudo chmod +x /sbin/ifup-local`
- have to modify another script so it can call our ifup-local script
  - `sudo vi /etc/sysconfig/network-scripts/ifup`
  - shift + g + o to go to bottom and then one line above the "exec" line
  - add this change: 
```
if [ -x /sbin/ifup-local ]; then
/sbin/ifup-local ${DEVICE}
fi
```

# Step 6 - Edit interface Eth1
- `sudo vi /etc/sysconfig/network-scripts/ifcfg-eth1`
- add these changes:
```
DEVICE=eth1
BOOTPROTO=none
ONBOOT=yes
NM_CONTROLLED=no
TYPE=Ethernet
```

# Step 7 - Restart the network to verify the checksumming remains off
- `sudo systemctl restart network`
- `sudo ethtool -k eth1`
- notice that the checksumming remains off. Yippee
- can also see via `ip a` that eth1 is in promisc mode

# Step 8 - test our capture inteface
- we will use tcpdump to monitor our network traffic and test inteface
- `sudo tcpdump -nn -i eth1`
  - the nn doesnt resolve ip or port to names
  - we are seeing SSH traffic so we will add a filter
  - `sudo tcpdump -nn -i eth1 '!port 22'`

# Summary 


Capture Interface (Sensor) Summary
- Ethtool install
- interface.sh: Disable nic-offloading
- /sbin/ifup-local: Make settings persistent
- /etc/sysconfig/network-scripts/ifup: call ifup-local
- /etc/sysconfig/network-scripts/ifcfg-eth1: capture interface config

End goal: capture traffic with sniffing interface using tcpdump
sudo tcpdump -nn -i eth1 '!port 22'