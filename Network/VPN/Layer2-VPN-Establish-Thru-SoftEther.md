# SoftEther Layer2 VPN Buildup
## Softether Server
Basic Server Install refer: [official guide](https://www.softether.org/4-docs/1-manual/7/7.3)
Server Install pack: [download(v4.44)](https://github.com/SoftEtherVPN/SoftEtherVPN_Stable/releases/download/v4.44-9807-rtm/softether-vpnserver-v4.44-9807-rtm-2025.04.16-linux-x64-64bit.tar.gz)

###  Install-Steps:
1. Use check_env_softether.sh (appendix.sh.1) 
2. Download Install Pack and Extract
3. Use ` sudo ./.install.sh`
4. Move Directory to /usr , and setup 
	```sh
	cd ..
	mv vpnserver /usr/local
	cd /usr/local/vpnserver/
	chmod 600 *
	chmod 700 vpncmd
	chmod 700 vpnserver
	```
5. Using Check Command
	```sh
	/usr/local/vpnserver/vpncmd
	# give 3 to Tools 
	# then:
	VPN Tools>check
	``` 
6. Register a starup script
	```sh
	nano /opt/vpnbridge.sh	
	```
	Given text to vpnbridge sh
	```text
	#!/bin/sh
	# chkconfig: 2345 99 01
	# description: SoftEther VPN Server
	DAEMON=/usr/local/vpnserver/vpnserver
	LOCK=/var/lock/subsys/vpnserver
	test -x $DAEMON || exit 0
	case "$1" in
	start)
	$DAEMON start
	touch $LOCK
	;;
	stop)
	$DAEMON stop
	rm $LOCK
	;;
	restart)
	$DAEMON stop
	sleep 3
	$DAEMON start
	;;
	*)
	echo "Usage: $0 {start|stop|restart}"
	exit 1
	esac
	exit 0
	```
	```sh
	chmod 755 /opt/vpnserver.sh
	nano /etc/systemd/system/vpnserver.service
	```
	Given Text to vpnserver.service
	```text
	[Unit]
	Description = vpnserver daemon

	[Service]
	ExecStart = /opt/vpnserver.sh start
	ExecStop = /opt/vpnserver.sh stop
	ExecReload = /opt/vpnserver.sh restart
	Restart = always
	Type = forking

	[Install]
	WantedBy = multi-user.target
	```
	Enable and Check
	```sh
	systemctl enable vpnserver
	systemctl start vpnserver
	systemctl status vpnserver --no-pager
	```
	
###  Config-Steps:
#### Config Server Thru Server Manager
1. Prepare Server Manager ( Windows/ MacOS only)
	Download Click: [official guide](https://www.softether-download.com/en.aspx?product=softether)
2. Open Firewall on VPC for TCP port: 443, 992, 1194, 5555
	*Restrict IP later for safer usages
3. Setup Password    
4. Server Manager > Check:
	- Remote Access VPN Server
	- Site-to-site VPN Server or VPN Bridge
		- VPN Server that Accepts Connection from Other Sites (Center) 
4. Creating a Virtual Hub
	- clearpass
5. Skip most of the things
	- Dynamic DNS
	- IPsec ...etc 
	- Azure
6. Create User/ Group on Server
7. See If need a "Local Bridge"

## Softether Bridge
Basic Bridge Install refer: [official guide](https://www.softether.org/4-docs/1-manual/9/9.3)
Bridge Install pack: [download(v4.44)](https://github.com/SoftEtherVPN/SoftEtherVPN_Stable/releases/download/v4.44-9807-rtm/softether-vpnbridge-v4.44-9807-rtm-2025.04.16-linux-x64-64bit.tar.gz)

###  Install-Steps:
1. Establish "br0" as target interface
2. Use check_env_softether.sh (appendix.sh.1) 
3. Download Install Pack and Extract
4. Use ` sudo ./.install.sh`
5. Move Directory to /usr , and setup 
	```sh
	cd ..
	mv vpnbridge /usr/local
	cd /usr/local/vpnbridge/
	chmod 600 *
	chmod 700 vpncmd
	chmod 700 vpnbridge
	```
6. Using Check Command
	```sh
	/usr/local/vpnbridge/vpncmd
	# give 3 to Tools 
	# then:
	VPN Tools>check
	```
7. Register a starup script
	```sh
	nano /opt/vpnbridge.sh	
	```
	Given text:
	```
	#!/bin/sh  
	# description: SoftEther VPN Server  
	DAEMON=/usr/local/vpnbridge/vpnbridge  
	LOCK=/var/lock/subsys/vpnbridge  
	test -x $DAEMON || exit 0  
	case "$1" in  
	start)  
	$DAEMON start  
	touch $LOCK  
	;;  
	stop)  
	$DAEMON stop  
	rm $LOCK  
	;;  
	restart)  
	$DAEMON stop  
	sleep 3  
	$DAEMON start  
	;;  
	*)  
	echo "Usage: $0 {start|stop|restart}"  
	exit 1  
	esac  
	exit 0
	```
	```
	chmod 755 /opt/vpnbridge.sh
	nano /etc/systemd/system/vpnbridge.service
	```
	Given text:
	```text
	[Unit]
	Description = vpnbridge daemon

	[Service]
	ExecStart = /opt/vpnbridge.sh start
	ExecStop = /opt/vpnbridge.sh stop
	ExecReload = /opt/vpnbridge.sh restart
	Restart = always
	Type = forking

	[Install]
	WantedBy = multi-user.target
	```
	```sh
	systemctl enable vpnbridge 
	systemctl start vpnbridge 
	systemctl status vpnbridge --no-pager
	```


### Config-Steps
1. Establish a cascade link to server
	```sh
	/usr/local/vpnbridge/vpncmd
	# 1 > localhost > (Press Enter skip) > keyin Password(if)
	VPN Server>Hub Bridge
	VPN Server /Bridge> CascadeCreate ToServer
	# Enter <Server IP>:<Port> // <Softether-Server>:443
	# Enter Hub Name:  clearpass
	# Enter user Name: dniuser/ ty3user/ dsonuser
	```  
2. Set Password for Cascade
	```
	VPN Server /Bridge> CascadePasswordSet ToServer
	# Typing Password  //12345678
	# Set Standard
	```
3. Activate Cascade
	```
	VPN Server /Bridge> CascadeOnline ToServer
	# Check Stauts:
	VPN Server /Bridge> CascadeStatusGet ToServer
	``` 
4. Establish Tap NIC
	```sh
	/usr/local/vpnbridge/vpncmd
	# 1 > localhost > (Press Enter skip) > keyin Password(if)
	VPN Server> BridgeCreate BRIDGE /Device:br0 /TAP:yes
	# Verify
	VPN Server> BridgeList
	```
5. Add tap_br0 into br0
	```
	# One Time Deal:
	sudo brctl addif br0 tap_br0
	# Check
	brctl show
	# Startup Process:
	nano /etc/systemd/system/vpnbridge.service
	```
	Given text to vpnbridge.service
	```text
	[Unit]
	Description = vpnbridge daemon

	[Service]
	ExecStart = /opt/vpnbridge.sh start
	ExecStop = /opt/vpnbridge.sh stop
	ExecReload = /opt/vpnbridge.sh restart
	ExecStartPost = /bin/bash -c 'sleep 10 && brctl addif br0 tap_br0 && ip link set br0 promisc on'
	Restart = always
	Type = forking

	[Install]
	WantedBy = multi-user.target
	```
	Refresh Daemon
	```
	sudo systemctl daemon-reload
	sudo systemctl restart vpnbridge
	```

6. Set br0 Promisc
	```
	# One Time Deal
	sudo ip link set br0 promisc on
	```

7. Change Server/ Virtual Hub ( If needed )
	```sh
	/usr/local/vpnbridge/vpncmd
	# 1 > localhost > (Press Enter skip) > keyin Password(if)
	VPN Server>hub bridge
	VPN Server /Bridge>cascadelist 
	# pick <OnDeleteCascade> 
	VPN Server /Bridge>cascadeoffline <OnDeleteCascade>
	VPN Server /Bridge>cascadedelete <OnDeleteCascade>
	# back to Step1:  CascadeCreate ToServer
	```

# Appendix
## sh script
### 1. check_env_softether.sh
```sh
#!/bin/bash

echo "=== SoftEther Prerequisite Check ==="
echo

missing=0

check_cmd() {
    if command -v "$1" >/dev/null 2>&1; then
        echo "[OK] Command: $1"
    else
        echo "[MISSING] Command: $1"
        missing=1
    fi
}

check_pkg() {
    if dpkg -s "$1" >/dev/null 2>&1; then
        echo "[OK] Package: $1"
    else
        echo "[MISSING] Package: $1"
        missing=1
    fi
}

check_lib() {
    if ldconfig -p | grep -q "$1"; then
        echo "[OK] Library: $1"
    else
        echo "[MISSING] Library: $1"
        missing=1
    fi
}

echo "--- Checking commands ---"
check_cmd gcc
check_cmd tar
check_cmd gzip
check_cmd cat
check_cmd cp

echo
echo "--- Checking packages ---"
check_pkg build-essential
check_pkg binutils
check_pkg zlib1g-dev
check_pkg libssl-dev
check_pkg libreadline-dev
check_pkg libncurses-dev

echo
echo "--- Checking libraries ---"
check_lib libc.so
check_lib libz.so
check_lib libssl.so
check_lib libreadline.so
check_lib libncurses.so
check_lib libpthread.so

echo
echo "--- Checking UTF-8 locale ---"
if locale | grep -q "UTF-8"; then
    echo "[OK] UTF-8 locale enabled"
else
    echo "[WARNING] UTF-8 locale not detected"
fi

echo
echo "--- Checking systemctl (instead of chkconfig) ---"
check_cmd systemctl

echo
echo "=== Result ==="
if [ $missing -eq 0 ]; then
    echo "All required components are installed ✅"
else
    echo "Some components are missing ❌"
    echo "Install them with:"
    echo "sudo apt update && sudo apt install build-essential libssl-dev zlib1g-dev libreadline-dev libncurses-dev -y"
fi
```

