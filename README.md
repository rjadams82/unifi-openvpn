# unifi-openvpn
## Setup unifi gateway openvpn site to site tunnel

![USG OpenVPN](/2023-04-10%20Unifi%20USG%20OpenVPN.png)

Here is a basic guide for establishing an openvpn tunnel between a Unifi Security Gateway and an OpenVPN device. In this scenario I am connecting a Unifi USG-3P with cellular modem/gateways running openWRT or rOOter. I have made this connection on multiple devices including 

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

You will need this secret data for the USG site-to-site connection details and your openvpn remote site so `cat` the file to see the contents:

`sudo cat /config/auth/openvpn-secret-site1`

Highlight/select the entire contents of the output and copy to a text file on your local computer.

You will also need to copy just the secret hash string (all the data between -----BEGIN OpenVPN Static key V1----- & -----END OpenVPN Static key V1-----) for inputting the "Pre-shared key" in your Unifi site-to-site connection. Copy the string to a seperate text file and remove all line breaks in this string so it is a single line.

Now, using the Unifi application, add a site-to-site vpn connection:

**Settings > VPN > Site-to-Site VPN > Create**

- `VPN Protocol` - openVPN
- `Pre-shared Key` - only the hash string from the secret you created, in one line
- `Local Tunnel IP Address` - 172.16.0.1 *(This is L3 address of primary site TUN interface)*
- `Local port` - 1194 *(this is port for openVPN to establish connection. Additional site-to-site connections will need a different port number! 1195, 1196, ... etc)*
- `Shared Remote Subnets` - 192.168.2.0/24 *(This is LAN of Remote site; openVPN process will add route internally when the tunnel establishes)*
- `Remote IP Address` - site2.mynetwork *(you can make this the DDNS of your remote site OR literally anything since we will be overriding remote site IP)*
- `Remote Tunnel IP Address` - 172.16.0.2 *(This is L3 address of remote site TUN interface)*
- `Port` - 1194 *(this is remote site port for openVPN to establish connection. It should match your local port & your remote site .ovpn configuration file)*

Save the connection and allow a few minutes for Unifi to push the config to your USG.

The last piece of info you need is to tell the USG to allow this openVPN connection to "float" (which means the remote site may not always have the same IP address). This is important if your remote site uses a dynamic public IP or if you are using a cellular gateway behind carrier grade NAT (CGNAT). 

Log back into your USG cli, enter configuration mode `configure` and run the command `show interfaces openvpn`. You should see the configuration of the site you just entered, as a vtun## interface:

```
admin@usg-01# show interfaces openvpn
 openvpn vtun64 {
     description vpn-site-1
     ...
     local-address 172.16.0.1 {
     }
     local-port 1194
     mode site-to-site
     remote-address 172.16.0.2
     remote-host site2.mynetwork
     remote-port 1194
     shared-secret-key-file /config/auth/secret_58232fc8ad8_35e231d6cd948
 }
 ```

Now, edit the interface `edit interfaces openvpn vtun64` and issue command `set openvpn-option --float`. Then issue `show` command to see the results. You should now see the float option enabled.

```
admin@usg-01# show interfaces openvpn
 openvpn vtun64 {
     description vpn-site-1
     ...
     local-address 172.16.0.1 {
     }
     local-port 1194
     mode site-to-site
     openvpn-option --float
     remote-address 172.16.0.2
     remote-host site2.mynetwork
     remote-port 1194
     shared-secret-key-file /config/auth/secret_58232fc8ad8_35e231d6cd948
 }
 ```
At this point you could `commit` the configuration and test the tunnel. However because the USG is managed by Unifi, you will lose the updated configuraiton on the next change from the Unifi application. To persist this manual cli change you will need to utilize the config.gateway.json file on your Unifi controller.

## Unifi controller advanced configuration

To persist manual CLI commands you can make use of a config.gateway.json file on your unifi controller. Changes specified in this file are merged with the Unifi application configuration at provisioning time (after changes are made in Unifi application) before being pushed to your USG.

For more information on this advanced option you can read the Unifi help documents:
[https://help.ui.com/hc/en-us/articles/215458888-UniFi-USG-Advanced-Configuration-Using-config-gateway-json] 

I have provided a sample json file in this repository. The file must adhere to JSON standards and should contain a JSON representation of your USG custom config. You can use the sample file and adjust for your interface number, in my case "vtun64". I have also added an option for setting the logging level. The entire USG config is not contained in this file, only the commands that you would like added(merged) with your Unifi configuration.

```
{
      "interfaces": {
                 "openvpn": {
                        "vtun64": {
                                "openvpn-option": [
                                        "--float",                                        
                                        "--verb 3"
                                ]
                        }
                }
        }
}
```

You will need to gain file access to the directory where this file is stored, create or edit the file, then save it. Since Unifi can be run on many different systems you will need to check the documentation to figure out where to edit this file. I run a Unifi docker instance and I found mine in the unifi directory /config/data/sites/default which is mounted at /usr/lib/unifi

[https://help.ui.com/hc/en-us/articles/215458888-UniFi-USG-Advanced-Configuration-Using-config-gateway-json] 

## Setup openVPN far end (point-to point or p2p mode)
By far the easiest way to setup your openVPN remote connection is using an openvpn (.ovpn) configuration file. A simple plaintext file with .ovpn extension that contains your openVPN commands is all that is needed. One is included in the repository. Change the relevant fields to match your connection.

```
mode p2p
dev tun
persist-tun
# the primary site IP or hostname
remote site1.mynetwork.net
# port to listen and connect on - local and remote
port 1194
# can use UDP or TCP but UDP is recommended
proto udp
resolv-retry infinite
nobind
# verbosity of logging. the higher the number the more info for debugging
verb 3
keepalive 10 120
# the TUN adapter IP information, local remote
ifconfig 172.16.0.2 172.16.0.1
# specify the route to your primary site network, this will be added to the routing table on your device
route 192.168.1.0 255.255.255.0
# this is where your 'secret' will go. mak sure you copy everything from the secret file you generated
<secret> 
#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
this should be filled with your hash
-----END OpenVPN Static key V1-----
</secret>
```

Save this .ovpn file on your computer, then gain access to the remote site device and upload the .ovpn file into the openVPN setup.

## Testing

Once all steps are complete your devices should attempt to connect. On the USG you can check the logs to see if connections are being made
`run show log | match openvpn`
`run show log | match vtun`

You should see some output like below, showing the setup of the tunnel.

```
Apr 11 12:54:43 usg-01 openvpn[17383]: SIGUSR1[soft,ping-restart] received, process restarting
Apr 11 12:54:43 usg-01 openvpn[17383]: Restart pause, 2 second(s)
Apr 11 12:54:45 usg-01 openvpn[17383]: Static Encrypt: Cipher 'BF-CBC' initialized with 128 bit key
Apr 11 12:54:45 usg-01 openvpn[17383]: Static Encrypt: Using 160 bit message hash 'SHA1' for HMAC authentication
Apr 11 12:54:45 usg-01 openvpn[17383]: Socket Buffers: R=[294912->131072] S=[294912->131072]
Apr 11 12:54:46 usg-01 openvpn[17383]: TUN/TAP device vtun64 opened
Apr 11 12:54:46 usg-01 openvpn[17383]: TUN/TAP TX queue length set to 100
Apr 11 12:54:46 usg-01 openvpn[17383]: /sbin/ip addr add dev vtun64 local 172.16.0.1 peer 172.16.0.2
Apr 11 12:54:46 usg-01 openvpn[17383]: UDPv4 link local (bound): [undef]
Apr 11 12:54:46 usg-01 openvpn[17383]: UDPv4 link remote: [AF_INET]12.58.109.87:1194
```

### Troubleshooting commands

Check to see if the routes are added correctly in each system
`show ip route`

Check to see the status of the openVPN interface on the USG
`show interfaces openvpn detail `

Check to see summary status of all openVPN connections on the USG
`show openvpn status site-to-site`

Restart openVPN interface on the USG
`reset openvpn interface vtun64`
