# Connecting Fortigate to Cisco using GRE-over-IPsec 

This is the configuration i used to connect my Fortigate 201E (running OS 6.2.5) to a cisco router using gre over ipsec

Connection was like below

![Toplogy](https://github.com/Amiri83/fortigate/blob/main/firewall.png)


## Enabling overlapping subnets

By default, each FortiGate unit network interface must be on a separate network. The configuration described in this chapter assigns an IPsec tunnel end point and the external interface to the same network. Enable subnet overlap as follows:

```sh
config system settings
set allow-subnet-overlap enable
end
```

## Configuring the IPsec VPN

fortigate side interface name which has GW address toward cisco network , named as to_cisco

```sh
config vpn ipsec phase1-interface
    edit "ipsec_cisco"
        set interface "Shaparak"
        set keylife 28800
        set peertype any
        set net-device disable
        set proposal aes128-sha256 aes256-sha256 aes128-sha1 aes256-sha1
        set dhgrp 5
        set nattraversal disable
        set remote-gw 172.40.20.10
        set psksecret <Your IPsec Secret>
    next
end

config vpn ipsec phase2-interface
    edit "ipsec_cisco"
        set phase1name "ipsec_cisco"
        set proposal aes128-sha1 aes128-md5
        set pfs disable
        set protocol 47
        set src-addr-type ip
        set dst-addr-type ip
        set keylifeseconds 30000
        set src-start-ip 172.40.20.20
        set dst-start-ip 172.40.20.10
    next
end


```

## Adding IPsec tunnel end addresses

The Cisco configuration requires an address for its end of the IPsec tunnel

```sh
config system interface
    edit "ipsec_cisco"
        set vdom "root"
        set ip 172.40.20.20 255.255.255.255
        set allowaccess ping https http
        set type tunnel
        set remote-ip 172.40.20.10 255.255.255.255
        set interface "to_cisco"
    next

```

## Configuring the GRE tunnel

The GRE tunnel runs between the IPsec public interface on the FortiGate unit and the Cisco router. 

```sh

config system gre-tunnel
    edit "gre_cisco"
        set interface "ipsec_cisco"
        set remote-gw 172.40.20.10
        set local-gw 172.40.20.20
    next
end
```
## Adding GRE tunnel end addresses

You will also need to add tunnel end addresses. The Cisco router configuration requires an address for its end of the GRE tunnel.

```sh
config system interface
    edit "gre_cisco"
        set vdom "root"
        set ip 172.20.20.14 255.255.255.255
        set allowaccess ping https http
        set type tunnel
        set tcp-mss 1360
        set remote-ip 172.20.20.13 255.255.255.252
        set interface "ipsec_cisco"
    next
end
```

## Configuring security policies

We need policies to allow traffic to pass in both directions between the GRE virtual interface and the IPsec interface.


```sh
 config firewall policy
    edit 1
        set name "gre_cisco > Ipsec_cisco"
        set srcintf "gre_cisco"
        set dstintf "ipsec_cisco"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set utm-status enable
    next
    edit 2
        set name "Ipsec_cisco > cisco"
        set srcintf "ipsec_cisco"
        set dstintf "gre_cisco"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set utm-status enable
    next

end

```

## Configure OSPF

you should know router ID side , are number , network to adverise , MTU and cost to do the configuration

```sh
config router ospf
    set router-id  <your router id got from cisco sude>
	config area
        edit 0.0.0.2 <id form cisco side>
        next
    end
        edit "GRE_cisco"
            set interface "gre_cisco"
            set cost xxxx # ask cust from cisco side
            set dead-interval 40
            set hello-interval 10
            set mtu  xxx # ask Mtu from cisco side
            set network-type point-to-point
        next
    end
config network
        edit 1
            set prefix network and mask advertised from cisco side
            set area 0.0.0.2
        next

    end 
	
```


## Check OSPF Status

```sh

FG-1 (root) # get router info ospf neighbor 

OSPF process 0, VRF 0:
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2     1   Full/ -         00:00:37    172.20.20.13  gre_cisco

 	
	

```


