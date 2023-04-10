# unifi-openvpn
## Setup unifi gateway openvpn site to site tunnel

![USG OpenVPN](/2023-04-10%20Unifi%20USG%20OpenVPN.png)

Here is a basic guide for establishing an openvpn tunnel between a Unifi Security Gateway and an OpenVPN device. In this scenario I am connecting a Unifi USG-3P with modem/gateways running openWRT or rOOter. I have made this connection on multiple devices including 

- GL.iNet GL-X750
- GL.iNet GL-MT300N-V2 
- TPLink TL-WR902AC 

I'm fairly certain this will work on other openVPN devices but HAVE NOT tested. Your mileage may vary.

In this guide the Unifi USG represents my primary site and the openVPN device is at a remote site.

Some prerequisistes for this to work include

- familiarity with the USG EdgeOS cli
- access to your unifi controller file system
- understanding of the unifi controller advanced configuration [https://help.ui.com/hc/en-us/articles/215458888-UniFi-USG-Advanced-Configuration-Using-config-gateway-json] 
- registered dynamic DNS (DDNS) service
- basic understanding of ip subnetting and CIDR
- proper ip address planning/management

> It's important to note openVPN will build a VPN tunnel over layer 3 (L3) with a logical "tun" interface with an IP address. Therefore you will need available IP addresses that are not in use on your network to assign to these L3 tunnel interfaces. I used 172.16.0.x addresses for this purpose. 

> It is also worth mentioning that you are linking two seperate gateways each with their own LAN ip ranges. Since creating a site-to-site tunnel implies you want traffic to move between the two networks you need to ensure your ip ranges are all unique; you cant have 192.168.1.0/24 assigned to the LAN at both locations.

## IP Address Planning
You need to plan your IP addressing for this to work properly. Assuming your Primary site USG LAN is setup with 192.168.1.0/24 then you could feasibly allocate the next available /24 block (192.168.2.0/24) to your remote site. 

> If you anticipate expansion at your primary site you could reserve 192.168.0.0/21 for your primary site and the next available addressing would be 192.168.8.0/24 for your remote site. The critical requirement is that your primary and remote site do not have overlapping IP ranges.

Since openVPN p2p connections require layer 3 TUN interfaces, you will also need one unique private IP address per site per tunnel. In my example I am using 172.16.0.1 & 172.16.0.2 just because I dont have any other 172 addresses on my network so it will be easy to identify which addresses are used for tunnel adapters.

![USG OpenVPN](/2023-04-10%20Unifi%20USG%20OpenVPN%20L3.png)

So to recap here is my IP addressing plan

```
192.168.1.1/24 - Primary Site USG LAN
192.168.2.1/24 - Remote Site device LAN

172.16.0.1 - Primary Site TUN adapter
172.16.0.2 - Remote Site TUN adapater
```

## DDNS Setup
You can use the public IP address of your Primary site USG (wan interface) or a hostname/domain for your Remote openVPN to connect to. If you do not have a static IP on your USG internet connection your VPN will obviously fail if your public IP address changes. The easiest way to get around this is by using a DDNS service. 

This is a topic that is covered in depth elsewhere so I won't go into detail. For more help with DDNS checkout UI article [https://help.ui.com/hc/en-us/articles/9203184738583-UniFi-Gateway-Dynamic-DNS]

## Setup Unifi Gateway
Log into USG cli to generate auth key. Name the key so you know which site-to-site connection it is for. (openvpn-secret-site1 etc)

`generate vpn openvpn-key /config/auth/openvpn-secret-site1`

You will need this key for the USB site-to-site connectino details and your openvpn remote site so copy the contents to a text file for now:

`sudo cat /config/auth/openvpn-secret-site1`

Using the Unifi application, add a site-to-site vpn connection.

**Settings > VPN > Site-to-Site VPN > Create**

- `VPN Protocol` - openVPN
- `Pre-shared Key` - only the hash from the secret you created, in a one line string
- `Local Tunnel IP Address` - 172.16.0.1 *(This is L3 address of TUN interface)*
- `Shared Remote Subnets` - 192.168.2.0/24 *(This is LAN of Remote site; openVPN process will add route internally when the tunnel establishes)*
- `Remote IP Address` - site2.mynetwork *(you can make this anything since we will be overriding remote site IP)*
- `Local port` - 1194 *(this is port for openVPN to establish connection. Additional site-to-site connections will need a different port number! 1195, 1196, ... etc)*

## Setup openVPN far end (point-to point or p2p mode)
By far the easiest way to setup your openVPN remote connection is using an openvpn (.ovpn) configuration file. A simple plaintext file with .ovpn extension that contains your openVPN commands is all that is needed.

```
dev tun
remote r0.ra-net.net
port 1195
proto udp
resolv-retry infinite
mode p2p
nobind
persist-tun
verb 3
keepalive 10 120
ifconfig 172.16.0.4 172.16.0.3
route 10.0.0.0 255.0.0.0
<secret>
#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
(this is where your 'secret' will go)
-----END OpenVPN Static key V1-----
</secret>
```

### Troubleshooting commands

show ip route

show interfaces openvpn detail 

show openvpn status site-to-site

reset openvpn interface vtun65
