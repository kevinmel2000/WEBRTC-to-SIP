# WEBRTC to SIP client and server
How to setup Kamailio + RTPEngine + TURN server to enable calling between WEBRTC client and legacy SIP clients. This setup will bridge SRTP --> RTP and ICE --> nonICE to make a WEBRTC client (SIPJs) be able to call legacy SIP clients.

This setup is for Debian 8 Jessie for all servers.

For the clients to avoid firewalls and to have the best setup, divide the servers like this:

1. Server - Kamailio + RTPEngine
2. Server - TURN
3. Server - WEBRTC client

The configuration is setup to try connecting with SIP with no modification. If the proxy receives a `488 Not Supported Here` from the other side, it will remove SRTP and ICE and try again. You can find an older commit that always translates before trying legacy SIP, this can be useful if you know that none of your legacy SIP clients support SRTP or ICE.

## Get certificates
For the certificates you need I recommend Let's Encrypt certificates. They will work for both Kamailio TLS and Nginx TLS. On the servers you need certificates, run the following (you must stop services running on port 443 during certificate request/renewal):
```bash
git clone https://github.com/letsencrypt/letsencrypt
cd letsencrypt
./letsencrypt certonly --standalone -d YOUR-DOMAIN
```
You will then find the certificates under:
```bash
/etc/letsencrypt/live/YOUR-DOMAIN/privkey.pem
/etc/letsencrypt/live/YOUR-DOMAIN/fullchain.pem
```

## Get configuration files
All files needed to setup all components on Debian 8 Jessie.
```bash
git clone https://github.com/havfo/WEBRTC-to-SIP.git
cd WEBRTC-to-SIP
find . -type f -print0 | xargs -0 sed -i 's/XXXXX-XXXXX/PUT-IP-OF-YOUR-SIP-SERVER-HERE/g'
find . -type f -print0 | xargs -0 sed -i 's/XXXX-XXXX/PUT-DOMAIN-OF-YOUR-SIP-SERVER-HERE/g'
find . -type f -print0 | xargs -0 sed -i 's/XXX-XXX/PUT-DOMAIN-OF-YOUR-TURN-SERVER-HERE/g'
```

## Install RTPEngine
This will do the SRTP-RTP bridging needed to make WEBRTC clients talk to legacy SIP server/clients.
```bash
apt-get install dpkg-dev debhelper iptables-dev libcurl4-openssl-dev libglib2.0-dev libhiredis-dev libpcre3-dev libssl-dev markdown zlib1g-dev libxmlrpc-core-c3-dev dkms linux-headers-`uname -r`
git clone https://github.com/sipwise/rtpengine.git
cd rtpengine
./debian/flavors/no_ngcp
dpkg-buildpackage
cd ..
dpkg -i 'ngcp-rtpengine-daemon_*' 'ngcp-rtpengine-iptables_*' 'ngcp-rtpengine-kernel-dkms_*'
cd WEBRTC-to-SIP
cp etc/default/ngcp-rtpengine-daemon /etc/default/
/etc/init.d/ngcp-rtpengine-daemon restart
```

## Install IPTables firewall (optional)
This is optional. If this is installed it will persist after reboot. You can run the iptables.sh at any time after it is set up.
```bash
cd WEBRTC-to-SIP
chmod +x iptables.sh
cp etc/network/if-up.d/iptables /etc/network/if-up.d/
chmod +x /etc/network/if-up.d/iptables
touch /etc/iptables/firewall.conf
touch /etc/iptables/firewall6.conf
./iptables.sh
```

## Install Kamailio
```bash
apt-get install kamailio kamailio-websocket-modules kamailio-mysql-modules kamailio-tls-modules kamailio-presence-modules mysql-server
cd WEBRTC-to-SIP
cp etc/kamailio/* /etc/kamailio/
kamdbctl create
```
Select yes (Y) to all options.

```bash
kamctl add websip websip
/etc/init.d/kamailio restart
```

## Install WEBRTC client
```sh
apt-get install nginx
cd WEBRTC-to-SIP
cp etc/nginx/sites-available/default /etc/nginx/sites-available/
cp -r client/* /var/www/html/
```

## Install TURN server
```sh
apt-get install coturn
cp etc/default/coturn /etc/default/
cp etc/turn* /etc/
/etc/init.d/coturn restart
```

## Testing
You should now be able to go to https://webrtcnginxserver/ and call to legacy SIP clients.
