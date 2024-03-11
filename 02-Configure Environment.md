# Step 1 - Start the containers

- `lxc list` - Lists the containers
- lxc start `<containername>`
  - lxc start `<containername>{0..1}`
  - `lxc start --all`


# Step 2 - SSH into each node and change IPs

- Edit ifcfg-eth0 (management interface)
  - `cd /etc/sysconfig/network-scripts/`
  - `vi ifcfg-eth0`
  - Original: 
  ```
  DEVICE=eth0
  BOOTPROTO=dhcp
  ONBOOT=yes
  HOSTNAME=<containername>
  NM_CONTROLLED=no
  TYPE=ethernet
  MTU= 
  DHCP_HOSTNAME=elastic
  ```
  - Make these Changes:
  ```
  DEVICE=eth0
  BOOTPROTO=none
  ONBOOT=yes
  HOSTNAME=<containername>
  NM_CONTROLLED=no
  TYPE=ethernet
  IPADDR=<desired static ip>
  GATEWAY=<gateway ip:> :(.1 in this case)
  PREFIX=24sudo
  ``` 
- Restart the networking service
  - `sudo systemctl restart network`
- Test new IP was assigned
  - ping `<new ip>`
- SSH again to also verify connectivity


# Step 3 - Edit /etc/hosts file to tie hostnames to IPs

- `sudo vi /etc/hosts`
- add the following additions to the file DELETE THE PREVIOUS
```
10.81.139.30 elastic0
10.81.139.31 elastic1
10.81.139.32 elastic2
10.81.139.40 pipeline0
10.81.139.41 pipeline1
10.81.139.42 pipeline2
10.81.139.50 kibana
10.81.139.10 repo
10.81.139.20 sensor
```
- test via ssh
  - Ex) `ssh elastic@kibana`

# Step 4 - Use a script to copy /etc/hosts file to all the containers

- Create a script
  - `sudo touch hosts_move_script.sh`
  - `sudo chmod +x hosts_move_script.sh`

```
#!/bin/bash

#array of container variables
container_ips=("10.81.139.30" "10.81.139.31" "10.81.139.32" "10.81.139.40" "10.81.139.41" "10.81.139.42" "10.81.139.50" "10.81.139.10" "10.81.139.20")

#start of for loop
for container_ip in "$container_ips[@]"; do

    #copy the hosts file to container
    scp /etc/hosts <username>@${container_ip}:/tmp/hosts_temp:

    #move the temp file to replace the original hosts file
    ssh -t <username>@${container_ip} 'sudo mv /tmp/hosts /etc/hosts'

    #restart the network service
    ssh -t <username>@${container_ip} 'sudo systemctl restart network'
done

```


OR

```
for host in elastic{0..2} pipeline{0..2} kibana sensor repo; do sudo scp /etc/hosts elastic@$host:~/hosts && ssh -t elastic@$host 'sudo mv ~/hosts /etc/hosts && sudo systemctl restart network'; done
```


# Step 5 - Create an SSH Config file
We are doing this so we dont have to type the username when SSHing to other boxes


- Create the SSH config file
  - `sudo vi ~/.ssh/config`
  - create the ssh config lines:
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
- test that you can SSH by only hostname now
  - Ex) sudo ssh repo

# Step 6 - Generate an SSH Key pair
We are doing this because... it will let us not have to type in the account password on every box.

`ssh-keygen`

- create the ssh keypair
  - you can just press enter through this command output




# Step 7 - Use a script to copy the SSH key files to all VMs

```
#!/bin/bash

#array of container variables
containers=("sensor" "repo" "elastic0" "elastic1" "elastic2" "pipeline0" "pipeline1" "pipeline2" "kibana")

#start of for loop
for container in "$containers[@]"; do

    #copy the ...
    #idk about this command:
    #ssh -t <username>@${container} 'ssh-copy-id $container'

    #instructors command
    ssh-copy-id $container
done
```

OR 


- copy the ssh keys to all the other VMs

```
for host in sensor repo elastic{0..2} pipeline{0..2} kibana; do ssh-copy-id $host; done
```



# TO GET INTO HOSTS VIA LXC

- lxc exect <HOSTNAME> /bin/bash