## cumulus-its-centralized

### This repo has been deprecated. Please check out the following Nvidia repos:
```
https://gitlab.com/cumulus-consulting/goldenturtle/cumulus_ansible_modules
https://gitlab.com/cumulus-consulting/goldenturtle/cumulus_ansible_modules/-/tree/master/inventories/evpn_centralized
```
### Summary:

  - Cumulus Linux 3.7.11
  - Underlying Topology Converter to 4.7.0
  - Tested against Vagrant 2.1.5 on Mac and Linux. Windows is not supported.
  - Tested against Virtualbox 5.2.32 on Mac 10.14
  - Tested against Libvirt 1.3.1 and Ubuntu 16.04 LTS

### Description:

  This is an Ansible demo which configures client isolation using VRF and Centralized VXLAN to accomplish microsegmentation and multitenancy.

### Network Diagram:

![Network Diagram](https://github.com/chronot1995/cumulus-its-centralized/blob/master/documentation/cumulus-its-centralized.png)

### Install and Setup Virtualbox on Mac

Setup Vagrant for the first time on Mojave, MacOS 10.14.6

1. Install Homebrew 2.1.9 (This will also install Xcode Command Line Tools)

    https://brew.sh

2. Install Virtualbox (Tested with 5.2.32)

    https://www.virtualbox.org

I had to go through the install process twice to load the proper security extensions (System Preferences > Security & Privacy > General Tab > "Allow" on bottom)

3. Install Vagrant (Tested with 2.1.5)

    https://www.vagrantup.com

### Install and Setup Linux / libvirt demo environment:

First, make sure that the following is currently running on your machine:

1. This demo was tested on a Ubuntu 16.04 VM w/ 4 processors and 32Gb of Diagram

2. Following the instructions at the following link:

    https://docs.cumulusnetworks.com/cumulus-vx/Development-Environments/Vagrant-and-Libvirt-with-KVM-or-QEMU/

3. Download the latest Vagrant, 2.1.5, from the following location:

    https://www.vagrantup.com/

### Initializing the demo environment:

1. Copy the Git repo to your local machine:

```
    git clone https://github.com/chronot1995/cumulus-its-centralized/
```

2. Change directories to the following

```
    cumulus-its-centralized
```

3a. Run the following for Virtualbox:

```
    ./start-vagrant-vbox-poc.sh
```

3b. Run the following for Libvirt:

```
    ./start-vagrant-libvirt-poc.sh
```

### Running the Ansible Playbook

1a. SSH into the Virtualbox oob-mgmt-server:

```
    cd vx-vbox-simulation
    vagrant ssh oob-mgmt-server
```

1a. SSH into the Libvirt oob-mgmt-server:

```
    cd vx-libvirt-simulation  
    vagrant ssh oob-mgmt-server
```

2. Copy the Git repo unto the oob-mgmt-server:

```
    git clone https://github.com/chronot1995/cumulus-its-centralized
```

3. Change directories to the following

```
    cumulus-its-centralized/automation
```

4. Run the following:

```
    ./provision.sh
```

This will run the automation script and configure the environment.

### Troubleshooting

Helpful VRF-enabled NCLU troubleshooting commands:

- net show route
- net show route vrf <name>
- net show bgp summary
- net show bgp vrf <name> summary
- net show interface | grep -i UP
- net show lldp

Helpful Linux troubleshooting commands:

- ip route
- ip link show
- ip address <interface>

The BGP Summary command for a VRF will show if each switch has formed an IPv4 neighbor relationship within that VRF:

One should see all of the routes in the default VRF with the following:

```
cumulus@border01:mgmt-vrf:~$ net show route bgp
RIB entry for bgp
=================
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR,
       > - selected route, * - FIB route

B>* 10.1.1.1/32 [20/0] via fe80::4638:39ff:fe00:5, swp3, 00:14:20
B>* 10.2.2.2/32 [20/0] via fe80::4638:39ff:fe00:5, swp3, 00:14:20
B>* 10.3.3.3/32 [20/0] via fe80::4638:39ff:fe00:5, swp3, 00:14:20
B>* 10.5.5.5/32 [20/0] via fe80::4638:39ff:fe00:8, swp1, 00:14:20
```

One can also view the VNIs that are established on all of the switches:

```
cumulus@leaf01:mgmt-vrf:~$ net show evpn vni
VNI        Type VxLAN IF              # MACs   # ARPs   # Remote VTEPs  Tenant VRF
22         L2   vni22                 4        4        2               default
11         L2   vni11                 4        4        2               default
```
```
cumulus@leaf02:mgmt-vrf:~$ net show evpn vni
VNI        Type VxLAN IF              # MACs   # ARPs   # Remote VTEPs  Tenant VRF
22         L2   vni22                 4        4        2               default
11         L2   vni11                 4        4        2               default
```

```
cumulus@border01:mgmt-vrf:~$ net show evpn vni
VNI        Type VxLAN IF              # MACs   # ARPs   # Remote VTEPs  Tenant VRF
22         L2   vni22                 3        4        2               default
11         L2   vni11                 3        4        2               default
```

Since this is a centralized design, the border will contain L2 VNIs to transmit the Type 2 and Type 3 routes for the default gateway.

In the below, server01 is tracing to server02, which is on the same leaf, but in a different VRF:

```
traceroute to 192.168.22.111 (192.168.22.111), 30 hops max, 60 byte packets
 1  192.168.11.1 (192.168.11.1)  8.775 ms  8.662 ms  8.547 ms
 2  192.168.22.111 (192.168.22.111)  12.878 ms  12.872 ms  12.863 ms
```
The above traffic goes to the default gateway, which is border01, and then takes VNI22 down to server02.

The next output is an intra-VRF traceroute between server01 and server03. This traceroute takes the VXLAN tunnel directly from server01 to server03:

```
cumulus@server01:~$ traceroute 192.168.11.222
traceroute to 192.168.11.222 (192.168.11.222), 30 hops max, 60 byte packets
 1  192.168.11.222 (192.168.11.222)  8.229 ms  8.104 ms  8.095 ms
```
Compared with the symmetric design, there are no Prefix / Type 5 routes in this design:

```
cumulus@leaf01:mgmt-vrf:~$ net show bgp evpn route type prefix
No EVPN prefixes (of requested type) exist
```

### Errata

1. To shutdown the demo, run the following command from the vx-simulation directory:

```
vagrant destroy -f
```

2. This topology was configured using the Cumulus Topology Converter found at the following URL:

    https://github.com/CumulusNetworks/topology_converter

3. The following command was used to run the Topology Converter within the appropriate vx-sim directory:

```
     ./topology_converter.py cumulus-its-centralized.dot -c --provider=virtualbox
     ./topology_converter.py cumulus-its-centralized.dot -c --provider=libvirt
```

After the above command is executed, the following configuration changes are necessary:

4. Within "<vx-sim>/helper_scripts/auto_mgmt_network/OOB_Server_Config_auto_mgmt.sh"

The following stanza:

echo " ### Creating cumulus user ###"
useradd -m cumulus

Will be replaced with the following:

echo " ### Creating cumulus user ###"
useradd -m cumulus -m -s /bin/bash

The following stanza:

    #Install Automation Tools
    puppet=0
    ansible=1
    ansible_version=2.6.3

Will be replaced with the following:

    #Install Automation Tools
    puppet=0
    ansible=1
    ansible_version=2.9.3

Add the following ```echo``` right before the end of the file.

    echo " ### Adding .bash_profile to auto login as cumulus user"
    echo "sudo su - cumulus" >> /home/vagrant/.bash_profile
    echo "exit" >> /home/vagrant/.bash_profile
    echo "### Adding .ssh_config to avoid HostKeyChecking"
    printf "Host * \n\t StrictHostKeyChecking no\n" >> /home/cumulus/.ssh/config

    echo "############################################"
    echo "      DONE!"
    echo "############################################"
