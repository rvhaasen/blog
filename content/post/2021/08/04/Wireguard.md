---
title: "Wireguard"
date: 2021-08-04T20:56:13Z
draft: true
---
**When deploying an IoT setup at location**, the nodes are typicaly put behind a NAT router in the network on location. Although the nodes can typicaly reach outside (dependend on how strict the network has been configured) it is not possible to reach the nodes from outside the router. In the toughest situation, where only port 80 is accesible from within the network, websocket base solution can be considered (see Inlets, created by Alex Ellis)
Wireguard is an interesting VPN solution. This document describes how this has been done. Wireguard is not part of a linux distribution yet, so installation takes a bot more effort than 'sudo apt get ....'.
After the installation on the peer nodes, getting the vpn working went without problems after having found how to setup the routing rules in the EC2 server node.
Getting SSH working took more efforts. After having found that the problem had to do with MTU of the wg-interfaces on the Pi, this was solved.

# Usecases
- Accessing RPi in local network via external node
- Accessing RPi via node in other private network. This reflects the situation where RPi's on deployment site should be accessible from for example Signfiy internal network.
This mode will enable configuring the remote nodes (CI/CD), for example configuration via Ansible of Kubernetes.

# Setup
## Setting up external VPN server node
For this, a micro instance has been setup on AWS with an Ubuntu server image. This server will be refered to as 'wg-server' \
Accessing wg-server:

ssh -i "C:\Users\nlv09202\.ssh\wg_server.pem" ubuntu@ec2-63-35-185-147.eu-west-1.compute.amazonaws.com

After the setup, inbound rule for echo-request (ping) can be added. And see that tne node responds. The same method will be used later to allow access to the vpn.

Setting up wireguard on wg-server
```
$ sudo -s
root# apt-add-repository ppa:wireguard/wireguard
root# apt update
root# apt install wireguard
root# umask 077
root# mkdir /etc/wireguard
root# wg genkey > /etc/wireguard/private.key
root# ip link add wg0 type wireguard
root# wg set wg0 private-key /etc/wireguard/private.key
root# wg showconf wg0 > /etc/wireguard/wg0.conf
root# echo "Address = 100.64.0.1/10 >> /etc/wireguard/wg0.conf
```
In case firewall on the server is an issue (currently unsolved problem how to ssh from one peer to another) following rules should be added to the interface

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

The config file /etc/wireguard/wg0.conf now should look something like this:
```
[Interface]
Address = 100.64.0.1/10
#MTU setting is requruired for RPi's (see discussion below)
MTU = 1412 
#SaveConfig = true
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POS
TROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D P
OSTROUTING -o eth0 -j MASQUERADE
ListenPort = 42840
PrivateKey = xxxxxxxxxxxxxxxxxxxxxxx

```
With this configuration, the network-interface can be started
```
root# ip link del wg0
root# wg-quick up wg0
root# wg-show
interface: wg0
  public key: NuHDg0RykQ6hdevLxBaSlCuTuSD1QlxKiG2qmzUZywM=
  private key: (hidden)
  listening port: 42840
```
The public key, node-ip address and listening port are needed for configuring the peers.

After the peers have been configured, the public-keys of the peers will be used to extend the configuration of wg-server.

Setting up Wireguard on RPi Raspbian
Update 09 Oct 2020 H re a link to a recent description of the installation process on Rasbberry Pi OS, tested and worked perfect: https://www.sigmdel.ca/michel/ha/wireguard/wireguard_03_en.html

Note: recently I found a new blog that nicely describes the process in raspbian : https://engineerworkshop.com/blog/how-to-set-up-wireguard-on-a-raspberry-pi/

Configuring RPi https://www.sigmdel.ca/michel/ha/wireguard/wireguard_02_en.html#buster
```
echo "deb http://deb.debian.org/debian/ unstable main" | sudo tee --append /etc/apt/sources.list.d/unstable.list
wget -O - https://ftp-master.debian.org/keys/archive-key-$(lsb_release -sr).asc | sudo apt-key add -
printf 'Package: *\nPin: release a=unstable\nPin-Priority: 150\n' | sudo tee --append /etc/apt/preferences.d/limit-unstable
sudo apt update
sudo apt install wireguard -y
sudo -s
umask 077
root# wg genkey > /etc/wireguard/private.key
root# wg genkey | tee /etc/wireguard/private.key | wg pubkey > /etc/wireguard/client_public_key
root# ip link add wg0 type wireguard
root# wg set wg0 private-key /etc/wireguard/private.key
root# wg showconf wg0 > /etc/wireguard/wg0.conf

For nodes behind NAT (which is the case) add following to the config file
PersistentKeepalive=25
```
The procedure as described above did not work on the second RPi. Following solution did work (from https://www.reddit.com/r/pihole/comments/bnihyz/guide_how_to_install_wireguard_on_a_raspberry_pi/)
```
git clone https://git.zx2c4.com/WireGuard
cd WireGuard/
cd src/
make

# ? If you get an error here that says  "No such file or dir" you're probably on an older kernel. Fix it by running 'sudo BRANCH=stable rpi-update' (refer to “Troubleshooting” at the end to update it manually)

sudo make install
```
Configure wg to use the external server 
```
cat << EOF >> EOF /etc/wireguard/wg0.conf
Address = 100.64.0.101/32
[Peer]
PublicKey = <pub-key of wg-server>
AllowedIPs = 100.64.0.0/10
Endpoint = <ip address of wg-server:port>
```
Add the key to the config of wg_server:
```
root# cat << EOF >> /etc/wireguard/wg0.conf                                               
[Peer]                                                                                                                 
PublicKey=vL2byDvX281fra3Xe9JSTu3CjHypL1UzHNfLPDCkI2A=                                                                 
AllowedIPs = 100.64.0.101/32                                                                            
EOF
```
This means that packets from a node with the specified public key and source address as specified in AllowedIPs will be accepted.

## Tunneling external access of peers through the vpn
If all external data of the peers has to go through the vpn interface, AllowedIPs should be set to 0.0.0.0/0

In that case, ip-tables on wg-server has to be configured to use masquerading NAT:
```
$ sudo iptables -t nat -A POSTROUTING -s 10.64.0.0/10 -o eth0 -j MASQUERADE

# or in firewalld
$ sudo firewall-cmd --zone=external --add-masquerade --permanent
$ sudo firewall-cmd --reload
```
Now remove and restart wg0 on wg_server
```
root# ip link del wg0                                                                     
root# wg-quick up wg0                                                                    
[#] ip link add wg0 type wireguard                                                                                     
[#] wg setconf wg0 /dev/fd/63                                                                                          
[#] ip -4 address add 100.64.0.1/10 dev wg0                                                                            
[#] ip link set mtu 8921 up dev wg0
```
Check wg
```
interface: wg0
  public key: NuHDg0RykQ6hdevLxBaSlCuTuSD1QlxKiG2qmzUZywM=
  private key: (hidden)
  listening port: 42840

peer: vL2byDvX281fra3Xe9JSTu3CjHypL1UzHNfLPDCkI2A=
  endpoint: 213.127.32.167:48202
  allowed ips: 100.64.0.101/32
  latest handshake: 11 minutes, 24 seconds ago
  transfer: 44.34 KiB received, 32.98 KiB sent
```
Next add an inbound rule in order to allow UDP on port 42840

Try to ping the nodes on both sides, it should work.

**Starting wireguard as service on wg-server**
First stop the interface:
$  sudo wg-quick down wg0

Configure to auto-start service at boot-time:
$  sudo systemctl enable --now wg-quick@wg0
## Usecase 1: ssh into the RPi from the external node
ssh into the ws-node which acts as a 'jump-box'. From there ssh into the RPi

## Usecase 2: ssh into the RPi from a second RPi in another private network.
For this, the second RPi resides in the same private network. This effectively is similar to the second RPi in another private network.

The second RPi will be configured in the same way as the first. The peer address will be added to the server.


After that, the RPi's could ping eachother using the vlan addresses.
ssh however not, it 'hangs'...
Using python3 http.server on one, and accessing it via telnet did work. So the connection seemst to be fine and the

Tried solution from https://gist.github.com/insdavm/b1034635ab23b8839bf957aa406b5e39 

still does not work, ssh hangs..

## Final part to get it working: MTU 
Finaly found the problem! 
The default MTU size (1500) did not work. By trying different values, 1400 (MTU = 1400) was a value that worked. I did not try to further increase it to find the maximum vallue that works.

(This setting was applied on all nodes, don't know if this is needed on the EC2 server node...)

It would be better to know how to set this value. Anyhow, 1400 works fine.

### Setting up Wireguard on RPi Archlinux
On arch, installing wireguard is easy:

#pacman - S wireguard-tools
#pacman - S wireguard-dkms
#pacman - S wireguard-linux-headers

After that , the installation is the same as for Raspbian

## Wireguard service dependency causing service start to take 2 minutes
After enabling the service and testing it by starting it manualy, it takes 2 minutes until the prompt returns.

The reason is the dependency of target network-online.target
```
# /usr/lib/systemd/system/wg-quick@.service
[Unit]
Description=WireGuard via wg-quick(8) for %I
After=network-online.target nss-lookup.target
Wants=network-online.target nss-lookup.target
PartOf=wg-quick.target
Documentation=man:wg-quick(8)
Documentation=man:wg(8)
Documentation=https://www.wireguard.com/
Documentation=https://www.wireguard.com/quickstart/
Documentation=https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8
Documentation=https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/wg-quick up %i
ExecStop=/usr/bin/wg-quick down %i
Environment=WG_ENDPOINT_RESOLUTION_RETRIES=infinity

[Install]
WantedBy=multi-user.target
```
This target is created by service systemd-networkd-wait-online.

The service will check for all links to be available. The RPi has 2 links, wlan0 and eth0. If eth0 is not connected (because wlan0 is used), then the target network-online.target will never be valid

The solution is to ignore eth0  (https://wiki.archlinux.org/index.php/Talk:WireGuard)

Modify the service: 

$sudo systemctl edit --full --force systemd-networkd-wait-online

The default editor will be opened with the current version of the unitfile

Add --ignore eth0 as shown in the unit-file:
```
#  SPDX-License-Identifier: LGPL-2.1+
#
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.

[Unit]
Description=Wait for Network to be Configured
Documentation=man:systemd-networkd-wait-online.service(8)
DefaultDependencies=no
Conflicts=shutdown.target
Requires=systemd-networkd.service
After=systemd-networkd.service
Before=network-online.target shutdown.target

[Service]
Type=oneshot
ExecStart=/usr/lib/systemd/systemd-networkd-wait-online --ignore eth0
RemainAfterExit=yes

[Install]
WantedBy=network-online.target
```
Write-exit the editor, the updated service will be enabled.

Test that the wg-service start now returns immediately 

Nodes only reachable after ping..
In the setup with the external wg-server (EC2) it seems that when a peer is started, they are not yet 'known' to the server. When 2 peers are started, and 1 tries to ping the other, the wg-server responds with a message that the peer is not reachable. Only after the destination peer also pings (to either the wg-server or the other node) then both pings work.
This might be explained in https://serverfault.com/questions/985425/only-able-to-connect-to-wireguard-peer-after-i-ping-the-server

TODO: find a solution 
```
```
