# Network Security Monitoring - Engineering

## Environment

Welcome to NSM Engineer.

This environment is virtual - meaning that the stack and its hosts do not live on bare metal hardware. The NSM Stack in this course is built with containers to provide a simple environment for learning. The skills developed in this course will transate directly to building these tools on real hardware, but the networking and physical assembly of a kit is simulated.

There are a group of machine instances that will be configured into a full NSM Stack. These machines are hosted as LXC containers and will be used to emulate a bare metal stack build. The containers will be individually referred to as the "box" or "host" and collectively referred to as the "stack" or the "kit".

There is only one network in this environment. This network is built using a virtual bridge that all of the containers can communicate on. The network that is used by each host in the stack to communicate and pass data is the same network that will be used to capture traffic. This practice is commonly used in a development and testing environment and it is very normal when bulding the stack to set it to capture traffic on itself in order to test that the kit pipeline is functional.

The desktop that loads when the Strigo environment started is an Ubuntu instance. Think of this instance as a laptop that is used to connect to the switch of a server rack that is hosting all of the machines that are a part of the NSM Stack. The laptop is used as the main interface between the engineer and the stack. This instance will be referred to as the "Student Laptop"

## Containers

To get a list of available containers, use:

`lxc list`

To start the machine enter:

`lxc start <name of machine>`

Once a container is started it can be accessed via SSH using credentials:

- User: `elastic`
- Pass: `training`

## Host Information

Student Laptop

- Ubuntu 20.04

Sensor Specifications (sensor):

- 2 vCPU
- 6 GB RAM
- 2 NICS
  - Management Interface (eth0)
  - Traffic Capture Interface (eth1)

Elastic Cluster Specifications (elastic 1,2,3)

- 1 vCPU
- 4 GB RAM

Logstash Specifications (pipeline):

- 2 vCPU
- 4 GB RAM

Kibana Specifications (Kibana):

- 1 vCPU
- 2 GB RAM

Repo Specifications (repo):

- 1 vCPU
- 1 GB RAM
