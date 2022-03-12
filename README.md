## CAUTION
* do **NOT** checkout this repo to `/`, read and edit configurations accordingly.
* this is more like a note for experienced administrators, not for a newbie to blindly follow, do **NOT** follow if you don't know what you're doing.
* absolutely no warranty.

## Overview
* our main router is likely a "home router" which also acts as switch and wireless AP.
* side router is just another (specially configured) computer **in our network**.
	* in this tutorial, it runs debian.
	* could be a raspberry pi.
	* could be a virtual machine.
* side router runs the VPN connection.
	* in this tutorial, we use wireguard.
		* preferably accompanied with udp2raw or phantun
* our default gateway will be the side router.
* when traffic reach side router, it will be routed to main router or VPN accordingly.
* and a special DNS configuration is required.
	* obviously we can't use DNS from ISP.
	* use DNS from VPN will not be optimal.
* tested on bullseye.
* the name is a tribute to Tor.

## Configuration
* if not specified, all configurations are done on side router by root.
* assumes:
	* main router `192.168.0.1/24`
	* side router `192.168.0.2/24`
		* wireguard `10.0.0.2/24`
	* VPN server `123.123.123.123`
		* wireguard `10.0.0.1/24`
* use static IP:
	* an example configuration in `/etc/network/interfaces.d/eth.conf`.
	* edit original `/etc/network/interfaces` accordingly.
* most likely `/etc/resolv.conf` still points to `192.168.0.1`, leave it.
* enable forwarding:
	* `/etc/sysctl.d/net.conf`
	* `sysctl -p /etc/sysctl.d/net.conf`
* add a new route table:
	* `echo "1000 potato" >> /etc/iproute2/rt_tables`
* wireguard:
	* wireguard: `/etc/wireguard/wg0.conf`.
	* wireguard interface: `/etc/network/interfaces.d/wg0.conf`.
		* make sure `/etc/network/interfaces` contains `source /etc/network/interfaces.d/*`.
			* debian default.
	* `ifup wg0`
	* configuration for the VPN server is not covered here.
		* almost identical sans lines for routing in interface config.
	* configuration for udp2raw/phantun is not covered here.
		* https://github.com/wangyu-/udp2raw
		* https://github.com/dndx/phantun
* IP list:
	* `/usr/local/bin/update-china-lst`
		```
		apt install git
		chmod a+x /usr/local/bin/update-china-lst
		mkdir /var/lib/potato
		cd /var/lib/potato
		git clone --depth 1 https://github.com/misakaio/chnroutes2
		update-china-lst
		```
	* `/var/lib/potato/nft-set-china.conf` should be populated now.
	* `ln -s /usr/local/bin/update-china-lst /etc/cron.daily/`
* nftables:
	* `/etc/nftables.conf`
	* `systemctl reload nftables;systemctl status nftables`
	* CAUTION: this is very bare bones, no firewall.
* DNS:
	* check if anything else (like systemd-resolved) is listening on :53 and stop/disable them:
		* `ss -uanp|grep :53`
		* `systemctl disable --now systemd-resolved`
	* diverge:
		* https://github.com/Jimmy-Z/go/tree/master/diverge
			* not open source as of now.
		* requires redis:
			* `apt install redis`
			* uncomment these lines in `/etc/redis/redis.conf`:
				```
				unixsocket /var/run/redis/redis-server.sock
				unixsocketperm 700
				```
			* `systemctl restart redis`
		* `/etc/systemd/system/diverge.service`
		* `/etc/diverge.env`
		* `systemctl enable --now diverge;systemctl status diverge`
	* AdGuard Home:
		* https://github.com/AdguardTeam/AdGuardHome/releases
			```
			mkdir /opt
			cd /opt
			tar -xvf ~/AdGuardHome_linux_amd64.tar.gz
			cd AdGuardHome
			./AdGuardHome -s install
			```
		* complete the initial setup on http://192.168.0.2/
		* configure, Settings -> DNS settings:
			* Upstream DNS servers:
				```
				# dnsmasq for local resolve
				[//]127.0.0.1:1053
				[/lan/]127.0.0.1:1053
				# diverge
				127.0.0.1:1054
				```
			* Bootstrap DNS servers:
				```
				1.1.1.1
				1.0.0.1
				```
			* Private reverse DNS servers:
				```
				127.0.0.1:1053
				```
			* Apply
* we can now test if routing/DNS are working:
	* on a test host, manually set gateway and DNS to `192.168.0.2`
	* https://www.ipip.net should show we're connected directly
	* https://ipaddress.com should show we're connected via VPN
* DHCP server:
	* `apt install dnsmasq`
	* `/etc/dnsmasq.conf`
	* (important) stop DHCP on the main router.
	* `systemctl restart dnsmasq;systemctl status dnsmasq`
* test if DHCP is working:
	* on that test host, set it back to DHCP.
	* on other hosts, disable/enable the adapter/Wi-Fi.
