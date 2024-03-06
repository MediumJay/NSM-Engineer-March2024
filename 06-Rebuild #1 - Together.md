# Step 1 - List your containers and check IPs
- `sudo lxc list`
- all IPs are randomized with exception of the repo

# Step 2 - Set Static IP for a container

- need to remove known_hosts first on the laptop since containers were reset
- `sudo vi /home/ubuntu/.ssh/known_hosts`
  - `:%d` wipes the file clean in vim
- `ssh elastic@<containerip>`

# Step 3 - Edit Eth0 interface

- `sudo vi /etc/sysconfig/network-scripts/ifcfg-eth0`
  - make it look like this: 
```
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=<containername>
NM_CONTROLLED=no
IPADDR=
GATEWAY=
PREFEX=
```

# Step 4 - Restart the network

- `sudo systemctl restart network`

# Step 5 - Repeat steps 3 through 4 for each container

# Step 6 - Edit the /etc/hosts file on "Laptop" to add the container hostnames + usernames
- `sudo vi /etc/hosts/`
- add the following: 
```
 Host repo
    HostName repo
    User elastic
  Host sensor
    HostName sensor
    User elastic
  Host elastic0
    HostName elastic0
    User elastic
  Host elastic1
    HostName elastic1
    User elastic
  Host elastic2
    HostName elastic2
    User elastic
  Host pipeline0
    HostName pipeline0
    User elastic
  Host pipeline1
    HostName pipeline1
    User elastic
  Host pipeline2
    HostName pipeline2
    User elastic
  Host kibana
    HostName kibana
    User elastic
```
- if these entries already exist then /etc/hosts is good

# Step 7 - Copy the /etc/hosts from the Laptop VM to the containers via the script
This is so we dont have to type IP addresses when SSH-ing

- run this script minus the repo hostname: 
```
for host in elastic{0..2} pipeline{0..2} kibana sensor; do sudo scp /etc/hosts elastic@$host:~/hosts && ssh -t elastic@$host 'sudo mv ~/hosts /etc/hosts && sudo systemctl restart network'; done
```

# Step 8 - The ssh-keygen was done already and so was the ssh config so move on
keys already generated (see 02-Configure Environment)

# Step 9 - Copy the ssh keys to all the containers
So we dont have to type password when ssh-ing
- run this script minus repo:
```
for host in sensor elastic{0..2} pipeline{0..2} kibana; do ssh-copy-id $host; done
```

# Step 10 - Push the LocalCA cert from repo container to the other containers

- certs already created and repo configured so skip that process
- ssh into the repo 
  - `ssh repo`
- use this script: 

```
for host in elastic{0..2} pipeline{0..2} kibana sensor; do sudo scp ~/certs/localCA.crt elastic@$host:~/localCA.crt && ssh -t elastic@$host 'sudo mv ~/localCA.crt /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust'; done
```

# Step 11 - Point the containers to the local repo

- `ssh <container name>`
- `mkdir ~/archive`
- `sudo mv /etc/yum.repos.d/* ~/archive/`
- `sudo vi /etc/yum.repos.d/local.repo`
  - paste this config in to set up the local repos: 

```
[local-base]
name=local-base
baseurl=https://repo/packages/local-base/
enabled=1
gpgcheck=0

[local-rocknsm-2.5]
name=local-rocknsm-2.5
baseurl=https://repo/packages/local-rocknsm-2.5/
enabled=1
gpgcheck=0

[local-elasticsearch-7.x]
name=local-elasticsearch-7.x
baseurl=https://repo/packages/local-elastic-7.x/
enabled=1
gpgcheck=0

[local-epel]
name=local-epel
baseurl=https://repo/packages/local-epel/
enabled=1
gpgcheck=0

[local-extras]
name=local-extras
baseurl=https://repo/packages/local-extras/
enabled=1
gpgcheck=0

[local-updates]
name=local-updates
baseurl=https://repo/packages/local-updates/
enabled=1
gpgcheck=0
```
- `sudo yum makecache fast` 
- `sudo yum list suricata` - to check yum is puling from local repo


# Step 12 - Move on to the Sensor capture interface configuration

- `ssh sensor`

- `sudo yum install ethtool -y` - to show features of interfaces

- Check if checksumming feature is turned on
  - `sudo ethtool -k eth1`
  - notice it is on so download script from repo to disable

- `sudo curl -LO https://repos/fileshare/interface.sh`

- `sudo chmod +x interface.sh`

- `sudo ./interface.sh eth1`

- `sudo ethtool -k eth1` - again to check



# Step 13 - Make the checksum disable persistent by creating a script

- `sudo vi /sbin/ifup-local`
  - insert this script: 
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
- make it executable
  - `sudo chmod +x /sbin/ifup-local`



# Step 14 - Modfy the ifup script to call our ifup-local script

- `sudo vi /etc/sysconfig/network-scripts/ifup`
  - shift + g + o to go to bottom and then insert one line above the "exec" line
  - add this change: 
```
if [ -x /sbin/ifup-local ]; then
/sbin/ifup-local ${DEVICE}
fi
```



# Step 15 - Edit interface Eth1
- `sudo vi /etc/sysconfig/network-scripts/ifcfg-eth1`
- add these changes:
```
DEVICE=eth1
BOOTPROTO=none
ONBOOT=yes
NM_CONTROLLED=no
TYPE=Ethernet
```

# Step 16 - Restart the network to verify the checksumming remains off
- `sudo systemctl restart network`
- `sudo ethtool -k eth1`
- notice that the checksumming remains off. Yippee
- can also see via `ip a` that eth1 is in promisc mode

# Step 17 - test our capture inteface
- we will use tcpdump to monitor our network traffic and test inteface
- `sudo tcpdump -nn -i eth1`
  - we are seeing SSH traffic so we will add a filter
  - `sudo tcpdump -nn -i eth1 '!port 22'`


# Summary 
- We statically set the eth0 interface on all containers + disabled dhcp
- We edited /etc/hosts and copied to all containers so we dont have to type IPs anymore when SSH-ing
- The ssh keys + config were already created so we copied the keys to all the containers so we dont have to type passwords anymore when SSH-ing
- The repo and CA/certs were already configured so we copied the localCA.crt to all the containers so they will trust the local CA
- We pointed all the containers to use the local repo container by editing their /etc/yum.repos.d/ directory
- We then configured the capture interface to have no checksumming, be in promisc mode, and verify it is sensing traffic
