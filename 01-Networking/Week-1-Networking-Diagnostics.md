## Week 1: Networking Diagnostics

# Goal
To map my local network topology and test DNS/Port
connectivity to establish a baseline for IAM troubleshooting.

## Commands Run & Output
*ip a*

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp1s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
    link/ether c8:5a:cf:b8:6f:9d brd ff:ff:ff:ff:ff:ff
3: wlp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 10:6f:d9:88:81:b5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.6/24 brd 192.168.1.255 scope global dynamic noprefixroute wlp2s0
       valid_lft 3164sec preferred_lft 3164sec
    inet6 2402:800:6e1a:89b5:48ef:441e:1dfc:aa24/64 scope global temporary dynamic 
       valid_lft 79431sec preferred_lft 79431sec    inet6 2402:800:6e1a:89b5:20cc:8c08:bc21:dd19/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 79431sec preferred_lft 79431sec
    inet6 fe80::f262:a098:573:513a/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether d2:24:55:3c:b0:cb brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever

*ip route*
Server:		127.0.0.53
Address:	127.0.0.53#53

*nslookup* (google.com)
Server:		127.0.0.53
Address:	127.0.0.53#53

Address: 142.250.199.206
Name:	google.com
Address: 2404:6800:4005:81b::200e
default via 192.168.1.1 dev wlp2s0 proto dhcp src 192.168.1.6 metric 600 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
192.168.1.0/24 dev wlp2s0 proto kernel scope link src 192.168.1.6 metric 600 


# Analysis
Private ip = `192.168.1.6`
Default gateway = `192.168.1.1` 
DNS resolution = `Server: 127.0.0.53`


#Process
First, I would verify if the user has general internet connectivity.
If they can browse the web but can't log in, it usually means the workstation
cannot resolve the Domain Controller. I would check if the workstation is
pointing to the correct internal DNS server, and verify that the DNS service
on the Domain Controller is running.

  Test internet connection
- *1.1.1.1* search this in your web browser, if it returns the page you
are connected to the internet
-  ping *8.8.8.8* in CLI and it will corresponde will ttl's if you are
 connected. It will time out if you not connected
 
  Check and test DNS
- *cat /etc/resolv.conf*  or *nslookup* (works for both linux and windows) 

# Troubleshooting Scenario (Mock Interview Prep)
In this exercise i will break the DNS by assigning a DNS that wont resovle
 (fake) and then fix the DNS again.

Commands
- *sudo resolvectl dns wlp2s0 10.255.255.1* This changes the DNS to a bad DNS
- *sudo resolvectl flush-caches* This cleans the cache incase the correct DNS
 is cached in which case the DNS would just revert back to default DNS in most modern systems.
- *nslookup google.com* This should return "servfail" because of incorrect DNS
- *sudo resolvectl dns wlp2s0 192.168.1.1* This returns to my default gateway
- *sudo resolvectl flush-caches* This clears the cache incase the bad DNS is still there  
 
findings
- What I tried: Used *resolvectl dns wlp2s0 10.255.255.1* to point my Wi-Fi adapter to
 a dead DNS server, then flushed the cache *resolvectl flush-caches*.
- The Result: *nslookup google.com* STILL worked. 
- Why it happened: *systemd-resolved* has built-in redundancy. It detected
 the primary DNS was dead and fell back to a global/fallback DNS to maintain
 internet connectivity. 
- How this maps to Active Directory: This proves the importance of DNS redundancy.
 If a workstation has a secondary DNS server (like a secondary Domain Controller)
 configured, it will still be able to resolve AD SRV records and log in, even if
 the primary DC goes down. In an enterprise, you NEVER configure just one DNS
 server for a workstation.
