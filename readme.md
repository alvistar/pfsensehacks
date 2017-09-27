# PFSENSE HACKS

This is a small collection of tips I had to use for an internal project and I wish to share with you and "future me".

## Installing FREEBSD packages
Instructions are at https://doc.pfsense.org/index.php/Installing_FreeBSD_Packages

## Installing TMUX
```Shell
pkg add http://pkg.freebsd.org/freebsd:10:x86:64/latest/All/tmux-2.5_1.txz
```

Unfortunately this is not enough:

> [2.3.4-RELEASE][root@pfSense.localdomain]/root: /usr/local/bin/tmux
>
> tmux: need UTF-8 locale (LC_CTYPE) but have US-ASCII

Unfortunately locales are missing from pfsense. Let's fix this

```Shell
curl -JLO http://ftp.freebsd.org/pub/FreeBSD/releases/amd64/10.3-RELEASE/base.txz
mkdir base
cd base
tar xvf ../base.txz
cp -r usr/share/locale /usr/share
```

## Installing FISH shell
If you want also a fancy a shell to interact with

```Shell
pkg add http://pkg.freebsd.org/freebsd:10:x86:64/latest/All/pcre2-10.21_1.txz
pkg add http://pkg.freebsd.org/freebsd:10:x86:64/latest/All/readline-7.0.3.txz
pkg add http://pkg.freebsd.org/freebsd:10:x86:64/latest/All/python36-3.6.2_1.txz
pkg add http://pkg.freebsd.org/freebsd:10:x86:64/latest/All/python3-3_3.txz
pkg add http://pkg.freebsd.org/freebsd:10:x86:64/latest/All/fish-2.6.0.txz
```

Fish should be **ALMOST** working now. Fish is using man pages for useful autocompletion, but man pages are missing in pfsense.

See next section for install MAN pages

## Installing MAN pages

Go to base directory as in "Installing TMUX"

```Shell
cd usr/bin
cp man groff tbl troff grotty apropos /usr/bin

cd ../share
cp -r man groff_font tmac /usr/share
```

## Executing a custom script for Openvpn client when connection is established
Since PFSense is already using "up" script you can think to use "route-up".

Just setup the custom options in Advanced Configuration on Openvpn client
```
route-up /root/yourscript.sh
```

## Ipv6 DHCP flooding

This is due to a bug in WIDE DHCPv6. I experienced myself problems in hosting on online.net.

See [DHCPV6 Flooding](https://hauweele.net/~gawen/blog/?tag=dhcp6c)

Here's how I fix it on PFSense:

First make sure to set WAN interface to a static address.

Add isc-dhcp43 package. It will complain it does already exist, force it.

```
pkg add -f http://pkg.freebsd.org/freebsd:10:x86:64/latest/All/isc-dhcp43-client-4.3.6.txz
```

Create /usr/local/etc/dhclient6.conf

```
interface "igb0" {
  send dhcp6.client-id DUID;
}
```

Now we need to run some commands at boot time. I used shellcmd package for this.

First enable igb0 to accept router advertisement and run rtsold daemon to send solicit messages.

```
/sbin/ifconfig igb0 inet6 accept_rtadv; /usr/sbin/rtsold igb0
```

Second start dhclient

```
/usr/local/sbin/dhclient -cf /usr/local/etc/dhclient6.conf -P igb0
```

Order should not matter

## Dynamically change IPV6 address of an interface after Openvpn connect
This was my scenario
- Openvpn Client connect to server and get an IPV6/64 address
- Another IPV6/64 is provided for internal lan and I need to route advertise it

Here's how I setup it.

- From PFsense web interface
  * Set a static IPV6 address to LAN interface. This is not important.
  * Go to **Services/DHCPv6 Server & RA/LAN/Router Advertisements** and enable **Router Mode** to **Managed**

Now for the script, I used Python and [Pfsense Faux Api](https://github.com/ndejong/pfsense_fauxapi)

Script
```python
#!/usr/local/bin/python3
import sys, json, logging,os
from python.fauxapi_lib import FauxapiLib


logging.basicConfig(level=logging.DEBUG)
newip = os.environ.get('OPENVPN_RADVD')
#example OPENVPN_RADVD = '2001:bc8:aaaa:111::1'
logging.info(newip)

fauxapi_host="127.0.0.1"
fauxapi_apikey="YOUR_API_KEY"
fauxapi_apisecret="YOUR_API_SECRET"
FauxapiLib = FauxapiLib(fauxapi_host, fauxapi_apikey, fauxapi_apisecret, debug=False)


# config get
config = FauxapiLib.config_get()
config = FauxapiLib.config_get('interfaces')
print(json.dumps(config))
config['lan']['ipaddrv6']=newip
FauxapiLib.config_set(config,'interfaces')
#This need to be enabled in /etc/fauxapi/pfsense_function_calls.txt
FauxapiLib.function_call({
         'function': 'interface_configure',
         'includes': ['interfaces.inc'],
         'args':['lan']
     }
)
```

And here are Openvpn server conf files:
```openvpn
#server.conf
port 443
proto tcp
;proto udp
dev tun
# Windows needs the TAP-Win32 adapter name
# from the Network Connections panel if you
# have more than one.
# Non-Windows systems usually don't need this.
;dev-node MyTap

ca ca.crt
cert gateway.crt
key gateway.key
dh dh.pem
cipher AES-256-CBC

ifconfig-pool-persist ipp.txt
client-config-dir ccd

# addresses from .100 to .200 are reserved for VPN clients.
# Obviously, they should not be used by real LAN clients.
--server 10.8.0.0 255.255.255.0
server-ipv6 2001:bc8:aaaa:100::/64
route-ipv6 2001:bc8:aaaa:100::/57

push "route-ipv6 2000::/3"

verb 3
```

```openvpn
#ccd/client2
iroute-ipv6 2001:bc8:aaaa:108::/64
push "setenv-safe RADVD 2001:bc8:aaaa:108::1"
```


