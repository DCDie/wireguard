## Install
Execute the apt command/apt-get command to install security updates for Debian 11
```bash
sudo apt update
```
```bash
sudo apt install wireguard wireguard-tools linux-headers-$(uname -r)
```

<span>&nbsp;&nbsp;</span>
<span>&nbsp;&nbsp;</span>

## Configure Wireguard Server
First, a pair of private and public keys must be generated for the WireGuard server. Let’s get to the /etc/wireguard/ directory using the cd command.
```bash
sudo -i
```
```bash
cd /etc/wireguard/
```

Now proceed and run the following line of code:
```bash
umask 077; wg genkey | tee privatekey | wg pubkey > publickey
```

Note if that command fails to do the trick for you, run this alternate command on your terminal:
```bash
wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
```

We can check the created keys using the ls and cat command as illustrated below:
```bash
ls -l privatekey publickey
```
```bash
cat privatekey && cat publickey
```

<span>&nbsp;&nbsp;</span>
<span>&nbsp;&nbsp;</span>

### Type of configuration
We can configure our wireguard server in 2 ways, one for using


for accessing private network, another one for redirect all our traffic via ip public on *wireguard server*.

#### Access private network
Open your editor and add the following to a new file called wg0.conf:
```bash
sudo vim /etc/wireguard/wg0.conf
```

Append the following lines:
```bash
## Edit or create WireGuard VPN on Debian By Editing/Creating wg0.conf File ##
[Interface]
## IP address ##
Address = 192.168.10.1/24
 
## Server port ##
ListenPort = 51122
 
## private key i.e. /etc/wireguard/privatekey ##
PrivateKey = ${content_of_private_key}
 
# IP forwarding
PreUp = sysctl -w net.ipv4.ip_forward=1
# IP masquerading
PreUp = iptables -t mangle -A PREROUTING -i wg0 -j MARK --set-mark 0x30
PreUp = iptables -t nat -A POSTROUTING ! -o wg0 -m mark --mark 0x30 -j MASQUERADE
PostDown = iptables -t mangle -D PREROUTING -i wg0 -j MARK --set-mark 0x30
PostDown = iptables -t nat -D POSTROUTING ! -o wg0 -m mark --mark 0x30 -j MASQUERADE

# firewall local host from wg peers
PreUp = iptables -A INPUT -i wg0 -m state --state ESTABLISHED,RELATED -j ACCEPT
PreUp = iptables -A INPUT -i wg0 -j REJECT
PostDown = iptables -D INPUT -i wg0 -m state --state ESTABLISHED,RELATED -j ACCEPT
PostDown = iptables -D INPUT -i wg0 -j REJECT
# firewall wg peers from other hosts
PreUp = iptables -A FORWARD -o wg0 -m state --state ESTABLISHED,RELATED -j ACCEPT
PreUp = iptables -A FORWARD -o wg0 -j REJECT
PostDown = iptables -D FORWARD -o wg0 -m state --state ESTABLISHED,RELATED -j ACCEPT
PostDown = iptables -D FORWARD -o wg0 -j REJECT

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```
> Make sure you change eth0 with the name of your local network interface after -A POSTROUTING.

<span>&nbsp;&nbsp;</span>

**Breakdown of the wg0.conf settings**
1.  **Address** – A list of v4 or v6 IP addresses for the wg0 interface, separated by commas. You can choose an IP address from the private network range
2. **ListenPort** – The port for listening.
3. **PrivateKey** – A private key created by running the wg genkey command. (To view the file’s contents, use sudo cat /etc/wireguard/privatekey.)
4. **PostUp** – A command or script run before the interface is created. In this example, we’re enabling masquerade with iptables. This permits traffic to exit the server, providing VPN clients with Internet access.

**NOTE**: Ensure you make the config file unreadable to users by executing this code:
```bash
sudo chmod 600 /etc/wireguard/{privatekey,wg0.conf}
```

Now launch the wg0 interface by running this line of code:
```bash
sudo wg-quick up wg0
```

To check the status of the interface, execute this command:
```bash
sudo wg show wg0
```
After stop server and run with systemctl.
```bash
systemctl start wg-quick@wg0
```

Enable systemctl service for *Wireguard* at startup:
```bash
systemctl enable wg-quick@wg0
``` 
> Firewall must be configured to allow from anywhere port of WG Server (protocol UDP)


#### Cover your public IP
Turn on IP forwarding on the server. We must activate IP forwarding on the VPN server for it to transit packets between VPN clients and the Internet. To do so, change the sysctl.conf file.
```bash
sudo vim /etc/sysctl.conf
```
Insert the syntax below at the end of this file.
```bash
net.ipv4.ip_forward = 1
```
Save the file, close it, and then apply the modifications using the below command. The -p option loads sysctl configuration from the /etc/sysctl.conf file. This command will save our modifications across system restarts.
```bash
sudo sysctl -p
```

Open your editor and add the following to a new file called wg0.conf:
```bash
sudo vim /etc/wireguard/wg0.conf
```

Append the following lines:
```bash
## Edit or create WireGuard VPN on Debian By Editing/Creating wg0.conf File ##
[Interface]
## IP address ##
Address = 192.168.10.1/24

## Server port ##
ListenPort = 51122

## private key i.e. /etc/wireguard/privatekey ##
PrivateKey = ${content_of_private_key}

PostUp = iptables -A FORWARD -i eth0 -o wg0 -j ACCEPT; iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i eth0 -o wg0 -j ACCEPT; iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```
> Make sure you change eth0 with the name of your local network interface after -A POSTROUTING.

Next steps you can follow from the previous step [[#Access private network]]

## Configure Wireguard Client

Download the wireguard client [link](https://www.wireguard.com/install/) Create a new empty file, after generating an empty file you will automatically be generated a public and private key. Put the public key in a peer of *Wireguard Server*.
```bash
...

[Peer]
AllowedIPs = 192.168.10.2/32
PublicKey = bRHDBzZd2aVah8evpSa/XUVK86aUpNpFTzEK/xseBns=

```

**In your empty config file add the next configurations**
Depending on which type of *Wireguard Server* is, you must set parameters in.

1. Access private network:
```bash
[Interface]
PrivateKey = ${private_key_of_client}
# Private address of client from virtual network
Address = 192.168.10.2/32
  
[Peer]
PublicKey = ${public_key_of_wireguard_server}
# Private network where you will need access, first is virtual network of vpn, second is private network on cloud.
AllowedIPs = 192.168.10.0/24, 10.0.0.0/24

Endpoint = ${public_ip_of_wireguard_server}$:51122

PersistentKeepalive = 20
```

2. Route traffic via *Wireguard Server*
```bash
[Interface]

PrivateKey = ${private_key_of_client}
# Private address of client from virtual network
Address = 192.168.10.2/24
# DNS which will use client [required]
DNS = 1.1.1.1, 8.8.8.8


[Peer]

PublicKey = ${public_key_of_wireguard_server}
# Here is there rule where you manage your traffic to route all via Wireguard Server
AllowedIPs = 0.0.0.0/0, ::/0

Endpoint = ${public_ip_of_wireguard_server}$:51122
```

## Script for automatic install *Wireguard Server*
This project is a bash script that aims to setup a [WireGuard](https://www.wireguard.com/) VPN on a Linux server, as easily as possible!
The server will apply NAT to the client's traffic so it will appear as if the client is browsing the web with the server's IP.
[github_repo](https://github.com/angristan/wireguard-install)

## Dashboard for wireguard
Dashboard hellping you with viewing all configurations and manage them in a easier way.
[github_repo](https://github.com/donaldzou/WGDashboard#-install)

*Notes*: must to be installed with *apt gunicorn*, *pip ifcfg,flask,flask_qrcode,icmplib*

