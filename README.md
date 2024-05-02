## Transit Gateway Connect Attachment with Linux using FRRouting and BGP over GRE

If you are looking to test TGW Connect attachment functionality it typically requires procuring 3rd party SD-WAN virtual appliances from Marketplace, obtain eval licenses etc, some of these steps are time consuming, setups are complex and prevents customers from testing this very important feature of TGW which is very commonly used to seamlessly integrate with SD-WAN fabrics.

The intent of this GitHub samples is to show how easy and seamlessly you can implement TGW Connect using Amazon Linux and [FRRouting (FRR)](https://frrouting.org/) which is a free and open source Internet routing protocol suite for Linux and Unix platforms which supports all features of BGP. You will not need any license and can configure and do the testing at free of cost (only pay for the underlying EC2 instance); once you are comfortable with the concept you can then choose to implement any of the 3rd party vendor solutions of your choice. 

![tgw-frr](https://github.com/aws-samples/aws-transit-gateway-connect-with-amazon-linux-and-frrouting/assets/168686031/2002fdaa-c457-4675-b71d-682d01b6b25a)

### Prerequisites:

* [Configure Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-getting-started.html)
* Configure Appliance VPC
* Configure Spoke VPCs
* [Configure VPC attachments to TGW](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-vpc-attachments.html#create-vpc-attachment)

### Install Amazon Linux instance in the Appliance VPC:

This testing was performed using below AMI:

AMI ID: ami-0895022f3dac85884

AMI name: amzn2-ami-kernel-5.10-hvm-2.0.20240223.0-x86_64-gp2

### Configure TGW Connect attachment:

A Connect attachment uses an existing VPC or Direct Connect attachment as the underlying transport mechanism.

[Configure TGW Connect attachments and TGW Connect peers](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-connect.html)

Verify:

![TGW1](https://github.com/aws-samples/aws-transit-gateway-connect-with-amazon-linux-and-frrouting/assets/168686031/60a63922-7a26-48f2-a649-bf9a76629693)
![TGW2](https://github.com/aws-samples/aws-transit-gateway-connect-with-amazon-linux-and-frrouting/assets/168686031/35d37d02-91c4-4ca6-854c-3af1185b655a)

### Setup GRE tunnel interfaces on Linux instance:

Configure:

```
sudo ip link set dev eth0 mtu 8500
! sudo ip tunnel add gre1 mode gre remote $TgwAddress local $PeerAddress ttl 255
! These are Underlay IPs
sudo ip tunnel add gre1 mode gre remote 192.0.2.40 local 172.31.1.152 ttl 255
! sudo ip addr add "$AttachmentInsideIp/29" dev gre1
! These are GRE Tunnel interface inside IPs
sudo ip addr add "169.254.6.1/29" dev gre1
sudo ip link set gre1 up
```

Verify:

```
[ec2-user@ip-172-31-1-152 ~]$ ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8500 qdisc mq state UP group default qlen 1000
    link/ether 02:1b:4c:bd:d9:d5 brd ff:ff:ff:ff:ff:ff
    inet 172.31.1.152/24 brd 172.31.1.255 scope global dynamic eth0
       valid_lft 2717sec preferred_lft 2717sec
    inet6 fe80::1b:4cff:febd:d9d5/64 scope link
       valid_lft forever preferred_lft forever
[ec2-user@ip-172-31-1-152 ~]$
```

```
[ec2-user@ip-172-31-1-152 ~]$ ip addr show gre1
8: gre1@NONE: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 8476 qdisc noqueue state UNKNOWN group default qlen 1000
    link/gre 172.31.1.152 peer 192.0.2.40
    inet 169.254.6.1/29 scope global gre1
       valid_lft forever preferred_lft forever
    inet6 fde2:aa1a:d66:443b:d9d8:30c9:9674:6379/125 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::5efe:ac1f:198/64 scope link
       valid_lft forever preferred_lft forever
[ec2-user@ip-172-31-1-152 ~]$
```

### Install FRR on the Linux instance: 
https://github.com/sciarrilli/free_range_routing?tab=readme-ov-file

https://github.com/sciarrilli/free_range_routing/blob/master/free_range_routing.sh

### BGP Setup:

Note: If you use eBGP, you must configure ebgp-multihop with a time-to-live (TTL) value of 2.

Configure:

```
! from the Linux prompt to get into FRR
!
vtysh
!
configure
!
ip route 192.168.0.0/24 192.168.0.1
!
router bgp 65000
 bgp router-id 169.254.6.1
 timers bgp 5 30
 neighbor ebgp peer-group
 neighbor ebgp remote-as 64512
 neighbor ebgp ebgp-multihop 64
 neighbor ebgp update-source 169.254.6.1
 neighbor 169.254.6.2 peer-group ebgp
 neighbor 169.254.6.3 peer-group ebgp
 !
 address-family ipv4 unicast
  network 192.168.0.0/24
  neighbor ebgp soft-reconfiguration inbound
 exit-address-family
!
```

Verify configurations:

```
[root@ip-172-31-1-152 ec2-user]# vtysh

Hello, this is FRRouting (version 7.2).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

ip-172-31-1-152.us-west-2.compute.internal# show running-config
Building configuration...

Current configuration:
!
frr version 7.2
frr defaults traditional
hostname ip-172-31-1-152.us-west-2.compute.internal
no ipv6 forwarding
!
ip route 192.168.0.0/24 192.168.0.1
!
router bgp 65000
 bgp router-id 169.254.6.1
 timers bgp 5 30
 neighbor ebgp peer-group
 neighbor ebgp remote-as 64512
 neighbor ebgp ebgp-multihop 2
 neighbor ebgp update-source 169.254.6.1
 neighbor 169.254.6.2 peer-group ebgp
 neighbor 169.254.6.3 peer-group ebgp
 !
 address-family ipv4 unicast
  network 192.168.0.0/24
  neighbor ebgp soft-reconfiguration inbound
 exit-address-family
!
line vty
!
end
ip-172-31-1-152.us-west-2.compute.internal#
```

Verify BGP neighborship status:

```
ip-172-31-1-152.us-west-2.compute.internal# sh ip bgp summary

IPv4 Unicast Summary:
BGP router identifier 169.254.6.1, local AS number 65000 vrf-id 0
BGP table version 127
RIB entries 7, using 1288 bytes of memory
Peers 2, using 41 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
169.254.6.2     4      64512  549550  549611        0    0    0 03w6d03h            3
169.254.6.3     4      64512  549534  549598        0    0    0 6d20h15m            3

Total number of neighbors 2
ip-172-31-1-152.us-west-2.compute.internal#
```

Verify BGP advertised and Received routes:

```
ip-172-31-1-152.us-west-2.compute.internal# sh ip bgp neighbors 169.254.6.2 advertised-routes
BGP table version is 127, local router ID is 169.254.6.1, vrf id 0
Default local pref 100, local AS 65000
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.0.0.0/16      169.254.6.2                            0 64512 i
*> 10.1.0.0/16      169.254.6.2                            0 64512 i
*> 172.31.0.0/16    169.254.6.2                            0 64512 i
*> 192.168.0.0/24   0.0.0.0                  0         32768 i

Total number of prefixes 4

ip-172-31-1-152.us-west-2.compute.internal# sh ip bgp neighbors 169.254.6.2 received-routes
BGP table version is 0, local router ID is 169.254.6.1, vrf id 0
Default local pref 100, local AS 65000
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.0.0.0/16      169.254.6.2            100             0 64512 i
*> 10.1.0.0/16      169.254.6.2            100             0 64512 i
*> 172.31.0.0/16    169.254.6.2            100             0 64512 i

Total number of prefixes 3
ip-172-31-1-152.us-west-2.compute.internal#
```

Verify that TGW has learned the routes from the virtual appliance:

![tgw3](https://github.com/aws-samples/aws-transit-gateway-connect-with-amazon-linux-and-frrouting/assets/168686031/c26f0b81-176d-48c0-bc89-ea0c557712de)

### Note:

* While code/config samples in this repository has been tested and believe it works well, as always, be sure to test it in your environment before using it in production!
* IPv6 considerations:
  * IPv6 BGP peering is not supported by TGW Connect as of today; only IPv4-based BGP peering is supported. IPv6 prefixes are exchanged over IPv4 BGP peering using MP-BGP
  * FRR supports MP-BGP
* In this example single instance in single AZ is shown for simplicity in production environment use instances across multiple AZs for redundancy

### Also see: 

https://aws.amazon.com/blogs/networking-and-content-delivery/integrate-sd-wan-devices-with-aws-transit-gateway-and-aws-direct-connect/

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

