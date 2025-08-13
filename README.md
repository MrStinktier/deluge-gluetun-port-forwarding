This Docker image is a modified version of [secunit404](https://github.com/secunit404)'s [gluetun-deluge-port-manager](https://github.com/secunit404/gluetun-deluge-port-manager).

# gluetun-deluge port forwarding
Automatically updates the listening port for deluge to the port forwarded by [Gluetun](https://github.com/qdm12/gluetun/). 

Using qBittorent? See here: [gluetun-qbittorrent-port-manager](https://github.com/plaexmaster/gluetun-qbittorrent-port-manager).

## Description
[Gluetun](https://github.com/qdm12/gluetun/) has the ability to forward ports for supported VPN providers, but Deluge does not have the ability to update its listening port dynamically.
This script available as a docker container automatically detects changes to the forwarded_port file created by [Gluetun](https://github.com/qdm12/gluetun/) and updates the Deluge's listening port. It also configures random port to false since we want to decide our own ports. This script uses curl to update the config and cat to look for changes in the forwarded_port file.

The original [docker image](https://github.com/secunit404/gluetun-deluge-port-manager) from [secunit404](https://github.com/secunit404) used [inotifywait](https://wiki.ubuntuusers.de/inotify/) instead of [cat](https://wiki.ubuntuusers.de/cat/) which brought up some reliability issues on my system.

## Setup
Add a mounted volume to [Gluetun](https://github.com/qdm12/gluetun/) (e.g. /yourfolder:/tmp/gluetun).

Finally, add the following snippet to your `docker-compose.yml`, substituting the default values for your own.

```yml
...

  gluetun-deluge-port-forwarding:
    image: mrstinktier/deluge-gluetun-port-forwarding
    container_name: gluetun-deluge-port-forwarding
    volumes:
      - /yourfolder:/gluetun #set "yourfolder" to the same directory you used for Gluetun
    network_mode: "service:gluetun"
    environment:
      DELUGE_SERVER: localhost      #change this to the IP of your deluge instance
      DELUGE_PORT: 8112             #change this to the PORT of your deluge instance
      DELUGE_PASS: YOURPASSWORD     #change this to your deluge password
      PORT_FORWARDED: /gluetun/forwarded_port
    restart: unless-stopped

...
```

## Examle Config

For [Gluetun](https://github.com/qdm12/gluetun/) configuration please refer to their [documentation](https://github.com/qdm12/gluetun-wiki/tree/main) and [setup guide](https://github.com/qdm12/gluetun/).

The only parts that are neccesary from the config below are the extra volume that points to a folder that stores the forwarded_port file in which the open port is written, the variable VPN_PORT_FORWARDING set to on and a vpn provider that supports port forwarding (You can find this out in Gluetuns [port forwarding documentation](https://github.com/qdm12/gluetun-wiki/blob/main/setup/advanced/vpn-port-forwarding.md)). 
```yml
services:
  gluetun:
    image: ghcr.io/qdm12/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    ports:
      - 8888:8888/tcp # HTTP proxy #Gluetun
      - 8388:8388/tcp # Shadowsocks #Gluetun
      - 8388:8388/udp # Shadowsocks #Gluetun
      - 8112:8112 # Deluge UI
    volumes:
      - /yourpath:/gluetun
      - /tmp/gluetun:/tmp/gluetun # This part is essential for the port forwarder to know what the open port is
    environment:
      # See https://github.com/qdm12/gluetun-wiki/tree/main/setup#setup
      - VPN_SERVICE_PROVIDER=ivpn
      - VPN_TYPE=openvpn
      # OpenVPN:
      - OPENVPN_USER=
      - OPENVPN_PASSWORD=
      # Wireguard:
      # - WIREGUARD_PRIVATE_KEY=wOEI9rqqbDwnN8/Bpp22sVz48T71vJ4fYmFWujulwUU=
      # - WIREGUARD_ADDRESSES=10.64.222.21/32
      # Timezone for accurate log times
      - TZ=
      # Server list updater
      # See https://github.com/qdm12/gluetun-wiki/blob/main/setup/servers.md#update-the-vpn-servers-list
      - UPDATER_PERIOD=
      # Activate Port forwarding
      # See https://github.com/qdm12/gluetun-wiki/blob/main/setup/advanced/vpn-port-forwarding.md
      - VPN_PORT_FORWARDING=on # This part is also essential because we would otherwise not have an open port that we can use.
  
  gluetun-deluge-port-forwarding:
    image: mrstinktier/deluge-gluetun-port-forwarding
    container_name: gluetun-deluge-port-forwarding
    volumes:
      - /tmp/gluetun:/gluetun #You can change this path to whatever path you want to use, as long as you also change it in the gluetun config.
    network_mode: "service:gluetun"
    environment:
      DELUGE_SERVER: localhost      #change this to the IP of your deluge instance
      DELUGE_PORT: 8112             #change this to the PORT of your deluge Web UI
      DELUGE_PASS: YOURPASSWORD     #change this to your deluge password
      PORT_FORWARDED: /gluetun/forwarded_port
    restart: unless-stopped
```
