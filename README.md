# VPN Killswitch

 > This is inspired from [CK-s-Firewall-killswitch](https://github.com/CHEF-KOCH/CK-s-Firewall-killswitch)

A VPN (Virtual Private Network) is often used to avoid censorship, surveillance, or geolocation. This is done by routing the internet traffic from your device to the remote VPN server through an encrypted tunnel. Sometimes, the VPN connection may drop, which will result your traffic being transmitted through the public internet rather than an encrypted VPN tunnel.

There are some reasons that cause a disconnection

* VPN Protocol
* Not good enough signal strength
* Firewall/router configuration

The chance of a drop in the connection to the remote server is less, but there may be an unexpected drop which shouldn't be risked on as it will reveal the user's real IP address.

For this reason, A Kill switch technique is implemented, which would prevent the unencrypted/unprotected access to the internet.

# VPN Killswitch with IPTABLES

Most of the VPNs do come with a killswitch, but are not as reliable as using iptables (as it is not dependent on the VPN service and is a kernel feature).

## Requirements

* A Linux machine with root privileges
* A VPN provider
* `iptables` should be installed in the machine

## Rules

* Set the base rules to disallow all the traffic

```
iptables -P INPUT DROP
iptables -P FORWARD DROP
ipables -P OUTPUT DROP
```

* Allow Loopback and Ping. Assuming that the VPN connection is on `tun0`, check with `ip a`.

```
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -o lo -j ACCEPT
iptables -A OUTPUT -o tun0 -j ACCEPT
```

* Allow to communicate to the DHCP server

```
iptables -A OUTPUT -d 255.255.255.255 -j ACCEPT
iptables -A INPUT -s 255.255.255.255 -j ACCEPT
```

* Allow to communicate within the LAN

```
iptables -A INPUT -s 192.168.1.0/24 -d 196.168.1.0/24 -j ACCEPT
iptables -A OUTPUT -s 192.168.1.0/24 -d 196.168.1.0/24 -j ACCEPT
```

* Allow established sessions to receive traffic

```
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

* Allow DNS. To do this, get the VPN's DNS server. To do so, see the `resolv.conf` if the VPN has access to it

```
iptables -A OUTPUT -d 193.138.218.74 -j ACCEPT
```

* Allow the VPN connection. 

In this context [Mullvad VPN](https://mullvad.net/) is used. Specifically Singapore

```
iptables -A OUTPUT -o eth*  -p udp -m multiport --dports 53,1300:1302,1194:1197 -d 185.128.24.0/24,37.120.208.0/24 -j ACCEPT
```

Assuming the network interface as `eth*`.

**Or**

In this context [OpenVPN](https://openvpn.net/) is used used.

```
iptables -A OUTPUT -p udp -m udp --dport 1194 -j ACCEPT
```

* IPv6

It is preferred to drop the IPv6 connections

```
ip6tables -P INPUT DROP
ip6tables -P FORWARD DROP
ip6tables -P OUTPUT DROP
```

* Log all the dropped packages (debug only)

```
iptables -N logging
iptables -A INPUT -j logging
iptables -A OUTPUT -j logging
iptables -A logging -m limit --limit 2/min -j LOG --log-prefix "IPTables: " --log-level 7
iptables -A logging -j DROP
```

* To save the rules

```
iptables-save > /etc/iptables/iptables.rules
```

* To restore the rules

```
iptables-restore < /etc/iptables/iptables.rules
```

* For Persistent rules, use `iptables-persistent`.
