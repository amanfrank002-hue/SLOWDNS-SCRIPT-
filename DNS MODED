#!/bin/bash
# ==================================================
# MR HACKER DNSTT PRO - ONE CLICK INSTALLER
# Smart • Stable • Auto Proxy • Auto Firewall
# ==================================================
set -euo pipefail

clear
echo "=============================================="
echo "        MR HACKER DNSTT PRO INSTALLER"
echo "=============================================="

# ROOT CHECK
if [ "$(id -u)" -ne 0 ]; then
  echo "Run as root: sudo bash mr-hacker-dnstt-pro.sh"
  exit 1
fi

# INPUT DOMAIN
read -rp "Enter DNS Tunnel Domain (example: ns.domain.com): " TDOMAIN
[ -z "$TDOMAIN" ] && { echo "Domain cannot be empty."; exit 1; }

MTU=1300
INTERNAL_PORT=5300

echo "Domain : $TDOMAIN"
echo "MTU    : $MTU"
sleep 1

echo "Stopping old services..."
for svc in dnstt dnstt-server slowdns dnstt-mr-hacker dnstt-mr-hacker-proxy; do
  systemctl disable --now $svc 2>/dev/null || true
done

# FREE PORT 53
if [ -f /etc/systemd/resolved.conf ]; then
  sed -i 's/^#DNSStubListener=.*/DNSStubListener=no/' /etc/systemd/resolved.conf || true
  sed -i 's/^DNSStubListener=.*/DNSStubListener=no/' /etc/systemd/resolved.conf || true
  systemctl restart systemd-resolved
  ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
fi

# INSTALL DEPENDENCIES
apt update -y
apt install -y curl python3

# DOWNLOAD DNSTT
mkdir -p /usr/local/bin
curl -L https://dnstt.network/dnstt-server-linux-amd64 \
  -o /usr/local/bin/dnstt-server
chmod +x /usr/local/bin/dnstt-server

# GENERATE KEYS
mkdir -p /etc/dnstt
if [ ! -f /etc/dnstt/server.key ]; then
  /usr/local/bin/dnstt-server -gen-key \
    -privkey-file /etc/dnstt/server.key \
    -pubkey-file /etc/dnstt/server.pub
fi
chmod 600 /etc/dnstt/server.key

# CREATE DNSTT SERVICE (INTERNAL 5300)
cat >/etc/systemd/system/dnstt-mr-hacker.service <<EOF
[Unit]
Description=MR HACKER DNSTT Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/dnstt-server -udp :${INTERNAL_PORT} -mtu ${MTU} -privkey-file /etc/dnstt/server.key ${TDOMAIN} 127.0.0.1:22
Restart=always
RestartSec=3
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF

# CREATE EDNS PROXY
cat >/usr/local/bin/dnstt-mr-hacker-proxy.py <<'PYEOF'
#!/usr/bin/env python3
import socket, threading, struct

LISTEN=("0.0.0.0",53)
UPSTREAM=("127.0.0.1",5300)
EXT=512
INT=1800

def patch(data,size):
    if len(data)<12: return data
    try:
        q,a,n,ar=struct.unpack("!HHHH",data[4:12])
    except: return data
    off=12
    def skip(buf,o):
        while True:
            if o>=len(buf): return len(buf)
            l=buf[o]; o+=1
            if l==0: break
            if l&0xC0==0xC0: o+=1; break
            o+=l
        return o
    for _ in range(q):
        off=skip(data,off)+4
    for _ in range(a+n):
        off=skip(data,off)
        if off+10>len(data): return data
        rd=struct.unpack("!H",data[off+8:off+10])[0]
        off+=10+rd
    new=bytearray(data)
    for _ in range(ar):
        off=skip(data,off)
        if off+10>len(data): return data
        t=struct.unpack("!H",data[off:off+2])[0]
        if t==41:
            new[off+2:off+4]=struct.pack("!H",size)
            return bytes(new)
        rd=struct.unpack("!H",data[off+8:off+10])[0]
        off+=10+rd
    return data

def handle(s,data,addr):
    u=socket.socket(socket.AF_INET,socket.SOCK_DGRAM)
    u.settimeout(5)
    try:
        u.sendto(patch(data,INT),UPSTREAM)
        r,_=u.recvfrom(4096)
        s.sendto(patch(r,EXT),addr)
    except: pass
    finally: u.close()

def main():
    s=socket.socket(socket.AF_INET,socket.SOCK_DGRAM)
    s.bind(LISTEN)
    while True:
        d,a=s.recvfrom(4096)
        threading.Thread(target=handle,args=(s,d,a),daemon=True).start()

if __name__=="__main__":
    main()
PYEOF

chmod +x /usr/local/bin/dnstt-mr-hacker-proxy.py

# PROXY SERVICE
cat >/etc/systemd/system/dnstt-mr-hacker-proxy.service <<EOF
[Unit]
Description=MR HACKER DNSTT EDNS Proxy
After=network-online.target dnstt-mr-hacker.service
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/dnstt-mr-hacker-proxy.py
Restart=always
RestartSec=1
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF

# FIREWALL AUTO DETECT
if command -v ufw >/dev/null 2>&1; then
  ufw allow 22/tcp || true
  ufw allow 53/udp || true
elif command -v iptables >/dev/null 2>&1; then
  iptables -I INPUT -p udp --dport 53 -j ACCEPT || true
  iptables -I INPUT -p tcp --dport 22 -j ACCEPT || true
fi

# START SERVICES
systemctl daemon-reload
systemctl enable --now dnstt-mr-hacker.service
systemctl enable --now dnstt-mr-hacker-proxy.service

IP=$(hostname -I | awk '{print $1}')

echo
echo "=============================================="
echo "      MR HACKER DNSTT PRO INSTALLED"
echo "=============================================="
echo "Server IP     : $IP"
echo "Tunnel Domain : $TDOMAIN"
echo "Internal Port : 5300"
echo "Public Port   : 53"
echo
echo "PUBLIC KEY:"
cat /etc/dnstt/server.pub
echo
echo "Service Status:"
systemctl status dnstt-mr-hacker.service --no-pager | head -n 5
systemctl status dnstt-mr-hacker-proxy.service --no-pager | head -n 5
echo "=============================================="
