Purpose: 從遠端連進 LAB 的 ClearPass
# 最終拓樸
```
[ LAB LAN ]
    |
 IPC01 (Host)
    |
  br0  (eno1, eno2, enp3s0)
    |
  VM01 (ZeroTier Bridge)
    |
  br-lab (enp1s0 + zt)
========== INTERNET ==========
  br-remote (enp1s0 + zt)
    |
  VM02 (ZeroTier Bridge)
    |
 IPC02 (Host)
    |
  br0
    |
[ Remote LAN / AP / Router ]
```


## Phase 1 IPC01（LAB side）Host 建置
### Step 1 — 建立 bridge（br0）
方法1:  用CLI 
```sh

nmcli connection add type bridge ifname br0
# 不要把WAN Port放進去
nmcli connection add type ethernet ifname eno1 master br0
nmcli connection add type ethernet ifname eno2 master br0
nmcli connection add type ethernet ifname enp3s0 master br0
```

然後設定 IP 在 br0（不是實體 NIC）
```
nmcli connection modify br0 ipv4.addresses 192.168.1.195/24
nmcli connection modify br0 ipv4.gateway 192.168.1.1
nmcli connection modify br0 ipv4.method manual
```


### Step 2 — 驗證
```sh
brctl show
# 要看到
br0 → eno1, eno2, enp3s0
```

##  Phase 2：IPC01 VM（Zerotier node Lab）
### Step 1 — VM 網卡配置
- enp1s0   -> 接 IPC br0（LAN）
- enp7s0   -> NAT（管理用）

### Step 2 — 安裝 ZeroTier 與授權
```sh
curl -s https://install.zerotier.com | sudo bash
zerotier-cli join <network_id>
```

ZeroTier 設定
- Network Details:  
	- 關閉 Auto-assign IPv4s
	- 關閉任何 assign
	- Enable Broadcast
- Device Details:
	- Authed
	- Do not assign ips
	- Allow Ethernet Bridging

### Step 3 — 建立 bridge（核心）
需要先確定要用哪一個 Bridge ITF name (br-remote/ br-lab)
```sh
nmcli connection add type bridge ifname <BR_ITF> con-name <BR_ITF>
nmcli connection add type bridge-slave ifname enp1s0 master <BR_ITF> con-name bridge-slave-enp1s0
nmcli connection modify <BR_ITF> ipv4.method disabled
nmcli connection modify <BR_ITF> ipv6.method disabled
nmcli connection modify <BR_ITF> bridge.stp no
nmcli connection up <BR_ITF>
nmcli connection up bridge-slave-enp1s0
```

建立一個 開機服務，將ZT網卡納至bridge
```
nano /etc/systemd/system/br-zt-add.service
```
放置以下內容:
```text
[Unit]
Description=Add Zerotier interface to br-remote
After=network-online.target zerotier-one.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/br-zt-add.sh

[Install]
WantedBy=multi-user.target
```
設置sh:
```
nano /usr/local/sbin/br-zt-add.sh
```
sh放置以下內容:   Note: 要先確定資料正確
```
#!/bin/bash
ZT_IF="ztxooenzhk"
BR_IF="br-remote"

while ! ip link show "$ZT_IF" >/dev/null 2>&1; do
    sleep 1
done

ip link set "$ZT_IF" up

if ! bridge link show | grep -q "$ZT_IF"; then
    ip link set "$ZT_IF" master "$BR_IF"
fi
```
進行啟動
```
sudo chmod +x /usr/local/sbin/br-zt-add.sh
sudo systemctl daemon-reload
sudo systemctl enable br-zt-add.service
sudo systemctl start br-zt-add.service
```

檢查:
brctl show 
應該要包含ZT網卡跟enp1s0

### Step 4 — 確定 ZT 網卡沒有 IP
```sh
# 如果有就清掉
ip addr flush dev ztXXXX
```

### Step 5 — 永久關掉 bridge filter
```sh
modprobe br_netfilter
echo br_netfilter > /etc/modules-load.d/br_netfilter.conf
touch /etc/sysctl.d/99-bridge.conf
```
放置以下內容
```
net.bridge.bridge-nf-call-iptables=0
net.bridge.bridge-nf-call-ip6tables=0
net.bridge.bridge-nf-call-arptables=0
```

```sh
sysctl --system
```
驗證:
```sh
# 要有東西
lsmod | grep br_netfilter
# 要等於0
sysctl net.bridge.bridge-nf-call-iptables
```

### Step 6  永久開 promisc 
```sh
nano /etc/systemd/system/promisc.service
```
放置以下內容:
```
[Unit]
Description=Enable Promiscuous Mode
After=network-online.target zerotier-one.service br-zt-add.service
Wants=network-online.target
Requires=br-zt-add.service

[Service]
Type=oneshot
ExecStart=/bin/bash /usr/local/sbin/promisc.sh
RemainAfterExit=yes
TimeoutStartSec=40

[Install]
WantedBy=multi-user.target
```
設置sh
```
nano /usr/local/sbin/promisc.sh
```
sh放置以下內容:   Note: 要先確定資料正確
```
#!/bin/bash
set -euxo pipefail

BR="br-lab"
PHY="enp1s0"
ZT="ztxooenzhk"

echo "[1] start promisc.sh"

echo "[2] waiting for bridge $BR"
for i in $(seq 1 30); do
    if ip link show "$BR" >/dev/null 2>&1; then
        echo "[2] bridge $BR found"
        break
    fi
    sleep 1
done

ip link show "$BR" >/dev/null 2>&1

echo "[3] waiting for zt $ZT"
for i in $(seq 1 30); do
    if ip link show "$ZT" >/dev/null 2>&1; then
        echo "[3] zt $ZT found"
        break
    fi
    sleep 1
done

ip link show "$ZT" >/dev/null 2>&1

echo "[4] set $BR promisc on"
ip link set "$BR" promisc on

echo "[5] set $PHY promisc on"
ip link set "$PHY" promisc on

echo "[6] set $ZT promisc on"
ip link set "$ZT" promisc on

echo "[7] done"
```

PS: ZT網卡記得要改，並且IPC1是要用br-lab；IPC2用br-remote
```sh
chmod +x /usr/local/sbin/promisc.sh
systemctl daemon-reexec
systemctl enable promisc
systemctl start promisc
```

### Step 7  永久關閉 ip_forward (VM + IPC)
```sh
echo "net.ipv4.ip_forward=0" >> /etc/sysctl.conf
sysctl -p
```


# 實地路由結構
```
          (bridge network)
       ┌───────────────────┐
       │     br-lab        │
       │  enp1s0 + ztXXXX  │
       └───────────────────┘
                 │
                 │ (Ethernet frame)
                 ▼
          ztXXXX (虛擬 NIC)
                 │
                 ▼
         zerotier-one daemon
                 │
         (封裝成 UDP packet)
                 │
                 ▼
          enp7s0 (真實 NIC)
                 │
                 ▼
              Internet
```
