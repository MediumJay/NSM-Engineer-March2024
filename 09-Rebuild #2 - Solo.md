# Step 1 - List your containers and check IPs
- `sudo lxc list`
- all IPs are randomized with exception of the repo

# Step 2 - Set Static IP for a container

- need to remove known_hosts first on the laptop since containers were reset
- `sudo vi /home/ubuntu/.ssh/known_hosts`
  - `:%d` wipes the file clean in vim
- `sudo vi /root/.ssh/known_hosts`
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
- pastable for time saving: 
```
mkdir ~/archive;sudo mv /etc/yum.repos.d/* ~/archive/
```

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
- pastable for quick access:
```
sudo yum makecache fast; sudo yum list suricata
```


# Step 12 - Move on to the Sensor capture interface configuration

- `ssh sensor`

- `sudo yum install ethtool -y` - to show features of interfaces

- Check if checksumming feature is turned on
  - `sudo ethtool -k eth1`
  - notice it is on so download script from repo to disable

- `sudo curl -LO https://repo/fileshare/interface.sh`

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



# Step 14 - Modify the ifup script to call our ifup-local script

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

# Step 17 - test our capture interface
- we will use tcpdump to monitor our network traffic and test inteface
- `sudo tcpdump -nn -i eth1`
  - we are seeing SSH traffic so we will add a filter
  - `sudo tcpdump -nn -i eth1 '!port 22'`


# STENOGRAPHER STENOGRAPHER STENOGRAPHER STENOGRAPHER STENOGRAPHER STENOGRAPHER


# Step 18 - Get onto the Sensor to configure Stenographer
- `ssh sensor`

# Step 19 - Verify sensor is pulling from local-rocknsm repo
- `sudo yum list stenographer`

# Step 20 - Install Stenographer
- `sudo yum install stenographer -y`

# Step 21 - Begin configuring Stenographer by editing the config file 
- Edit the config file
  - `sudo vi /etc/stenographer/config`
- make the following changes:
```
"PacketsDirectory": "/data/stenographer/packets"
"IndexDirectory": "/data/stenographer/index"
"DiskFreePercentage": 30 
"Interface":"eth1"
```
- Note: DiskFreePercentage is just the amount of disk that must always remain free


# Step 23 - Run stenographer script file to generate keys for steno user and group
- These keys are needed for...
- `sudo stenokeys.sh stenographer stenographer`
  - specify the user (stenographer) and the group (stenographer)
- verify keys were installed
  - `ll /etc/stenographer/certs/`

# Step 24 - Create the directories that the stenographer config file is referencing
- `sudo mkdir -p /data/stenographer/{packets,index}`
  - {} just makes 2 directories at the same time
- notice root owns these two directories

# Step 25 - Give Stenographer user and group ownership to packets and index directories
- `sudo chown -R stenographer:stenographer /data/stenographer`

# Step 26 - Start and Enable Stenographer service
- `sudo systemctl start stenographer`
- `^start^enable` - linux kungfu to replace start with enable

# Step 27 - Verify Stenographer is capturing packets
- On sensor `ping 8.8.8.8 -c 5`
- Carve it `sudo stenoread 'host 8.8.8.8' -nn`
- Traffic can take a while to carve. 
- Rerun stenoread command to eventually see the traffic. 
- 5+ minutes and a config might be messed up. 


# Step 28 - Verify another way
- check our directories to see if they're being written
- `ll /data/stenographer/packets`
- `ll /data/stenographer/index`

# SURICATA SURICATA SURICATA SURICATA SURICATA SURICATA 

# Step 29 - Get on the Sensor
- `ssh sensor` 

# Step 30 - Install Suricata
- `sudo yum install suricata -y`

# Step 31 - Configure the Suricata.yaml file
- `sudo vi /etc/suricata/suricata.yaml`
- This config is HUGE (version 5.0.1 for class)
- make the following changes on the respective lines:
  - `:###` will help you jump around 
```
56 default-log-dir: /data/suricata/
60 enabled: no (turns off global statistics)
76 enabled: no (turns off fast.log)
404 enabled: no (turns off statistics log)
557 enabled: no (turns console output off)
580 - interface: eth1 (sets capture int for af-packet)
582 threads: 3 (uncomment and set to 3)
981 run-as (uncomment)
982 user: suricata (uncomment and change to suricata)
983 group: suricata (uncomment and change to suricata)

1434 set-cpu-affinity: yes (turns on CPU pinning)
1452 cpu: ["0-2"] (pinning cores 0 through 2)
1458 low [0] (sets affinity???)
1459 medium: [1] (sets affinity???)
1460 high [2] (sets affinity???)

1461 default: "high" (sets affinity???)
1500 enabled: no (turns off rule_perf log)
1516 enabled: no (turns off keyword_perf log)
1521 enabled: no (turns off prefilter_perf log) 
1527 enabled: no (turns off rule_group_perf log)
1536 enabled: no (turns off packet_stats log)
```

# Step 32 - Configure /etc/sysconfig/suricata
- This file plugs in the options into the service file so we can use af-packet and run it as user/group suricata
  - the service file is what configures services when managed by systemctl
- `sudo vi /etc/sysconfig/suricata`
- make the following change: 
```
OPTIONS="--af-packet=eth1 --user suricata --group suricata" 
```

# Step 33 - Import rules into Suricata
- we moved the emerging rules file 
to the fileshare
- we want to download this to import into Suricata
  - `sudo suricata-update add-source emergingthreats https://repo/fileshare/emerging.rules.tar.gz`
- Once downloaded we need to do the import 
  - `sudo suricata-update` 
- Rules will be loaded and then tested with -T

# Step 34 - Create the data directory for Suricata
- `sudo mkdir /data/suricata`

# Step 35 - Change permissions of the /data/suricata directory
- `sudo chown -R suricata: /data/suricata`
  - suricata: will change user and group to suricata since they are identical 


# Step 36 - Start and enable Suricata
- `sudo systemctl enable --now suricata` to start+enable

# Step 37 - Checkout logs with Journalctl (for errors)
- `journalctl -xeu suricata`

# Step 38 - Generate some traffic and Look at eve.json
- `ping 8.8.8.8; curl google.com`
- `sudo cat /data/suricata/eve.json`

# Summary