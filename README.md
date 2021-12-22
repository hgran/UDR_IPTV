# UDR_IPTV
Configure IGMP Proxy on UDR to get IPTV work on Unifi Dream Router


## Configuring Internal LAN
(Source: Create IPTV LAN according to https://github.com/fabianishere/udm-iptv#configuring-internal-lan)

To operate correctly, the IPTV decoders on the internal LAN possibly require
additional DHCP options. You can add these DHCP options as follows:

1. In your UniFi Dashboard, go to **Settings > Networks**.
2. Select the LAN network on which IPTV will be used.
   We recommend creating a separate LAN network for IPTV traffic if possible in
   order to reduce interference of other devices on the network.
4. Enable **Advanced > IGMP Snooping**, so IPTV traffic is only sent to
   devices that should receive it.
5. Go to **Advanced > DHCP Option** and add the following options:

   | Name      | Code | Type       | Value          |
   |-----------|:----:|------------|----------------|
   | IPTV      |  60  | Text       | IPTV_RG        |
   | Broadcast |  28  | IP Address | _BROADCAST_ADDRESS_ |

   Replace _BROADCAST_ADDRESS_ with the broadcast address of your LAN network.
   To get this address, you can obtain it by setting all bits outside the subnet
   mask of your IP range, for instance:
   ```
   192.168.X.1/24 => 192.168.X.255
   192.168.0.1/16 => 192.168.255.255
   ```
   See [here](https://en.wikipedia.org/wiki/Broadcast_address) for more
   information.

## Install IGMP Proxy on UDR
1. Make sure you have enable SSH access to the UDR (source: https://evanmccann.net/blog/2020/5/udm-ssh)
   Enable SSH in the UDR Device settings:
   Click on the gear icon to access the UDR device settings
   Click on Advanced
   Enable SSH and set your SSH password
2. SSH to the UDR using the credentials created in 1.
3. apt install igmpproxy
4. run ifconfig to find the broadcast interface for the IPTV LAN (For me it is br3
```
br3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.4.1  netmask 255.255.255.0  broadcast 0.0.0.0
```
6. Edit the file /etc/igmpproxy.conf to look like this:
 ```
quickleave

##------------------------------------------------------
## Configuration for eth4 (Upstream Interface)
##------------------------------------------------------
phyint eth4 upstream  ratelimit 0  threshold 1
        altnet 192.168.0.0/24

##------------------------------------------------------
## Configuration for br4 (IPTV Broadcast Interface)
##------------------------------------------------------
phyint br3 downstream ratelimit 0 threshold 1
```
5. Run igmpproxy with debug and verbose flags
```
igmpproxy -v -d /etc/igmpproxy.conf
```
6. Check the output to find the subnets of you provider and update /etc/igmpproxy.conf with them, example:
 ```
quickleave

##------------------------------------------------------
## Configuration for eth4 (Upstream Interface)
##------------------------------------------------------
phyint eth4 upstream  ratelimit 0  threshold 1
        altnet 192.168.0.0/24
        altnet 233.89.188.0/24
        altnet 239.251.255.0/24
        altnet 239.255.255.0/24
        altnet 94.246.72.0/24
        altnet 88.87.37.0/24

##------------------------------------------------------
## Configuration for br4 (IPTV Broadcast Interface)
##------------------------------------------------------
phyint br4 downstream ratelimit 0 threshold 1
```
7. Set the Port Profile on the port where you have the setop box to use the IPTV LAN
8. Create systemd config: /etc/systemd/system/igmpproxy.service
```
[Unit]
Description=IGMP proxy
After=network.target

[Service]
ExecStart=/usr/sbin/igmpproxy /etc/igmpproxy.conf

[Install]
WantedBy=multi-user.target
```
10. Start the igmp proxy systemd service
```
systemctl start igmpproxy.service
```
