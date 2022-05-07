---
title: "EVE-NG lab with Cisco IOS-XR, Juniper vMX and Huawei VRP(NE40E)"
date: 2022-05-04T23:51:19-07:00
draft: false
tags: ["python", "EVE-NG"]
categories: ["Network"]
---

>This series of blog will be related to network automation, my plan is to get familiar with [Nornir](https://github.com/nornir-automation/nornir), but before we can dive into the good stuff, let's build a lab that I can run automation on. 

## 1 Why using 3 different platform for the lab?

It's very common (at least for ISPs) to use network equipment from more than 1 vendor. From my specific experience, I've spent most time configuring and troubleshooting on Cisco IOS-XR and Huawei VRP routers for daily job. And invested quite a lot of time on Cisco IOS for my CCIE-RS exam. But didn't have a chance to work on Junos as much. Just like Chris mentioned in his blog [post, ](https://www.networkfuntimes.com/new-series-a-guide-to-junos-for-ios-engineers/) I kind of hate to use Junos, guess it just didn't click for me yet. 

Automating network equipment of multiple vendors is a painful process, especially when it comes to the ones that doesn't have too much global presence, or the ones that are banned (\*cough\* _Huawei_ \*cough\*). Most of modern automation tools have limited support for Huawei. Let's see if I can have [Nornir](https://github.com/nornir-automation/nornir) to work with Huawei VRP routers and get some automation action going.

## 2 Lab details

I prefer [EVE-NG](https://www.eve-ng.net/) over [GNS3](https://www.gns3.com/), since I used to lab a lot on it when I was preparing for CCIE exam, and I will be using community version (version v2.0.3-112) for this lab.

### 2.1 Import router images and get it running
First thing first, we need to import router images to EVE-NG. It's a bit of a hassle to install these images, but definitely not rocket science.

Follow official guides for [Juniper vMX](https://www.eve-ng.net/index.php/documentation/howtos/howto-add-juniper-vmx-16-x-17-x/) & [Cisco IOS-XRv](https://www.eve-ng.net/index.php/documentation/howtos/howto-add-cisco-xrv/) and this post for [Huawei VRP](https://forum.huawei.com/enterprise/en/run-ne40-ce12800-in-eve-ng/thread/672651-861).

{{< admonition type=tip title="Tip" open=true >}}
For Junos: log in to VCP as root, type "cli" to enter Junos CLI
{{< /admonition >}}

<br>
Off to a good start, all 3 images are running, let's see the version info for each router:<br><br>


Juniper Junos: 20.4R3.8

    root@vMX-R1> show version 
    Hostname: vMX-R1
    Model: vmx
    Junos: 20.4R3.8
    JUNOS OS Kernel 64-bit  [20210618.f43645e_builder_stable_11-204ab]
    JUNOS OS libs [20210618.f43645e_builder_stable_11-204ab]
    ...


Huawei VRP: 8.180

    <Huawei-R2>display ver
    Huawei Versatile Routing Platform Software
    VRP (R) software, Version 8.180 (NE40E V800R011C00SPC607B607)
    Copyright (C) 2012-2018 Huawei Technologies Co., Ltd.
    HUAWEI NE40E uptime is 0 day, 14 hours, 3 minutes 
    SVRP Platform Version 1.0

Cisco IOS-XR: 6.1.3

    RP/0/0/CPU0:IOS-XR-R3#show version
    Thu May  5 08:09:51.127 UTC
    
    Cisco IOS XR Software, Version 6.1.3[Default]
    Copyright (c) 2017 by Cisco Systems, Inc.
    
    ROM: GRUB, Version 1.99(0), DEV RELEASE
    ...
<br>

### 2.2 Build a lab in EVE-NG

It's time to build a simple lab around them to see if they work as intended. 
Interface IP addresses between Router X,Y is set to 192.168.XY.X/24 format. For connection between R1 and R2, it will be 192.168.12.1/24 for R1 and 192.168.12.2/24 for R2. And all 3 routers have lo0 interface with X.X.X.X/32 address assigned. <br>

{{< admonition type=tip title="Juniper vMX tip" open=true >}}
You need to add both VCP and VFP, connect them together through em1 interface.
{{< /admonition >}}

{{< figure src="/images/topo.png" title="EVE-NG Topology" >}}<br>

### 2.3 Change hostname & Configure interfaces<br>

Let's configure interface IP addresses and do bunch of pings to make sure they are connected.

{{< tab-widget " " >}}
Juniper
$$$
```
set system root-authentication plain-text-password
set groups router system host-name vMX-R1
set apply-groups router 
set interfaces ge-0/0/0 unit 0 family inet address 192.168.12.1/24
set interfaces ge-0/0/1 unit 0 family inet address 192.168.13.1/24
set interfaces lo0 unit 0 family inet address 1.1.1.1/32
commit
```
---
Huawei
$$$
```
<HUAWEI>system-view 
[~HUAWEI]sysname Huawei-R2
[~Huawei-R2]interface ether1/0/0
[~Huawei-R2-Ethernet1/0/0]ip address 192.168.12.2 255.255.255.0
[~Huawei-R2]interface ether1/0/1
[~Huawei-R2-Ethernet1/0/0]ip address 192.168.23.2 255.255.255.0
[~Huawei-R2]interface LoopBack0
[~Huawei-R2-LoopBack0]ip address 2.2.2.2 255.255.255.255
[~Huawei-R2-LoopBack0]commit
```
---
Cisco
$$$
```
RP/0/0/CPU0:ios#conf t
RP/0/0/CPU0:ios(config)#hostname IOS-XR-R3
RP/0/0/CPU0:IOS-XR-R3(config)#int g0/0/0/0 
RP/0/0/CPU0:IOS-XR-R3(config-if)#ipv4 address 192.168.13.3 255.255.255.0 
RP/0/0/CPU0:IOS-XR-R3(config-if)#no shut
RP/0/0/CPU0:IOS-XR-R3(config-if)#int g0/0/0/1
RP/0/0/CPU0:IOS-XR-R3(config-if)#ipv4 address 192.168.23.3 255.255.255.0
RP/0/0/CPU0:IOS-XR-R3(config-if)#no shut
RP/0/0/CPU0:IOS-XR-R3(config-if)#int lo0
RP/0/0/CPU0:IOS-XR-R3(config-if)#ipv4 address 3.3.3.3 255.255.255.255
RP/0/0/CPU0:IOS-XR-R3(config-if)#commit
```

{{< /tab-widget >}}
<br>

[comment]: # (tab-widget was modified from https://yairgadelov.me/simple-tab-widget-for-hugo-blog/ )


### 2.4 iBGP configuration (Full-mesh)

Setting up BGP for Cisco and Huawei was very straight-forward.
Junos is a little tricky, you have to add AS number under "routing-options" before you can configure iBGP commands.

{{< tab-widget " " >}}
Juniper 
$$$
```
root@vMX-R1> show configuration 
routing-options {
    autonomous-system 123;
}
protocols {
    bgp {
        group ibgp-peers {
            type internal;
            neighbor 192.168.12.2;
            neighbor 192.168.13.3;
        }
    }
}
```
---
Huawei
$$$
```
<Huawei-R2>dis current-configuration configuration bgp
bgp 123
 peer 192.168.12.1 as-number 123
 peer 192.168.23.3 as-number 123
 #
 ipv4-family unicast
  undo synchronization
  peer 192.168.12.1 enable
  peer 192.168.23.3 enable
```
---
Cisco 
$$$
```
RP/0/0/CPU0:IOS-XR-R3#sh run router bgp
router bgp 123
 address-family ipv4 unicast
 !
 neighbor 192.168.13.1
  remote-as 123
  address-family ipv4 unicast
  !
 !
 neighbor 192.168.23.2
  remote-as 123
  address-family ipv4 unicast
  !
 !
!
```
{{< /tab-widget >}}
<br>

### 2.5 Advertise loopback address to iBGP peers

Advertising lo0 address is very simple for both Cisco and Huawei, one network command will do the job. However, with Junos you have to configure a policy and apply it to the BGP config.

{{< tab-widget " " >}}
Juniper 
$$$
```
set policy-options policy-statement advertise.lo0 term 1 from route-filter 1.1.1.1/32 exact 
set policy-options policy-statement advertise.lo0 term 1 then accept 
set protocols bgp group ibgp-peers export advertise.lo0 
```
---
Huawei
$$$
```
[~Huawei-R2]bgp 123
[~Huawei-R2-bgp]network 2.2.2.2 32
```
---
Cisco 
$$$
```
router bgp 123 address-family ipv4 unicast network 3.3.3.3 255.255.255.255
```
{{< /tab-widget >}}
<br>
Let's make sure BGP peering is correctly established and they are receving prefixes from each other.

{{< tab-widget " " >}}
Juniper
$$$
{{< figure src="/images/SNAG-0010.png" title="" >}}
---
Huawei
$$$
{{< figure src="/images/SNAG-0011.png" title="" >}}
---
Cisco
$$$
{{< figure src="/images/SNAG-0012.png" title="" >}}
{{< /tab-widget >}}
<br>

## 3 Complete configs

Looks great so far, the following is full configuration for this lab.



{{< tab-widget " " >}}
Juniper 
$$$
```
root@vMX-R1> show configuration
groups {
    router {
        system {
            host-name vMX-R1;
        }
    }
}
apply-groups router;
system {
    root-authentication {
        encrypted-password "$6$PJBo9jFx$C2/jRVRHt/6wIJeRzJ4BJH6i.iDpoM5ZCkKkEMyUsnn9nPSDQcccK4mmvd8IGt.21Gp9x3bg3fZpmp6L/HxBI."; ## SECRET-DATA
    }
}
interfaces {
    ge-0/0/0 {
        unit 0 {
            family inet {
                address 192.168.12.1/24;
            }
        }
    }
    ge-0/0/1 {
        unit 0 {
            family inet {
                address 192.168.13.1/24;
            }
        }                               
    }
    lo0 {
        unit 0 {
            family inet {
                address 1.1.1.1/32;
            }                           
        }
    }
}
policy-options {
    policy-statement advertise.lo0 {
        term 1 {
            from {
                route-filter 1.1.1.1/32 exact;
            }
            then accept;
        }
    }
}
routing-options {
    autonomous-system 123;
}
protocols {
    router-advertisement {
        interface fxp0.0;
    }
    bgp {
        group ibgp-peers {
            type internal;              
            export advertise.lo0;
            neighbor 192.168.12.2;
            neighbor 192.168.13.3;
        }
    }
}
```
---
Huawei
$$$
```
<Huawei-R2>dis current-configuration 
sysname Huawei-R2
#
interface Ethernet1/0/0
 undo shutdown
 ip address 192.168.12.2 255.255.255.0
#
interface Ethernet1/0/1
 undo shutdown
 ip address 192.168.23.2 255.255.255.0
#
interface LoopBack0
 ip address 2.2.2.2 255.255.255.255
#
bgp 123
 peer 192.168.12.1 as-number 123
 peer 192.168.23.3 as-number 123
 #
 ipv4-family unicast
  undo synchronization
  network 2.2.2.2 255.255.255.255
  peer 192.168.12.1 enable
  peer 192.168.23.3 enable
```
---
Cisco 
$$$
```
RP/0/0/CPU0:IOS-XR-R3#sh run
hostname IOS-XR-R3
interface Loopback0
 ipv4 address 3.3.3.3 255.255.255.255
!
interface GigabitEthernet0/0/0/0
 ipv4 address 192.168.13.3 255.255.255.0
!
interface GigabitEthernet0/0/0/1
 ipv4 address 192.168.23.3 255.255.255.0
!
router bgp 123
 address-family ipv4 unicast
  network 3.3.3.3/32
 !
 neighbor 192.168.13.1
  remote-as 123
  address-family ipv4 unicast
  !
 !
 neighbor 192.168.23.2
  remote-as 123
  address-family ipv4 unicast
  !
 !
!
```
{{< /tab-widget >}}
<br>

## 4 What's next & TIL

{{< admonition type=success title="Success" open=true >}}
Initial part of the lab is finished.
{{< /admonition >}}

### 4.1 Next step
I'll add bits and pieces to make the lab suitable for automation, or completely overhaul the entire topology, just for the sake of messing around.  :joy: 

### 4.2 TIL
TIL section is dedicated to the things that I've learned while writing blog posts, they can be either very trivial commands, or more complex concepts.

{{< admonition type=Note title="TIL : Today I Learned" open=true >}}
To show / compare uncommitted changes, simply run these commands:<br>
`show | compare (Junos)`<br>
`display configuration candidate changes (Huawei VRP)`<br>
`show commit changes diff (Cisco IOS-XR)`<br>
{{< /admonition >}}
