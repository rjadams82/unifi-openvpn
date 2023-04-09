# unifi-openvpn
## Setup unifi gateway openvpn site to site tunnel

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

## Setup Unifi Gateway
Add a site-to-site vpn connection in Unifi Network application.

**Settings > VPN > Site-to-Site VPN > Create**

- `VPN Protocol` - openVPN
- `Local Tunnel IP Address` - 172.16.0.1 *This is the L3 address of the TUN interface
- `Local port` - 1194 *this is the port for openVPN to establish a connection. If you add additional site-to-site connections you will need to use a different port number! 1195, 1196, ... etc
- `Shared Remote Subnets` - 192.168.2.0/24 *This is the 

## Setup openVPN far end (point-to point or p2p mode)
Setup your openVPN connection using an openvpn (.ovpn) configuration file. 

### Troubleshooting commands

show ip route

show interfaces openvpn detail 

show openvpn status site-to-site

reset openvpn interface vtun65
