# Step 1 - Start the containers

- lxc list - Lists the containers
- lxc start `<containername>`
  - lxc start `<containername>{0..1}`


# Step 2 - SSH into each node and change IPs

- Edit ifcfg-eth0 (management interface)
  - cd /etc/sysconfig/network-scripts/
  - vi ifcfg-eth0
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
  MTU= 
  DHCP_HOSTNAME=elastic
  IPADDR=<desired static ip>

  not necessary i dont think
  NETMASK=<subnet mask>
  GATEWAY=<gateway ip:>
  ``` 
- Restart the networking service
  - sudo systemctl restart network
- Test new IP was assigned
  - ping `<new ip>`
- SSH again to also verify connectivity


# Step 3 - Edit /etc/hosts file to tie hostnames to IPs

- sudo vi /etc/hosts
- add the following additions to the file
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
  - Ex) ssh elastic@kibana 

# Step 4 - Use a script to copy /etc/hosts file to all the containers

- Create a script
  - sudo touch hosts_move_script.sh
  - sudo chmod +x hosts_move_script.sh

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

# Step 5 - Create an SSH Config file
We are doing this because...
Allows us to specify the username for the elastic service? 

-


# Step 6 - Generate an SSH Key pair
We are doing this because...

- 

# Step 7 - Use a script to copy the SSH files to all VMs

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