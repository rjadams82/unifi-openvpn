# unifi-openvpn
## Setup unifi gateway openvpn site to site

Here are basic instructions for establishing an openvpn tunnel between a Unifi Security Gateway and an OpenVPN device. In this scenario I am connecting a Unifi USG-3P with modem/gateways running openWRT or rOOter. I have made this connection on multiple devices including 

- GL.iNet GL-X750
- GL.iNet GL-MT300N-V2 
- TPLink TL-WR902AC 

I'm fairly certain this will work on other openVPN devices but HAVE NOT tested. Your mileage may vary.

Some prerequisistes for this to work include

- familiarity with the USG EdgeOS cli
- access to your unifi controller file system
- understanding of the unifi controller configuration 
- registered dynamic DNS (DDNS) service
- 

It's important to note this is a layer 3 (L3) connection and will utilize a "tun" interface with an IP address. Therefore you will need available IP addresses that are not in use on your network to assign to these L3 tunnel interfaces.

## Setup Unifi Gateway
Add a site-to-site vpn connection in Unifi Network application.

Settings > VPN > Site-to-Site VPN

'VPN Protocol' openVPN
'Local Tunnel IP Address' 172.16.0.1 *This is the L3 address of the TUN interface


## Setup openVPN far end (point-to point or p2p mode)
Setup your openVPN connection using an openvpn (.ovpn) configuration file. 

### Troubleshooting commands

show ip route

show interfaces openvpn detail 

show openvpn status site-to-site

reset openvpn interface vtun65
