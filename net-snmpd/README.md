# Deployment of Net-SNMP for Platina Cluster Heads

## Overview

This directory contains an Ansible playbook for deployment of Net-SNMP on a Platina Cluster Head.  Once pulled to a Platina Cluster Head, the same playbook can be used to configure multiple CLuster Heads as long as there connectivity. This playbook also includes a an extension module and MIB for additional stats and metrics not available by default within Net-SNMP.

This playbook can currently be executed by hand, but will eventually be rolled into the PCC orchestration framework.

## Dependencies
We need to ensure that Clusterhead's BMC has an IP address configured so we can talk to the Redis database. Here are the steps to ensure BMC is configured:
1.Access the Clusterhead's BMC console port (UART) via a Console server.  From the command line use the command below to switch to the BMC console.
```
platina@clusterhead1:~$ sudo goes toggle
```
1.Check if the BMC's eth0 is configured
```
platina-mk1-bmc> ip addr show
1: lo: <loopback,up,lower-up> mtu 65536 qdisc noqueue state unknown mode default group 0 qlen 1000

    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

    inet 127.0.0.1/8 scope host lo

       valid_lft forever preferred_lft forever

    inet6 ::1/128 scope host

       valid_lft forever preferred_lft forever
```
1.Set up an IP for BMC console port and Reboot.
```
To assign a static IP address to the BMC processor, enter the following commands at the BMC console:
ipcfg -ip [ipaddress]::[gateway]:[mask]::eth0:on
reboot
Example:
platina-mk1-bmc> ipcfg -ip 192.168.4.211::192.168.4.1:255.255.255.0::eth0:on
platina-mk1-bmc> reboot
```
1.Get back to your management console from the BMC:
```
platina@clusterhead1> toggle
```
1. Test the connectivity to the IPv4 address we just set up: 
```
platina@clusterhead1:~$ ping 192.168.4.211

PING 192.168.4.211 (192.168.4.211) 56(84) bytes of data.

64 bytes from 192.168.4.211: icmp_seq=1 ttl=64 time=2.31 ms
```
1. Use the command below to query the IPv6 Link Local address:
```
platina@clusterhead1:~$ sudo goes mac-ll

       Base MAC: 50:18:4c:00:56:0c

IPv6 link-local: fe80::5218:4cff:fe00:560c
```
1. Ping the IPv6 Link Local address: 
```
platina@clusterhead1:~$ ping6 fe80::5218:4cff:fe00:560c%eth0

PING fe80::5218:4cff:fe00:560c%eth0(fe80::5218:4cff:fe00:560c%eth0) 56 data bytes

64 bytes from fe80::5218:4cff:fe00:560c%eth0: icmp_seq=1 ttl=64 time=2.32 ms
```
1.Ensure ansible is installed.
```
To install ansible, the following is the command:
apt-get install ansible
```
1. Test the ansible connectivity among all the the Cluster Heads using SSH, on which snmpd is to be configured(It is recommended to configure the passwordless ssh- i.e using key-based ssh into the Cluster Heads).

## Execution

1. Pull this repo from github.
1. Update the given `host_vars.yml` with your own host variables(Ex. communityString, managementIP, etc.)  See example below.
1. In the `inventory.ini` file, include all the Cluster Heads on which you want to install the snmpd, so that all  get configured at the same time.  See example below. 
1. Run the Ansible playbook (`snmpd.yml`) using the following command: `ansible-playbook -i ./inventory.ini ./snmpd.yml`, where `snmpd.yml` is the playbook and `./inventory.ini` is an Ansible-format inventory file indicating the Cluster Head(s) you wish Net-SNMP be deployed to.
1. The included `Platina-MIB.txt` file is an SNMP MIB file that may be used with the SNMP client of your choice.


## inventory.ini Input Parameters
The provided playbook makes use of the following variables as its input from the `inventory.ini`:
```
---
192.168.1.49
192.168.1.50

```
* each entry above references the mgmt IP of a Cluster Head you would like NET-SNMP installed

## host_vars Input Parameters
The provided playbook makes use of the following variables as its input from the `host_vars`:

```
---
managementIp: 192.168.1.49
communityString: public
location: SantaClara
email: customer@customer.com
contact: John_Smith
include_nonfree: true

```
* community\_string -- this is where you set the snmp v2 community string.
* management\_ip -- this is the IP address of the management interface. This playbook binds the SNMP service to the management interface to reduce attack surface.
* location -- this is the standard SNMP location field
* contact -- this is the standard SNMP contact field
* email -- this is the standard SNMP contact email address field
* include\_nonfree -- some of the components this playbook deploys are in the Debian non-free repository. When set to _true_, this variable causes the playbook to add the non-free repository and install packages from there (primarily third party SNMP MIBs). These MIBs are not required for normal operation.

## Sample SNMP query

```
    snmpbulkwalk -v2c -c public 172.17.2.69 -m PLATINA-MIB .1.3.6.1.4.1.48226
```

Where:
* _public_ is the community name,
* _203.0.113.10_ is the management IP of the cluster heaaad,
* PLATINA-MIB is the module name in the Platina MIB
* _.1.3.6.1.4.1.48226_ is the OID of the Platina OID space

