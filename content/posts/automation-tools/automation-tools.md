---
title: "Automation tools: Paramiko, Netmiko, NAPALM, Ansible, Nornir or ...?"
date: 2022-05-15T21:45:12-07:00
draft: false
author: "Kevin"
authorLink: "https://kevinjin.com"
description: "Automation tools: Paramiko, Netmiko, NAPALM, Ansible, Nornir or ...?"

tags: ["EVE-NG", "Automation", "Nornir"]
categories: ["Network"]

lightgallery: true

toc:
  auto: false
math:
  enable: true
---

>[Last post]({{< ref "posts/eve-ng/eve-ng.md" >}}) was all about setting up a simple EVE-NG lab with 3 different vendors (Cisco, Juniper and Huawei). In this post, let's get our hands dirty and get some network automation action going. 

## 1 What are the differences of Paramiko, Netmiko, NAPALM, Ansible and Nornir?

Before we start network automation lab, I feel obligated to discuss a bit about how different automation tools work and what their differences are.

I'm not an expert in any of the following tools by any stretch, but I will try my best to describe these in layman's terms based on my experience.

- [Paramiko](https://www.paramiko.org/): Just a handy Python SSH library, used for ssh-ing (obviously) to devices.

- [Netmiko](https://github.com/ktbyers/netmiko): Another Python SSH library which is based on Paramiko, but geared more towards network devices. Unlike Paramiko, it supports Telnet. When combined with Python scripts, this tool is sufficient to kickstart your network automation. Very easy to get started and have results of your commands printed and screen scrape the things you need with regex, or even better if you utilize TextFSM & NTC templates for parsing results. Little caveat is that you must handle multi-threading yourself if the intention is to use Netmiko for large scale network.

- [NAPALM](https://napalm.readthedocs.io/en/latest/): A Python library/framework that supports multiple vendors using API. It's an abstraction framework, and NAPALM's underlying network drivers will enable it to return the same output for your request (Eg: get_interfaces, get_facts) regardless of which type of device you're working on. If your devices are not fully supported, you have the option to write your own Python library (which I tried, and I don't recommend, will discuss in detail [later in this post](#23-napalm)). NAPALM framework can be used in conjunction with Ansible, Salt and Nornir.

- [Ansible](https://github.com/ansible/ansible): One of the most famous automation tools. The thing with Ansible is that you have to write Ansible playbooks in YAML, which is kind of a double-edged sword; it doesn't require you to know how to code in Python, but working with only YAML takes away the flexibility of your automation tasks. Ansible IMO is better suited for overall IT environment rather than network devices.

- [Nornir](https://nornir.readthedocs.io/en/latest/): A Python framework made specifically for network devices, developed and maintained by the same guys who did NAPALM and Netmiko. It's 100% Python, framework is written in Python, and you write Python code in order to use Nornir. NAPALM and Netmiko libraries can act as getters and connection driver for Nornir. Also, Nornir is multi-threaded and it's absurdly fast compared to Ansible.

There are some other automation tools that I didn't mention; like Scrapli, Puppet, Chef, Salt, pyATS, etc.

## 2 Testing out automation tools (Netmiko, NAPALM, Nornir)

> In this section, I'm going to show basic usage of some of the tools covered above.

### 2.1 SSH Connectivity to devices

#### First, enable SSH on each device: 

Juniper Junos: 20.4R3.8

    set system login user juniper class super-user authentication plain-text-password
    set system services ssh root-login allow
    set system services ssh protocol-version v2
    set system services ssh connection-limit 10


Huawei VRP: 8.180

    aaa
        local-user huawei password irreversible-cipher $1c$jF...
        local-user huawei service-type telnet ssh
        local-user huawei state block fail-times 3 interval 5
    user-interface vty 0 4
        authentication-mode aaa
        user privilege level 3
        idle-timeout 0 0
    stelnet server enable
    ssh user huawei
    ssh user huawei authentication-type all
    ssh user huawei service-type all
    ssh authorization-type default password



Cisco IOS-XR: 6.1.3

    crypto key generate rsa
    ssh server v2
    line console transport input all
<br>
Above commands will give us full SSH connectivity from outside VM I'm using. Both my EVE-NG VM and my Linux VM (Kali Linux) are configured as NAT, and I will run my automation setup on Kali Linux. 
Since my NAT is using 192.168.13.0/24 subnet, I've modified interface addresses between routers to 10.0.XY.X, instead of 192.168.XY.X pool. 
All 3 devices are connected to Management(Cloud0) node, ensuring their connectivity to my Kali Linux.

<br>

{{< figure src="/images/automation-tools/topo.png" title="" >}}<br>

### 2.2 Netmiko

Following is a very simple example of how to use Netmiko ConnectHandler to retrieve show command output from routers. In this example, send_command() method will return interface description and IP addresses of each device. If you want to convert the result to structured data, you might want to use [TextFSM & NTC templates](https://pynet.twb-tech.com/blog/netmiko-and-textfsm.html). 

```python
#netmiko_test.py
from netmiko import ConnectHandler

juniper_vMX = {
    'device_type': 'juniper',
    'ip': '192.168.13.11',
    'username': 'juniper',
    'password': 'Juniper'
}
net_connect = ConnectHandler(**juniper_vMX)
output = net_connect.send_command("show interface terse")
print("Juniper IP:\n\n"+output+"\n---------------------------------------\n")

huawei_vrp = {
    'device_type': 'huawei',
    'ip': '192.168.13.22',
    'username': 'huawei',
    'password': 'Admin@1231'
}
net_connect = ConnectHandler(**huawei_vrp)
output = net_connect.send_command("display ip int br")
print("Huawei IP:\n\n"+output+"\n---------------------------------------\n")

ios_xr = {
    'device_type': 'cisco_xr',
    'ip': '192.168.13.33',
    'username': 'cisco',
    'password': 'cisco'
}
net_connect = ConnectHandler(**ios_xr)
output = net_connect.send_command("show ip int br")
print("Cisco IP:\n\n"+output+"\n---------------------------------------\n")
```

### 2.3 NAPALM

My plan was to retrieve BGP neighbor info with NAPALM. NAPALM is very straightforward tool _**IF**_ all of your device types are suppored. However that's not the case in our scenario. Huawei routers are not officially supported by NAPALM, there is a community NAPALM library ([NAPALM-Huawei-VRP](https://github.com/napalm-automation-community/napalm-huawei-vrp)), but that only has very limited functions built-in. My only option was to write my own functions based on the community version, while I was researching, found out I wasn't the only one who has attempted this. [Michael](https://codingnetworks.blog/napalm-network-automation-python-collect-data-from-multiple-vendors/) did some heavy lifting but his code was still missing some parts. Expand the following code snippet for BGP neighbor implementation for Huawei VRP platform. My [github repo](https://github.com/skndmx/napalm-huawei-vrp/blob/master/napalm_huawei_vrp/huawei_vrp.py "Folow me on github!") has complete code so that you don't have to reinvent the wheel.

{{< highlight python "linenos=table,linenostart=848" >}}
    @staticmethod
    def bgp_time_conversion(bgp_uptime):
        """
        Convert string time to seconds.
        Examples
        00:14:23
        00:13:40
        00:00:21
        00:00:13
        00:00:49
        1d11h
        1d17h
        1w0d
        8w5d
        1y28w
        never
        """
        bgp_uptime = bgp_uptime.strip()
        uptime_letters = set(['w', 'h', 'd', 'm'])

        if 'never' in bgp_uptime:
            return -1
        elif ':' in bgp_uptime:
            times = bgp_uptime.split(":")
            times = [int(x) for x in times]
            hours, minutes, seconds = times
            return (hours * 3600) + (minutes * 60) + seconds
        # Check if any letters 'w', 'h', 'd' are in the time string
        elif uptime_letters & set(bgp_uptime):
            form0 = r'(\d+)h(\d+)m'  # 03h21m
            form1 = r'(\d+)d(\d+)h'  # 1d17h
            form2 = r'(\d+)w(\d+)d'  # 8w5d
            form3 = r'(\d+)y(\d+)w'  # 1y28w
            match = re.search(form0, bgp_uptime)
            if match:
                hours = int(match.group(1))
                minutes = int(match.group(2))
                return (hours * 3600) + (minutes * 60)
            match = re.search(form1, bgp_uptime)
            if match:
                days = int(match.group(1))
                hours = int(match.group(2))
                return (days * DAY_SECONDS) + (hours * 3600)
            match = re.search(form2, bgp_uptime)
            if match:
                weeks = int(match.group(1))
                days = int(match.group(2))
                return (weeks * WEEK_SECONDS) + (days * DAY_SECONDS)
            match = re.search(form3, bgp_uptime)
            if match:
                years = int(match.group(1))
                weeks = int(match.group(2))
                return (years * YEAR_SECONDS) + (weeks * WEEK_SECONDS)
        raise ValueError("Unexpected value for BGP uptime string: {}".format(bgp_uptime))

    ## custom bgp config for VRP, reference:https://codingnetworks.blog/napalm-network-automation-python-collect-data-from-multiple-vendors/
    def get_bgp_neighbors(self):
        """
        Returns a dictionary of dictionaries. The keys for the first dictionary will be the vrf
        (global if no vrf). The inner dictionary will contain the following data for each vrf:

          * router_id
          * peers - another dictionary of dictionaries. Outer keys are the IPs of the neighbors. \
            The inner keys are:
             * local_as (int)
             * remote_as (int)
             * remote_id - peer router id
             * is_up (True/False)
             * is_enabled (True/False)
             * description (string)
             * uptime (int in seconds)
             * address_family (dictionary) - A dictionary of address families available for the \
               neighbor. So far it can be 'ipv4' or 'ipv6'
                * received_prefixes (int)
                * accepted_prefixes (int)
                * sent_prefixes (int)

            Note, if is_up is False and uptime has a positive value then this indicates the
            uptime of the last active BGP session.

            Example::

                {
                  "global": {
                    "router_id": "10.0.1.1",
                    "peers": {
                      "10.0.0.2": {
                        "local_as": 65000,
                        "remote_as": 65000,
                        "remote_id": "10.0.1.2",
                        "is_up": True,
                        "is_enabled": True,
                        "description": "internal-2",
                        "uptime": 4838400,
                        "address_family": {
                          "ipv4": {
                            "sent_prefixes": 637213,
                            "accepted_prefixes": 3142,
                            "received_prefixes": 3142
                          },
                          "ipv6": {
                            "sent_prefixes": 36714,
                            "accepted_prefixes": 148,
                            "received_prefixes": 148
                          }
                        }
                      }
                    }
                  }
                }
        """
        afi_supported = {
        "Ipv6 Unicast" : "ipv6 unicast",
        "Ipv4 Unicast" : "ipv4 unicast",
        "Vpnv4 All" : "vpnv4 unicast",
        "Vpnv6 All" : "vpnv6 unicast"
        }
        bgp_neighbors = {}
        command_bgp = "display bgp all summary"
        output = self.device.send_command(command_bgp)
    
        if output == "":
            return bgp_neighbors

        ASN_REGEX = r"[\d\.]+"
        IP_ADDR_REGEX = r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"
        IPV4_ADDR_REGEX = IP_ADDR_REGEX
        IPV6_ADDR_REGEX_1 = r"::"
        IPV6_ADDR_REGEX_2 = r"[0-9a-fA-F:]{1,39}::[0-9a-fA-F:]{1,39}"
        IPV6_ADDR_REGEX_3 = r"[0-9a-fA-F]{1,3}:[0-9a-fA-F]{1,3}:[0-9a-fA-F]{1,3}:[0-9a-fA-F]{1,3}:" \
                             r"[0-9a-fA-F]{1,3}:[0-9a-fA-F]{1,3}:[0-9a-fA-F]{1,3}:[0-9a-fA-F]{1,3}"
        # Should validate IPv6 address using an IP address library after matching with this regex
        IPV6_ADDR_REGEX = r"(?:{}|{}|{})".format(IPV6_ADDR_REGEX_1, IPV6_ADDR_REGEX_2, IPV6_ADDR_REGEX_3)
        IPV4_OR_IPV6_REGEX = r"(?:{}|{})".format(IPV4_ADDR_REGEX, IPV6_ADDR_REGEX)
        #Regular Expressions
        re_separator = r"\n\s*(?=Address Family:\s*\w+)"
        re_vpn_instance_separator = r"\n\s*(?=VPN-Instance\s+[-_a-zA-Z0-9]+)"
    
        re_address_family = r"Address Family:(?P<address_family>\w+\s\w+)"
        re_global_router_id = r"BGP local router ID :\s+(?P<glob_router_id>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})"
        re_global_local_as = r"Local AS number :\s+(?P<local_as>{})".format(ASN_REGEX)
        re_vrf_router_id = r"VPN-Instance\s+(?P<vrf>[-_a-zA-Z0-9]+), [rR]outer ID\s+" \
                              r"(?P<vrf_router_id>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})"
        re_peers = r"(?P<peer_ip>({})|({}))\s+" \
                   r"(?P<as>{})\s+\d+\s+\d+\s+\d+\s+(?P<updown_time>[a-zA-Z0-9:]+)\s+" \
                   r"(?P<state>[a-zA-Z0-9\(\)]+)\s+(?P<received_prefixes>\d+)\s+(?P<adv_prefixes>\d+)".format(
                            IPV4_ADDR_REGEX, IPV6_ADDR_REGEX, ASN_REGEX)
    
        re_remote_rid = r"Remote router ID\s+(?P<remote_rid>{})".format(IPV4_ADDR_REGEX)
        re_peer_description = r"Peer's description:\s+\"(?P<peer_description>.*)\""
        re_accepted_routes = r"Received active routes total:\s+(?P<accepted_routes>.*)"
    
        #Separation of AFIs
        afi_list = re.split(re_separator, output, flags=re.M)
        #return afi_list
    
        bgp_global_router_id = ""
        bgp_global_local_as = ""
        for afi in afi_list:
        
            match_afi = re.search(re_global_router_id,afi, flags=re.M)
            match_local_as = re.search(re_global_local_as,afi, flags=re.M)
            if match_afi is not None:
                bgp_global_router_id = match_afi.group('glob_router_id')
                bgp_global_local_as = match_local_as.group('local_as')
             
            match_afi = re.search(re_address_family, afi, flags=re.M)
    
            if match_afi is not None and any((s in match_afi.group('address_family') for s in ['Ipv4 Unicast','Ipv6 Unicast'])):
            
                bgp_neighbors.update({"global": {"router_id": bgp_global_router_id, "peers" : {}}})
    
                for peer in afi.splitlines():
                
                    match_peer = re.search(re_peers, peer, flags=re.M)
                     
                    if match_peer:
                        peer_bgp_command = ""
                        if "Ipv6" in match_afi.group('address_family'):
                            peer_bgp_command = "display bgp ipv6 peer {} verbose".format(match_peer.group('peer_ip'))
                        else:
                            peer_bgp_command = "display bgp peer {} verbose".format(match_peer.group('peer_ip'))
                         
                        peer_detail = self.device.send_command(peer_bgp_command)
    
                        match_remote_rid = re.search(re_remote_rid, peer_detail, flags=re.M)
                        match_peer_description = re.search(re_peer_description, peer_detail, flags=re.M)
                        match_accepted_routes = re.search(re_accepted_routes,peer_detail, flags=re.M )                       

                        bgp_neighbors["global"]["peers"].update( { 
                        match_peer.group('peer_ip'): { 
                        "local_as": int(bgp_global_local_as), 
                        "remote_as": int(match_peer.group('as')), 
                        "remote_id": "" if match_remote_rid is None else match_remote_rid.group('remote_rid'), 
                        "is_up": True if "Established" in match_peer.group('state') else False, 
                        "is_enabled": False if "Admin" in match_peer.group('state') else True, 
                        "description": "" if match_peer_description is None else match_peer_description.group('peer_description'), 
                        "uptime": int(self.bgp_time_conversion(match_peer.group('updown_time'))), 
                        "address_family": { 
                            afi_supported[match_afi.group('address_family')]: { 
                                "received_prefixes": int(match_peer.group('received_prefixes')), 
                                #"accepted_prefixes": "Unknown", 
                                "accepted_prefixes": int(match_accepted_routes.group('accepted_routes')),
                                "sent_prefixes": int(match_peer.group('adv_prefixes')) 
                                           }
                                        }
                               }
                            })
             
            elif match_afi is not None and any((s in match_afi.group('address_family') for s in ['Vpnv4 All','Vpnv6 All'])):
                if bgp_neighbors['global'] is False:
                    bgp_neighbors.update({"global": {"router_id": bgp_global_router_id, "peers" : {}}})
    
                #Separation of VPNs
                vpn_instance_list = re.split(re_vpn_instance_separator, afi, flags=re.M)
                 
                for vpn_peers in vpn_instance_list:
                
                    if "VPN-Instance " not in vpn_peers:
                        for peer in vpn_peers.splitlines():
                            match_peer = re.search(re_peers, peer, flags=re.M)
                            if match_peer:
                            
                                if bgp_neighbors["global"]["peers"][match_peer.group('peer_ip')]:
                                   bgp_neighbors["global"]["peers"][match_peer.group('peer_ip')]["address_family"].update(
                                   { 
                                    afi_supported[match_afi.group('address_family')]: { 
                                        "received_prefixes": int(match_peer.group('received_prefixes')), 
                                        "accepted_prefixes": "Unknown", 
                                        "sent_prefixes": int(match_peer.group('adv_prefixes')) 
                                                   }
                                                }
                                   ) 
                                else:
                                
                                    peer_bgp_command = ""
                                    if "Ipv6" in match_afi.group('address_family'):
                                        peer_bgp_command = "display bgp ipv6 peer {} verbose".format(match_peer.group('peer_ip'))
                                    else:
                                        peer_bgp_command = "display bgp peer {} verbose".format(match_peer.group('peer_ip'))            
                                    peer_detail = self.device.send_command(peer_bgp_command)
    
                                    match_remote_rid = re.search(re_remote_rid, peer_detail, flags=re.M)
                                    match_peer_description = re.search(re_peer_description, peer_detail, flags=re.M)
                                     
                                    bgp_neighbors["global"]["peers"].update( { 
                                    match_peer.group('peer_ip'): { 
                                    "local_as": int(bgp_global_local_as), 
                                    "remote_as": int(match_peer.group('as')), 
                                    "remote_id": "" if match_remote_rid is None else match_remote_rid.group('remote_rid'), 
                                    "is_up": True if "Established" in match_peer.group('state') else False, 
                                    "is_enabled": False if "Admin" in match_peer.group('state') else True, 
                                    "description": "" if match_peer_description is None else match_peer_description.group('peer_description'), 
                                    "uptime": int(self.bgp_time_conversion(match_peer.group('updown_time'))), 
                                    "address_family": { 
                                       afi_supported[match_afi.group('address_family')]: { 
                                            "received_prefixes": int(match_peer.group('received_prefixes')), 
                                            "accepted_prefixes": "Unknown", 
                                            "sent_prefixes": int(match_peer.group('adv_prefixes')) 
                                                       }
                                                    }
                                           }
                                        })
                    else:
                        match_vrf_router_id = re.search(re_vrf_router_id, vpn_peers, flags=re.M)
    
                        if match_vrf_router_id is None:
                            msg = "No Match Found"
                            raise ValueError(msg)
    
                        peer_vpn_instance = match_vrf_router_id.group('vrf')
                        peer_router_id = match_vrf_router_id.group('vrf_router_id')
    
                        bgp_neighbors.update({peer_vpn_instance: {
                                        "router_id": peer_router_id, "peers" : {}}})
    
                        for peer in vpn_peers.splitlines():
                                     
                            match_peer = re.search(re_peers, peer, flags=re.M)
                            if match_peer:
                            
                                peer_bgp_command = ""
                                afi_vrf = ""
                                if "Ipv6" in match_afi.group('address_family'):
                                    peer_bgp_command = "display bgp ipv6 peer {} verbose".format(match_peer.group('peer_ip'))
                                    afi_vrf = "ipv6 unicast"
                                else:
                                    peer_bgp_command = "display bgp peer {} verbose".format(match_peer.group('peer_ip'))
                                    afi_vrf = "ipv4 unicast"
                                 
                                peer_detail = self.device.send_command(peer_bgp_command)
    
                                match_remote_rid = re.search(re_remote_rid, peer_detail, flags=re.M)
                                match_peer_description = re.search(re_peer_description, peer_detail, flags=re.M)
    
                                bgp_neighbors[peer_vpn_instance]["peers"].update( { 
                                match_peer.group('peer_ip'): { 
                                "local_as": int(bgp_global_local_as), 
                                "remote_as": int(match_peer.group('as')), 
                                "remote_id": "" if match_remote_rid is None else match_remote_rid.group('remote_rid'), 
                                "is_up": True if "Established" in match_peer.group('state') else False, 
                                "is_enabled": False if "Admin" in match_peer.group('state') else True, 
                                "description": "" if match_peer_description is None else match_peer_description.group('peer_description'), 
                                "uptime": int(self.bgp_time_conversion(match_peer.group('updown_time'))),         
                                "address_family": { 
                                    afi_vrf: { 
                                        "received_prefixes": int(match_peer.group('received_prefixes')), 
                                        "accepted_prefixes": "Unknown", 
                                        "sent_prefixes": int(match_peer.group('adv_prefixes')) 
                                                   }
                                                }
                                       }
                                    })
        return bgp_neighbors
{{< / highlight >}}

Now the NAPALM library is finally ready for action, let's give it try! 

```python
#napalm_test.py
import napalm

def main():
    driver_juniper = napalm.get_network_driver("junos")
    driver_vrp = napalm.get_network_driver("huawei_vrp")
    driver_ios = napalm.get_network_driver("iosxr")
    
    juniper_router = driver_juniper(
    hostname = "192.168.13.11",
    username = "juniper",
    password = "Juniper"
    )

    vrp_router = driver_vrp(
    hostname = "192.168.13.22",
    username = "huawei",
    password = "Admin@1231"
    )

    ios_router = driver_ios(
    hostname = "192.168.13.33",
    username = "cisco",
    password = "cisco"
    )
    
    print("Connecting to Juniper Router...")
    juniper_router.open()
    print("Checking Juniper Router BGP Neighbors:")  
    print(juniper_router.get_bgp_neighbors())
    juniper_router.close()
    print("Test Completed\n")

    print("Connecting to Huawei Router...")
    vrp_router.open()
    print("Checking Huawei Router BGP Neighbors:")  
    print(vrp_router.get_bgp_neighbors())
    vrp_router.close()
    print("Test Completed\n")

    print("Connecting to IOS Router...")
    ios_router.open()
    print("Checking IOS Router BGP Neighbors:")  
    print(ios_router.get_bgp_neighbors())
    ios_router.close()
    print("Test Completed\n")
 
if __name__ == "__main__":
    main()
```

Unlike what Netmiko did, NAPALM generates structured result for each router. The BGP neighbors result is returned as a dictionary of dictionaries. 

{{< tab-widget " " >}}
Juniper
$$$
```python
{
    "global": {
        "router_id": "1.1.1.1",
        "peers": {
            "10.0.12.2": {
                "local_as": 123,
                "remote_as": 123,
                "remote_id": "2.2.2.2",
                "is_up": True,
                "is_enabled": True,
                "description": "",
                "uptime": 211073,
                "address_family": {
                    "ipv4": {
                        "received_prefixes": 1,
                        "accepted_prefixes": 1,
                        "sent_prefixes": 1,
                    },
                    "ipv6": {
                        "received_prefixes": -1,
                        "accepted_prefixes": -1,
                        "sent_prefixes": -1,
                    },
                },
            },
            "10.0.13.3": {
                "local_as": 123,
                "remote_as": 123,
                "remote_id": "3.3.3.3",
                "is_up": True,
                "is_enabled": True,
                "description": "",
                "uptime": 211079,
                "address_family": {
                    "ipv4": {
                        "received_prefixes": 1,
                        "accepted_prefixes": 1,
                        "sent_prefixes": 1,
                    },
                    "ipv6": {
                        "received_prefixes": -1,
                        "accepted_prefixes": -1,
                        "sent_prefixes": -1,
                    },
                },
            },
        },
    }
}
```
-----
Huawei
$$$
```python
{
    "global": {
        "router_id": "2.2.2.2",
        "peers": {
            "10.0.12.1": {
                "local_as": 123,
                "remote_as": 123,
                "remote_id": "1.1.1.1",
                "is_up": True,
                "is_enabled": True,
                "description": "",
                "uptime": 211080,
                "address_family": {
                    "ipv4 unicast": {
                        "received_prefixes": 1,
                        "accepted_prefixes": 1,
                        "sent_prefixes": 1,
                    }
                },
            },
            "10.0.23.3": {
                "local_as": 123,
                "remote_as": 123,
                "remote_id": "3.3.3.3",
                "is_up": True,
                "is_enabled": True,
                "description": "",
                "uptime": 211080,
                "address_family": {
                    "ipv4 unicast": {
                        "received_prefixes": 1,
                        "accepted_prefixes": 1,
                        "sent_prefixes": 1,
                    }
                },
            },
        },
    }
}
```
-----
Cisco
$$$
```python
{
    "global": {
        "peers": {
            "10.0.13.1": {
                "local_as": 123,
                "remote_as": 123,
                "remote_id": "1.1.1.1",
                "description": "",
                "is_enabled": False,
                "is_up": True,
                "uptime": 211044,
                "address_family": {
                    "ipv4": {
                        "received_prefixes": 1,
                        "accepted_prefixes": 1,
                        "sent_prefixes": 1,
                    }
                },
            },
            "10.0.23.2": {
                "local_as": 123,
                "remote_as": 123,
                "remote_id": "2.2.2.2",
                "description": "",
                "is_enabled": False,
                "is_up": True,
                "uptime": 211062,
                "address_family": {
                    "ipv4": {
                        "received_prefixes": 1,
                        "accepted_prefixes": 1,
                        "sent_prefixes": 1,
                    }
                },
            },
        },
        "router_id": "3.3.3.3",
    }
}
```

{{< /tab-widget >}}
<br>

With nested dictionaries, you can easily access the elements using the [ ] syntax. This line of code returns the BGP uptime of peer 10.0.12.2 in seconds.

    print (juniper_router.get_bgp_neighbors()['global']['peers']['10.0.12.2']['uptime'])
    > 211073
    
### 2.4 Nornir

#### 2.4.1 Initializing Nornir
In this lab, we are using SimpleInventory plugin, which stores all the relevant data in three files (_hosts.yaml_, _groups.yaml_, _defaults,yaml_). 

{{< tab-widget " " >}}
hosts.yaml
$$$
```yaml
---
router1:
    hostname: 192.168.13.11
    username: juniper
    password: Juniper
    groups:
        - juniper
router2:
    hostname: 192.168.13.22
    username: huawei
    password: Admin@1231    
    groups:
        - huawei
router2`:
    hostname: 192.168.13.22
    username: huawei
    password: Admin@1231    
    groups:
        - huawei_vrpv8
router3:
    hostname: 192.168.13.33
    username: cisco
    password: cisco
    groups:
        - cisco
```
-----
groups.yaml
$$$
```yaml
---
cisco:
    platform: ios-xr

huawei:
    platform: huawei_vrp

huawei_vrpv8:
    platform: huawei_vrpv8

juniper:
    platform: junos
```
-----
defaults.yaml 
$$$
```yaml
---
username: juniper
password: Juniper
```
{{< /tab-widget >}}




We need a _config.yaml_ file to let Nornir know we have inventory files ready for Nornir. You can change multi-thread option in this file as well.
```yaml
#config.yaml
---
inventory:
    plugin: SimpleInventory
    options:
        host_file: "hosts.yaml"
        group_file: "groups.yaml"
        defaults_file: "defaults.yaml"
runner:
    plugin: threaded
    options:
        num_workers: 100
```
Now we can create Nornir object like below:
```python
from nornir import InitNornir
nr = InitNornir(config_file="config.yaml")
```
<br>

As I already mentioned, Nornir supports [third-party plugins](https://nornir.tech/nornir/plugins/) such as Netmiko, Scrapli, NAPALM, Ansible, Jinja2, Netbox, etc.

Let's see how we can use Netmiko and NAPALM within Nornir.

#### 2.4.2 Nornir with Netmiko plugin

First off, let's try Netmiko plugin. Let's make it more fun with the use of Nornir filter function, to select specific group of routers in our inventory and show BGP neighbor info on that router with Netmiko. Getting command result requires importing the plugin and running only one line of code. 

```python
#nornir_netmiko_test.py
from nornir import InitNornir
from nornir_netmiko import netmiko_send_command
from nornir_utils.plugins.functions import print_result
from nornir.core.filter import F
 
nr = InitNornir(config_file="config.yaml")
group1 = nr.filter(F(groups__contains="huawei_vrpv8"))
results = group1.run(netmiko_send_command, command_string='dis bgp peer')

print_result(results)
```
Result from above code:

{{< figure src="/images/automation-tools/nornir_netmiko.png" title="" >}}<br>

#### 2.4.3 Nornir with NAPALM plugin

Now, let's try NAPALM plugin for Nornir. This time, I'm using `~F` function to filter routers that are not _"huawei_vrpv8"_. Then run NAPALM getters to retrieve BGP neighbor info. 

```python
#nornir_napalm_test.py
from nornir import InitNornir
from nornir_napalm.plugins.tasks import napalm_get
from nornir_utils.plugins.functions import print_result
from nornir.core.filter import F
 
nr = InitNornir(config_file="config.yaml", dry_run=True)
group2 = nr.filter(~F(groups__contains="huawei_vrpv8"))
results = group2.run(task=napalm_get, getters=["bgp_neighbors"])

print_result(results)
```
Again, NAPALM plugin is capable of returning structured data for easier manipulation of the data.
{{< figure src="/images/automation-tools/nornir_napalm.png" title="" >}}<br>

{{< admonition type=bug title="Bug?" open=true >}}
Did you notice I had duplicates in _hosts.yaml_ and _groups.yaml_ file? That's due to Netmiko and NAPALM  uses different platform name for Huawei VRP routers, and a little workaround was needed to amend the incompatibility. 
{{< /admonition >}}

This lab is merely scratching the surface of what Nornir and other automation tools can achieve. 

## 3 Which automation tool to use?

Nobody can tell you which automation tool is the best; it all depends on the scenario. A great way to start learning automation is Python combined with Netmiko. Build a lab on either GNS3 or EVE-NG and writing some scripts will build your confidence over time. Although both Netmiko and NAPALM are great in lab environment, with real-world use cases, you may need to take scalability and performance into consideration. Ansible and Nornir are more capable in real-world production networks. Of course, you can make your own multiprocessing/multithreading script with lower level automation tools as well, but it makes the code more complicated than necessary. 

My personal choice of automation tool at the moment is Nornir, as it gives me the flexibility to write in Python, also it's very powerful when combined with various third-party tools. And it's fast, according to this [speed challange](https://networklore.com/ansible-nornir-speed/).