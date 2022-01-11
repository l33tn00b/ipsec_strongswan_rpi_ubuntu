# ipsec_strongswan_rpi_ubuntu
Hooking up a RasPi and Ubuntu 20.04 LTS public server with IPSec.

# Prerequisites
- IP forwarding has to be activated on both machines.
- We don't mess with the MTU.
- Make sure the firewall accepts connections on UDP 500/4500 at the public side. 
- Don't forget about any cloud based firewall your public server might be behind. *bonk*
- You're root on the Ubuntu server (no need to sudo).

# Installation
## RasPi
- `sudo apt-get install strongswan`

## Ubuntu
- `sudo apt-get install strongswan`

# Configuration
## RasPi
- `sudo nano /etc/ipsec.conf`
```
config setup
        # strictcrlpolicy=yes
        uniqueids = no
        # turn on some debug info
        charondebug = "ike 2, knl 2, cfg 2"

# Add connections here.

conn <name>
        right=<public ip of server to connect to>
        rightid=%<ip from above>
        rightsubnet=10.8.5.0/24
        auto=add
        compress=no
        type=tunnel
        # we're the one behind a nat
        # so let's restart the connection if 
        # it goes dead because of an ip change
        dpdaction=restart
        dpddelay=300s
        rekey=no
        keyexchange=ikev2
        # just some random, mostly secure ciphers
        ike=aes256-sha1-modp1024,3des-sha1-modp1024!
        esp=aes256-sha1,3des-sha1!
        fragmentation=yes
        forceencaps=yes
        # we chose to have a pre-shared key/secret because this is
        # point-to-point
        authby=secret
        leftsourceip=%config
```
- `sudo nano /etc/ipsec.secrets`
```
%any : PSK "<supersecretrandomstring64chars>"
```

## Ubuntu
- `nano /etc/ipsec.conf`
```
 basic configuration

config setup
        # strictcrlpolicy=yes
        uniqueids = no
        # add some debug information
        charondebug = "ike 2, knl 2, cfg 2"

# Add connections here.

conn presharedkey
        # we clear the connection
        # public ip, wait for client behind NAT 
        # to reconnect after having obtained a fresh ip
        dpdaction=clear
        dpddelay=300s
        rekey=no
        auto=add
        compress=no
        type=tunnel
        keyexchange=ikev2
        ike=aes256-sha1-modp1024,3des-sha1-modp1024!
        esp=aes256-sha1,3des-sha1!
        fragmentation=yes
        forceencaps=yes
        left=%any
        leftid=<public server ip>
        leftsubnet=10.8.5.0/24
        right=%any
        rightid=%any
        rightsourceip=10.8.5.0/24
        # just auth by secret (point-to-point)
        authby=secret
```
- `sudo nano /etc/ipsec.secrets`
```
%any : PSK "<supersecretrandomstring64chars>"
```

# Activate
## RasPi
- `sudo systemctl restart strongswan`
- `sudo ipsec up <connection name`

## Ubuntu
- `systemctl restart strongswan-starter`


# Verfify
## RasPi
- `ip addr` shows ip 10.8.5.1 on interface eth0 (as configured): 
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether dc:a6:32:ac:51:12 brd ff:ff:ff:ff:ff:ff
    inet XX brd XX scope global dynamic noprefixroute eth0
       valid_lft 1863sec preferred_lft 1413sec
    inet 10.8.5.1/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::73aa:c0b4:5ae8:de4f/64 scope link
       valid_lft forever preferred_lft forever
```

## Ubuntu
- check for incoming connections/debug: `cat /var/log/syslog | grep charon`
 
