### nomor 2
```
sed -i -E 's/tengen\)( +)IP=\$P\.4\.2 /tengen)\1IP=$P.4.10 /; s/inumaki\)( +)IP=\$P\.4\.10
/inumaki)\1IP=$P.4.2 /' /root/soal1.sh
ip addr flush dev eth0
bash /root/soal1.sh
cp /root/soal1.sh /etc/network/soal1.sh
```

### nomor 3
```
cat > /root/soal3.sh << 'EOF'
#!/bin/bash
# Soal 3 - jichi: DHCP Relay ke tengen
echo "nameserver 192.168.122.1" > /etc/resolv.conf
export DEBIAN_FRONTEND=noninteractive
command -v dhcrelay >/dev/null || (apt-get update && apt-get install -y isc-dhcp-relay)
cat > /etc/default/isc-dhcp-relay << EOT
SERVERS="192.227.4.10"
INTERFACES=""
OPTIONS="-id eth2 -id eth3 -iu eth4"
EOT
pkill dhcrelay 2>/dev/null; sleep 1
# -id = interface ke klien (Network 2 & 3), -iu = interface ke DHCP server (Network 4)
dhcrelay -4 -id eth2 -id eth3 -iu eth4 192.227.4.10
echo "===== Konfigurasi relay ====="
cat /etc/default/isc-dhcp-relay
echo "===== Proses relay ====="
ps aux | grep [d]hcrelay
echo "===== Cek jalur ke tengen ====="
ping -c 2 192.227.4.10
EOF
bash /root/soal1.sh ; bash /root/soal3.sh
cp /root/soal3.sh /etc/network/soal3.sh
```

### nomor 4
```
cat > /root/soal4.sh << 'EOF'
#!/bin/bash
# Soal 4 - tengen: 2 pool DHCP di Network 2 berdasarkan MAC (class)
P=192.227
echo "nameserver 192.168.122.1" > /etc/resolv.conf
command -v dhcpd >/dev/null || (apt-get update && apt-get install -y isc-dhcp-server)
echo 'INTERFACESv4="eth0"' > /etc/default/isc-dhcp-server
cat > /etc/dhcp/dhcpd.conf << EOT
authoritative;
default-lease-time 600;
max-lease-time 7200;
# ===== Class berdasarkan MAC address =====
# Pool 1 - Tim Dev (itadori, fushiguro)
class "pool1" { match hardware; }
subclass "pool1" 1:02:00:00:00:a1:01; # itadori
subclass "pool1" 1:02:00:00:00:a1:02; # fushiguro
# Pool 2 - Tim Support (zenin, okkotsu)
class "pool2" { match hardware; }
subclass "pool2" 1:02:00:00:00:a2:01; # zenin
subclass "pool2" 1:02:00:00:00:a2:02; # okkotsu
# Network 4 - subnet lokal tengen, tanpa pool dinamis
subnet $P.4.0 netmask 255.255.255.0 {
}
# Network 2 - Divisi A (.36 - .69 sengaja tidak dialokasikan)
subnet $P.2.0 netmask 255.255.255.0 {
 option routers $P.2.1;
 option broadcast-address $P.2.255;
 option domain-name-servers 192.168.122.1;
 pool {
 range $P.2.10 $P.2.35;
 allow members of "pool1";
 }
 pool {
 range $P.2.70 $P.2.95;
 allow members of "pool2";
 }
}
# Network 3 - Divisi B
subnet $P.3.0 netmask 255.255.255.0 {
 range $P.3.10 $P.3.100;
 option routers $P.3.1;
 option broadcast-address $P.3.255;
 option domain-name-servers 192.168.122.1;
}
EOT
echo "===== Validasi konfigurasi ====="
dhcpd -t -cf /etc/dhcp/dhcpd.conf || exit 1
# kosongkan lease lama supaya hasil uji bersih
> /var/lib/dhcp/dhcpd.leases
service isc-dhcp-server stop 2>/dev/null; pkill dhcpd; rm -f /var/run/dhcpd.pid
service isc-dhcp-server start
sleep 2
service isc-dhcp-server status
EOF
bash /root/soal4.sh
cp /root/soal4.sh /etc/network/soal4.sh
cat > /root/dhcp.sh << 'EOF'
#!/bin/sh
# Minta IP lewat DHCP (melalui relay jichi), dengan MAC tetap untuk class DHCP
case "$(hostname)" in
 itadori) MAC=02:00:00:00:a1:01 ;;
 fushiguro) MAC=02:00:00:00:a1:02 ;;
 zenin) MAC=02:00:00:00:a2:01 ;;
 okkotsu) MAC=02:00:00:00:a2:02 ;;
esac
ip addr flush dev eth0
if [ -n "$MAC" ]; then
 ip link set eth0 down
 ip link set eth0 address $MAC
fi
ip link set eth0 up
udhcpc -i eth0 -n -q
echo "===== Hasil lease ====="
ip link show eth0 | grep ether
ip -br addr show eth0
ip route
ping -c 3 $(ip route | awk '/default/ {print $3}')
EOF
sh /root/dhcp.sh
cp /root/dhcp.sh /etc/network/dhcp.sh
```

### nomor 5
```
cat > /root/soal5.sh << 'EOF'
#!/bin/bash
# Soal 5 - tengen: short/long lease di Network 3 + DNS & domain-search
P=192.227
DOMAIN=k32.com
echo "nameserver 192.168.122.1" > /etc/resolv.conf
command -v dhcpd >/dev/null || (apt-get update && apt-get install -y isc-dhcp-server)
echo 'INTERFACESv4="eth0"' > /etc/default/isc-dhcp-server
cat > /etc/dhcp/dhcpd.conf << EOT
authoritative;
default-lease-time 600;
max-lease-time 7200;
# ===== Network 2: class berdasarkan MAC (soal 4) =====
class "pool1" { match hardware; }
subclass "pool1" 1:02:00:00:00:a1:01; # itadori
subclass "pool1" 1:02:00:00:00:a1:02; # fushiguro
class "pool2" { match hardware; }
subclass "pool2" 1:02:00:00:00:a2:01; # zenin
subclass "pool2" 1:02:00:00:00:a2:02; # okkotsu
# ===== Network 3: class berdasarkan MAC (soal 5) =====
class "short-lease" { match hardware; }
subclass "short-lease" 1:02:00:00:00:b3:01; # ino
subclass "short-lease" 1:02:00:00:00:b3:02; # kusakabe
class "long-lease" { match hardware; }
subclass "long-lease" 1:02:00:00:00:b3:03; # miwa
# Network 4 - subnet lokal tengen, tanpa pool dinamis
subnet $P.4.0 netmask 255.255.255.0 {
}
# Network 2 - Divisi A (.36 - .69 tidak dialokasikan)
subnet $P.2.0 netmask 255.255.255.0 {
 option routers $P.2.1;
 option broadcast-address $P.2.255;
 option domain-name-servers 192.168.122.1;
 pool {
 range $P.2.10 $P.2.35;
 allow members of "pool1";
 }
 pool {
 range $P.2.70 $P.2.95;
 allow members of "pool2";
 }
}
# Network 3 - Divisi B
subnet $P.3.0 netmask 255.255.255.0 {
 option routers $P.3.1;
 option broadcast-address $P.3.255;
 option domain-name-servers $P.1.2, $P.1.3;
 option domain-search "$DOMAIN";
 # short lease 30 menit - ino, kusakabe
 pool {
 range $P.3.10 $P.3.89;
 allow members of "short-lease";
 default-lease-time 1800;
 max-lease-time 1800;
 }
 # long lease 12 jam - miwa
 pool {
 range $P.3.100 $P.3.199;
 allow members of "long-lease";
 default-lease-time 43200;
 max-lease-time 43200;
 }
}
EOT
echo "===== Validasi konfigurasi ====="
dhcpd -t -cf /etc/dhcp/dhcpd.conf || exit 1
service isc-dhcp-server stop 2>/dev/null; pkill dhcpd; rm -f /var/run/dhcpd.pid
service isc-dhcp-server start
sleep 2
service isc-dhcp-server status
EOF
bash /root/soal5.sh
cp /root/soal5.sh /etc/network/soal5.sh
cat > /root/dhcp.sh << 'EOF'
#!/bin/sh
# Minta IP lewat DHCP (melalui relay jichi), dengan MAC tetap untuk class DHCP
case "$(hostname)" in
 itadori) MAC=02:00:00:00:a1:01 ;;
 fushiguro) MAC=02:00:00:00:a1:02 ;;
 zenin) MAC=02:00:00:00:a2:01 ;;
 okkotsu) MAC=02:00:00:00:a2:02 ;;
 ino) MAC=02:00:00:00:b3:01 ;;
 kusakabe) MAC=02:00:00:00:b3:02 ;;
 miwa) MAC=02:00:00:00:b3:03 ;;
 kugisaki) MAC=02:00:00:00:b3:04 ;;
esac
ip addr flush dev eth0
if [ -n "$MAC" ]; then
 ip link set eth0 down
 ip link set eth0 address $MAC
fi
ip link set eth0 up
udhcpc -i eth0 -n -q -O search -O domain
echo "===== Hasil lease ====="
ip link show eth0 | grep ether
ip -br addr show eth0
ip route
echo "----- /etc/resolv.conf -----"
cat /etc/resolv.conf
ping -c 3 $(ip route | awk '/default/ {print $3}')
EOF
sh /root/dhcp.sh
cp /root/dhcp.sh /etc/network/dhcp.sh
```
### nomor 6
```
cat > /root/soal6.sh << 'EOF'
#!/bin/bash
# Soal 6 - tengen: static lease (fixed address) untuk kugisaki
P=192.227
bash /root/soal5.sh > /dev/null
cat >> /etc/dhcp/dhcpd.conf << EOT
# ===== Static lease - kugisaki (VIP) =====
# .3.99 berada di luar pool short (.10-.89) dan pool long (.100-.199)
host kugisaki {
 hardware ethernet 02:00:00:00:b3:04;
 fixed-address $P.3.99;
 option routers $P.3.1;
 option domain-name-servers $P.1.2, $P.1.3;
 option domain-search "k32.com";
}
EOT
echo "===== Validasi konfigurasi ====="
dhcpd -t -cf /etc/dhcp/dhcpd.conf || exit 1
service isc-dhcp-server stop 2>/dev/null; pkill dhcpd; rm -f /var/run/dhcpd.pid
service isc-dhcp-server start
sleep 2
service isc-dhcp-server status
echo "===== Reservasi kugisaki ====="
grep -A6 "host kugisaki" /etc/dhcp/dhcpd.conf
EOF
bash /root/soal6.sh
cp /root/soal6.sh /etc/network/soal6.sh
```
### nomor 7
```
cat > /root/soal7.sh << 'EOF'
#!/bin/bash
# Soal 7 - tengen: samakan DNS & domain-search untuk Network 2
bash /root/soal6.sh > /dev/null
sed -i 's|option domain-name-servers 192.168.122.1;|option domain-name-servers 192.227.1.2,
192.227.1.3;\n option domain-search "k32.com";|' /etc/dhcp/dhcpd.conf
echo "===== Validasi konfigurasi ====="
dhcpd -t -cf /etc/dhcp/dhcpd.conf || exit 1
service isc-dhcp-server stop 2>/dev/null; pkill dhcpd; rm -f /var/run/dhcpd.pid
service isc-dhcp-server start
sleep 2
service isc-dhcp-server status
echo "===== Opsi per subnet ====="
grep -E "subnet|routers|broadcast|domain-name-servers|domain-search" /etc/dhcp/dhcpd.conf
EOF
cat > /root/cek7.sh << 'EOF'
#!/bin/bash
# Soal 7 - tengen: cocokkan lease aktif dengan kebijakan pool & cek bentrok alamat
LEASES=${1:-/var/lib/dhcp/dhcpd.leases}
awk '
 /^lease / { ip=$2; mac="-"; st="-" }
 $1=="binding" { st=$3; sub(";","",st) }
 /hardware ethernet/ { mac=$3; sub(";","",mac) }
 /^}/ { if (ip!="") L[ip]=mac" "st; ip="" }
 END { for (i in L) print i, L[i] }' "$LEASES" | sort -t. -k3,3n -k4,4n > /tmp/lease_now
printf "%-16s %-19s %-10s %-9s %-14s %s\n" IP MAC KLIEN STATUS POOL HASIL
ERR=0
while read -r ip mac st; do
 [ "$st" = "active" ] || continue
 net=$(echo "$ip" | cut -d. -f3); h=$(echo "$ip" | cut -d. -f4)
 case "$mac" in
 02:00:00:00:a1:01) n=itadori; lo=10; hi=35; want=2 ;;
 02:00:00:00:a1:02) n=fushiguro; lo=10; hi=35; want=2 ;;
 02:00:00:00:a2:01) n=zenin; lo=70; hi=95; want=2 ;;
 02:00:00:00:a2:02) n=okkotsu; lo=70; hi=95; want=2 ;;
 02:00:00:00:b3:01) n=ino; lo=10; hi=89; want=3 ;;
 02:00:00:00:b3:02) n=kusakabe; lo=10; hi=89; want=3 ;;
 02:00:00:00:b3:03) n=miwa; lo=100; hi=199; want=3 ;;
 *) n="?"; lo=0; hi=-1; want=0 ;;
 esac
 if [ "$net" = "$want" ] && [ "$h" -ge "$lo" ] && [ "$h" -le "$hi" ]; then
 res="OK"
 else
 res="DI LUAR POOL"; ERR=1
 fi
 printf "%-16s %-19s %-10s %-9s %-14s %s\n" "$ip" "$mac" "$n" "$st" ".$want.$lo-.$want.$hi" "$res"
done < /tmp/lease_now
echo
DUP_IP=$(awk '$3=="active"{print $1}' /tmp/lease_now | sort | uniq -d)
DUP_MAC=$(awk '$3=="active"{print $2}' /tmp/lease_now | sort | uniq -d)
if [ -z "$DUP_IP$DUP_MAC" ]; then
 echo "Bentrok alamat : TIDAK ADA (setiap IP & MAC aktif unik)"
else
 echo "Bentrok alamat : ADA -> IP: $DUP_IP MAC: $DUP_MAC"; ERR=1
fi
if awk '$3=="active"{print $1}' /tmp/lease_now | grep -q '\.3\.99$'; then
 echo "Fixed address : .3.99 ikut terpakai sebagai lease dinamis (SALAH)"; ERR=1
else
 echo "Fixed address : .3.99 tidak dipakai pool dinamis (khusus kugisaki)"
fi
[ $ERR -eq 0 ] && echo "KESIMPULAN : semua lease sesuai kebijakan pool" \
 || echo "KESIMPULAN : ada lease yang melanggar kebijakan"
EOF
bash /root/soal7.sh
cp /root/soal7.sh /root/cek7.sh /etc/network/
```
### nomor 8
```
root@tengen:~# cat > /root/soal8.sh << 'EOF'
#!/bin/bash
# Soal 8 - tengen: capture DHCP DORA (lewat relay jichi)
command -v tcpdump >/dev/null || (apt-get update && apt-get install -y tcpdump)
echo ">>> Menunggu 4 paket DHCP... sekarang jalankan 'sh /root/dhcp.sh' di client"
tcpdump -i eth0 -n -vv -c 4 -l 'udp port 67 or udp port 68' 2>&1 | tee /root/soal8_capture.txt
echo
echo "================ RINGKASAN ================"
grep -E "IP [0-9]|DHCP-Message|Gateway-IP|Your-IP|Server-ID|Lease-Time|Client-Ethernet"
/root/soal8_capture.txt
EOF
cp /root/soal8.sh /etc/network/soal8.sh
bash /root/soal8.sh
>>> Menunggu 4 paket DHCP... sekarang jalankan 'sh /root/dhcp.sh' di client
tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
22:35:21.666226 IP (tos 0x0, ttl 64, id 63358, offset 0, flags [DF], proto UDP (17), length 328)
 192.227.4.1.67 > 192.227.4.10.67: [udp sum ok] BOOTP/DHCP, Request from 02:00:00:00:a1:01,
length 300, hops 1, xid 0x981a035b, Flags [none] (0x0000)
 Gateway-IP 192.227.2.1
 Client-Ethernet-Address 02:00:00:00:a1:01
 Vendor-rfc1048 Extensions
 Magic Cookie 0x63825363
 DHCP-Message (53), length 1: Discover
 MSZ (57), length 2: 576
 Parameter-Request (55), length 8:
 Subnet-Mask (1), Default-Gateway (3), Domain-Name-Server (6), Hostname (12)
 Domain-Name (15), BR (28), NTP (42), Unknown (119)
 Vendor-Class (60), length 12: "udhcp 1.37.0"
 Client-ID (61), length 7: ether 02:00:00:00:a1:01
22:35:21.666632 IP (tos 0x0, ttl 64, id 37379, offset 0, flags [DF], proto UDP (17), length 328)
 192.227.4.10.67 > 192.227.2.1.67: [udp sum ok] BOOTP/DHCP, Reply, length 300, hops 1, xid
0x981a035b, Flags [none] (0x0000)
 Your-IP 192.227.2.10
 Gateway-IP 192.227.2.1
 Client-Ethernet-Address 02:00:00:00:a1:01
 Vendor-rfc1048 Extensions
 Magic Cookie 0x63825363
 DHCP-Message (53), length 1: Offer
 Server-ID (54), length 4: 192.227.4.10
 Lease-Time (51), length 4: 600
 Subnet-Mask (1), length 4: 255.255.255.0
 Default-Gateway (3), length 4: 192.227.2.1
 Domain-Name-Server (6), length 8: 192.227.1.2,192.227.1.3
 BR (28), length 4: 192.227.2.255
 Unknown (119), length 9: 3.107.51.50.3.99.111.109.0
22:35:21.679161 IP (tos 0x0, ttl 64, id 63359, offset 0, flags [DF], proto UDP (17), length 328)
 192.227.4.1.67 > 192.227.4.10.67: [udp sum ok] BOOTP/DHCP, Request from 02:00:00:00:a1:01,
length 300, hops 1, xid 0x981a035b, Flags [none] (0x0000)
 Gateway-IP 192.227.2.1
 Client-Ethernet-Address 02:00:00:00:a1:01
 Vendor-rfc1048 Extensions
 Magic Cookie 0x63825363
 DHCP-Message (53), length 1: Request
 Requested-IP (50), length 4: 192.227.2.10
 Server-ID (54), length 4: 192.227.4.10
 MSZ (57), length 2: 576
 Parameter-Request (55), length 8:
 Subnet-Mask (1), Default-Gateway (3), Domain-Name-Server (6), Hostname (12)
 Domain-Name (15), BR (28), NTP (42), Unknown (119)
 Vendor-Class (60), length 12: "udhcp 1.37.0"
 Client-ID (61), length 7: ether 02:00:00:00:a1:01
22:35:21.680220 IP (tos 0x0, ttl 64, id 37382, offset 0, flags [DF], proto UDP (17), length 328)
 192.227.4.10.67 > 192.227.2.1.67: [udp sum ok] BOOTP/DHCP, Reply, length 300, hops 1, xid
0x981a035b, Flags [none] (0x0000)
 Your-IP 192.227.2.10
 Gateway-IP 192.227.2.1
 Client-Ethernet-Address 02:00:00:00:a1:01
 Vendor-rfc1048 Extensions
 Magic Cookie 0x63825363
 DHCP-Message (53), length 1: ACK
 Server-ID (54), length 4: 192.227.4.10
 Lease-Time (51), length 4: 600
 Subnet-Mask (1), length 4: 255.255.255.0
 Default-Gateway (3), length 4: 192.227.2.1
 Domain-Name-Server (6), length 8: 192.227.1.2,192.227.1.3
 BR (28), length 4: 192.227.2.255
 Unknown (119), length 9: 3.107.51.50.3.99.111.109.0
4 packets captured
4 packets received by filter
0 packets dropped by kernel
================ RINGKASAN ================
 Gateway-IP 192.227.2.1
 Client-Ethernet-Address 02:00:00:00:a1:01
 DHCP-Message (53), length 1: Discover
 Your-IP 192.227.2.10
 Gateway-IP 192.227.2.1
 Client-Ethernet-Address 02:00:00:00:a1:01
 DHCP-Message (53), length 1: Offer
 Server-ID (54), length 4: 192.227.4.10
 Lease-Time (51), length 4: 600
 Gateway-IP 192.227.2.1
 Client-Ethernet-Address 02:00:00:00:a1:01
 DHCP-Message (53), length 1: Request
 Server-ID (54), length 4: 192.227.4.10
 Your-IP 192.227.2.10
 Gateway-IP 192.227.2.1
 Client-Ethernet-Address 02:00:00:00:a1:01
 DHCP-Message (53), length 1: ACK
 Server-ID (54), length 4: 192.227.4.10
 Lease-Time (51), length 4: 600
root@tengen:~#
```
### nomor 9
```
itadori:~# sh /root/dhcp.sh
udhcpc: started, v1.37.0
udhcpc: broadcasting discover
udhcpc: broadcasting select for 192.227.2.10, server 192.227.4.10
udhcpc: lease of 192.227.2.10 obtained from 192.227.4.10, lease time 600
===== Hasil lease =====
 link/ether 02:00:00:00:a1:01 brd ff:ff:ff:ff:ff:ff
eth0 UNKNOWN 192.227.2.10/24 fe80::ff:fe00:a101/64
default via 192.227.2.1 dev eth0 metric 83254
192.227.2.0/24 dev eth0 proto kernel scope link src 192.227.2.10
PING 192.227.2.1 (192.227.2.1) 56(84) bytes of data.
64 bytes from 192.227.2.1: icmp_seq=1 ttl=64 time=0.381 ms
64 bytes from 192.227.2.1: icmp_seq=2 ttl=64 time=0.403 ms
64 bytes from 192.227.2.1: icmp_seq=3 ttl=64 time=0.484 ms
--- 192.227.2.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2284ms
rtt min/avg/max/mdev = 0.381/0.422/0.484/0.044 ms
itadori:~#
root@jichi:~# bash /root/soal9.sh start
Requesting: eth2 as upstream: N downstream: Y
Requesting: eth3 as upstream: N downstream: Y
Requesting: eth4 as upstream: Y downstream: N
Internet Systems Consortium DHCP Relay Agent 4.4.3-P1
Copyright 2004-2022 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/
Listening on LPF/eth1/02:42:df:69:70:01
Sending on LPF/eth1/02:42:df:69:70:01
Listening on LPF/eth0/02:42:df:69:70:00
Sending on LPF/eth0/02:42:df:69:70:00
Listening on LPF/eth4/02:42:df:69:70:04
Sending on LPF/eth4/02:42:df:69:70:04
Listening on LPF/eth3/02:42:df:69:70:03
Sending on LPF/eth3/02:42:df:69:70:03
Listening on LPF/eth2/02:42:df:69:70:02
Sending on LPF/eth2/02:42:df:69:70:02
Sending on Socket/fallback
===== Konfigurasi relay =====
SERVERS="192.227.4.10"
INTERFACES=""
OPTIONS="-id eth2 -id eth3 -iu eth4"
===== Proses relay =====
root 532 0.0 0.0 5988 3456 ? Ss 22:54 0:00 dhcrelay -4 -id eth2 -id eth3 -iu eth4
192.227.4.10
===== Cek jalur ke tengen =====
PING 192.227.4.10 (192.227.4.10) 56(84) bytes of data.
64 bytes from 192.227.4.10: icmp_seq=1 ttl=64 time=1.03 ms
64 bytes from 192.227.4.10: icmp_seq=2 ttl=64 time=0.221 ms
--- 192.227.4.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1112ms
rtt min/avg/max/mdev = 0.221/0.626/1.032/0.405 ms
```
### nomor 10
```
root@jichi:~# cat > /root/soal10.sh << 'EOF'
#!/bin/bash
# Soal 10 - autostart DHCP Relay (jichi) & DHCP Server (tengen) setelah restart
# 1) Script yang dijalankan otomatis setiap node boot
cat > /etc/network/autostart.sh << 'EOT'
#!/bin/bash
echo "===== autostart $(hostname) $(date) ====="
sleep 3
cp -n /etc/network/*.sh /root/ 2>/dev/null
case "$(hostname)" in
 jichi)
 bash /root/soal1.sh
 bash /root/soal3.sh
 ;;
 tengen)
 bash /root/soal1.sh
 # tunggu jichi siap (internet dibutuhkan bila paket harus diinstal ulang)
p 2; done in $(seq 1 60); do ping -c1 -W1 8.8.8.8 >/dev/null 2>&1 && break; sleep
 bash /root/soal7.sh
 ;;
esac
echo "===== autostart selesai $(date) ====="
bash /root/soal10.shostart.shstart.sh ====="/dev/null/null ;;e saat boot/dev/nul
===== /etc/network/interfaces =====
#
# This is a sample network config, please uncomment lines to configure the network
#
# Uncomment this line to load custom interface files
# source /etc/network/interfaces.d/*
# Static config for eth0
#auto eth0
#iface eth0 inet static
# address 192.168.0.2
# netmask 255.255.255.0
# gateway 192.168.0.1
# up echo nameserver 192.168.0.1 > /etc/resolv.conf
# DHCP config for eth0
#auto eth0
#iface eth0 inet dhcp
# hostname debinet-1
# Static config for eth1
#auto eth1
#iface eth1 inet static
# address 192.168.1.2
# netmask 255.255.255.0
# gateway 192.168.1.1
# up echo nameserver 192.168.1.1 > /etc/resolv.conf
# DHCP config for eth1
#auto eth1
#iface eth1 inet dhcp
# hostname debinet-1
# Static config for eth2
#auto eth2
#iface eth2 inet static
# address 192.168.2.2
# netmask 255.255.255.0
# gateway 192.168.2.1
# up echo nameserver 192.168.2.1 > /etc/resolv.conf
# DHCP config for eth2
#auto eth2
#iface eth2 inet dhcp
# hostname debinet-1
# Static config for eth3
#auto eth3
#iface eth3 inet static
# address 192.168.3.2
# netmask 255.255.255.0
# gateway 192.168.3.1
# up echo nameserver 192.168.3.1 > /etc/resolv.conf
# DHCP config for eth3
#auto eth3
#iface eth3 inet dhcp
# hostname debinet-1
# Soal 10 - autostart layanan DHCP saat node boot
auto lo
iface lo inet loopback
 up setsid /etc/network/autostart.sh > /var/log/autostart.log 2>&1 < /dev/null &
===== /etc/network/autostart.sh =====
#!/bin/bash
echo "===== autostart $(hostname) $(date) ====="
sleep 3
cp -n /etc/network/*.sh /root/ 2>/dev/null
case "$(hostname)" in
 jichi)
 bash /root/soal1.sh
 bash /root/soal3.sh
 ;;
 tengen)
 bash /root/soal1.sh
 # tunggu jichi siap (internet dibutuhkan bila paket harus diinstal ulang)
 for i in $(seq 1 60); do ping -c1 -W1 8.8.8.8 >/dev/null 2>&1 && break; sleep 2; done
 bash /root/soal7.sh
 ;;
esac
echo "===== autostart selesai $(date) ====="
root@jichi:~# ls /etc/network
autostart.sh if-post-down.d if-up.d interfaces.d soal10.sh soal9.sh
if-down.d if-pre-up.d interfaces soal1.sh soal3.sh
root@jichi:~#
root@tengen:~# ls /etc/network
cek7.sh if-down.d if-post-down.d if-pre-up.d if-up.d interfaces interfaces.d soal1.sh soal2.sh
soal4.sh soal5.sh soal6.sh soal7.sh soal8.sh
root@tengen:~#
root@tengen:~# ls /etc/network ; tail -4 /etc/network/interfaces
cek7.sh if-down.d if-post-down.d if-pre-up.d if-up.d interfaces interfaces.d soal1.sh soal2.sh
soal4.sh soal5.sh soal6.sh soal7.sh soal8.sh
# DHCP config for eth3
#auto eth3
#iface eth3 inet dhcp
# hostname debinet-1
root@tengen:~#
```
### script utahime
```
# NOMOR 11
apt-get update
apt-get install bind9 bind9utils bind9-doc dnsutils -y

cat << 'EOF' > /etc/bind/named.conf.local
zone "jujutsu.k32.com" {
    type master;
    file "/etc/bind/db.jujutsu";
    allow-transfer { 192.227.1.3; };
};

zone "1.227.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.reverse";
    allow-transfer { 192.227.1.3; };
};
EOF

cat << 'EOF' > /etc/bind/db.jujutsu
; BIND data file for jujutsu.k32.com
$TTL    604800
@       IN      SOA     utahime.jujutsu.k32.com. root.jujutsu.k32.com. (
                              3         ; Serial dinaikkan
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      utahime.jujutsu.k32.com.
@       IN      NS      shoko.jujutsu.k32.com.

@       IN      A       192.227.1.2
utahime IN      A       192.227.1.2
shoko   IN      A       192.227.1.3
gojo    IN      A       192.227.1.4

www     IN      CNAME   gojo
EOF

kill $(pidof named) 2>/dev/null
named -u bind

# NOMOR 12
apt-get update
apt-get install -y bind9 dnsutils
cat << 'EOF' > /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";
    
    forwarders {
        192.168.122.1;
    };
    
    allow-query { any; };
};
EOF

cat << 'EOF' > /etc/bind/db.jujutsu
; BIND data file for jujutsu.k32.com
$TTL    604800
@       IN      SOA     utahime.jujutsu.k32.com. root.jujutsu.k32.com. (
                              4         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      utahime.jujutsu.k32.com.
@       IN      NS      shoko.jujutsu.k32.com.

@       IN      A       192.227.1.4
utahime IN      A       192.227.1.2
shoko   IN      A       192.227.1.3
gojo    IN      A       192.227.1.4

www     IN      CNAME   jujutsu.k32.com.
EOF

cat << 'EOF' > /etc/bind/db.reverse
; BIND reverse data file for 1.227.192.in-addr.arpa
$TTL    604800
@       IN      SOA     utahime.jujutsu.k32.com. root.jujutsu.k32.com. (
                              4         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      utahime.jujutsu.k32.com.
@       IN      NS      shoko.jujutsu.k32.com.

2       IN      PTR     utahime.jujutsu.k32.com.
3       IN      PTR     shoko.jujutsu.k32.com.
4       IN      PTR     jujutsu.k32.com.
EOF

kill $(pidof named) 2>/dev/null
named -u bind
```

### script shoko
```
# NOMOR 11
apt-get update
apt-get install bind9 dnsutils -y

cat << 'EOF' > /etc/bind/named.conf.local
zone "jujutsu.k32.com" {
    type slave;
    masters { 192.227.1.2; };
    file "/var/cache/bind/db.jujutsu";
};

zone "1.227.192.in-addr.arpa" {
    type slave;
    masters { 192.227.1.2; };
    file "/var/cache/bind/db.reverse";
};
EOF

named -u bind

sleep 3
ls -l /var/cache/bind/

# NOMOR 12
apt-get update
apt-get install -y bind9 dnsutils
cat << 'EOF' > /etc/bind/named.conf.local
zone "jujutsu.k32.com" {
    type slave;
    masters { 192.227.1.2; };
    file "/var/cache/bind/db.jujutsu";
    masterfile-format text;
};

zone "1.227.192.in-addr.arpa" {
    type slave;
    masters { 192.227.1.2; };
    file "/var/cache/bind/db.reverse";
    masterfile-format text;
};
EOF

rm -f /var/cache/bind/db.jujutsu /var/cache/bind/db.reverse

kill $(pidof named) 2>/dev/null
named -u bind

sleep 2
cat /var/cache/bind/db.jujutsu | grep -i Serial
```

### script yaga
```
# NOMOR 13
apt-get update -y
apt-get install -y mariadb-server mariadb-client
service mariadb start

mkdir -p /var/log/mysql
chown -R mysql:mysql /var/log/mysql

sed -i 's/bind-address.*/bind-address = 0.0.0.0/' /etc/mysql/mariadb.conf.d/50-server.cnf
sed -i '/\[mariadb\]/a server-id = 1\nlog_bin = /var/log/mysql/mariadb-bin.log\nbinlog_do_db = db_shibuya_k32' /etc/mysql/mariadb.conf.d/50-server.cnf

service mariadb restart

mariadb -u root <<EOF
CREATE DATABASE db_shibuya_k32;

CREATE USER 'k32'@'192.227.4.%' IDENTIFIED BY 'password123';
GRANT ALL PRIVILEGES ON db_shibuya_k32.* TO 'k32'@'192.227.4.%';

CREATE USER 'replica_user'@'%' IDENTIFIED BY 'password123';
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'%';
FLUSH PRIVILEGES;

USE db_shibuya_k32;
CREATE TABLE request_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    log_text VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
EOF

echo "=== CATAT NAMA FILE DAN POSISI DI BAWAH INI ==="
mariadb -u root -e "SHOW MASTER STATUS;"
```

### script inumaki
```
# NOMOR 13
apt-get update -y
apt-get install -y mariadb-server mariadb-client
service mariadb start

sed -i 's/bind-address.*/bind-address = 0.0.0.0/' /etc/mysql/mariadb.conf.d/50-server.cnf
sed -i '/\[mariadb\]/a server-id = 2' /etc/mysql/mariadb.conf.d/50-server.cnf

service mariadb restart

mariadb -u root <<EOF
CREATE DATABASE db_shibuya_k32;

USE db_shibuya_k32;
CREATE TABLE request_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    log_text VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CHANGE MASTER TO
    MASTER_HOST = '192.227.4.9',
    MASTER_USER = 'replica_user',
    MASTER_PASSWORD = 'password123',
    MASTER_LOG_FILE = 'mariadb-bin.000001',
    MASTER_LOG_POS = 1846;

START SLAVE;
EOF

echo "=== CEK STATUS REPLIKASI SLAVE ==="
mariadb -u root -e "SHOW SLAVE STATUS\G"
```

### script maki & panda
```
# NOMOR 14
apt-get update -y
apt-get install -y nginx php-fpm

rm -f /var/www/html/index.html

cat << 'EOF' > /var/www/html/index.php
<!DOCTYPE html>
<html>
<head>
    <title>Jujutsu Frontend</title>
</head>
<body style="font-family: Arial; text-align: center; margin-top: 50px;">
    <h1>Halo dari Frontend Worker!</h1>
    <p>Disajikan oleh Hostname: <strong><?php echo gethostname(); ?></strong></p>
</body>
</html>
EOF

cat << 'EOF' > /etc/nginx/sites-available/default
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root /var/www/html;
    index index.php index.html index.htm index.nginx-debian.html;

    server_name web.jujutsu.com;
    allow 192.227.1.4;
    deny all;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }
}
EOF

nginx -t
service nginx restart
service php8.4-fpm restart
```

### script nanami, todo, kirara
```
# NOMOR 14
apt-get update -y
apt-get install -y nginx php-fpm php-mysql

cat << 'EOF' > /var/www/html/index.php
<?php
$host = '192.227.4.9'; // IP Master yaga
$db   = 'db_shibuya_k32';
$user = 'k32';
$pass = 'password123';
$worker = gethostname();

$log_text = "API Request diterima dan diproses oleh worker: " . $worker;

try {
    $pdo = new PDO("mysql:host=$host;dbname=$db", $user, $pass);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    
    $stmt = $pdo->prepare("INSERT INTO request_logs (log_text) VALUES (:log_text)");
    $stmt->bindParam(':log_text', $log_text);
    $stmt->execute();
    
    echo "<h1>Sukses!</h1>";
    echo "<p>$log_text. Log telah berhasil dicatat ke database yaga.</p>";
} catch(PDOException $e) {
    echo "Koneksi/Insert Gagal: " . $e->getMessage();
}
?>
EOF

cat << 'EOF' > /etc/nginx/sites-available/default
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root /var/www/html;
    index index.php index.html index.htm index.nginx-debian.html;
    
    server_name api.jujutsu.com;
    allow 192.227.4.3;
    deny all;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }
}
EOF

nginx -t
service nginx restart
service php8.4-fpm restart
```

### script nonor 15 (geto)
```
#!/bin/bash
apt-get update -y
apt-get install -y nginx

cat << 'EOF' > /etc/nginx/sites-available/default
# Mendefinisikan grup API dan rasio bebannya (3:1)
upstream api_cluster {
    server 192.227.4.6 weight=3; 
    server 192.227.4.7 weight=1; 
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name api.jujutsu.com _;

    location / {
        # Melempar trafik ke grup upstream
        proxy_pass http://api_cluster;

        # Meneruskan informasi IP asli klien
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF

nginx -t
service nginx restart

echo "Load Balancer Geto dengan rasio 3:1 berhasil dikonfigurasi!"
```

### script nomor 16 (gojo)
`nano /root/soal16.sh`
```
#!/bin/bash
apt-get update -y
apt-get install -y nginx

cat << 'EOF' > /etc/nginx/sites-available/default
upstream frontend_cluster {
    server 192.227.4.4; # GANTI dengan IP Maki
    server 192.227.4.5; # GANTI dengan IP Panda
}

server {
    listen 80 default_server;
    server_name jujutsu.com www.jujutsu.com;

    location / {
        proxy_pass http://frontend_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/ {
        proxy_pass http://192.227.4.3/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

nginx -t
service nginx restart

echo "Gojo berhasil dikonfigurasi sebagai Reverse Proxy!"
```

### script nomor 17 gojo
```
#!/bin/bash
cat << 'EOF' > /etc/nginx/sites-available/default
server {
    listen 80 default_server;
    server_name jujutsu.com www.jujutsu.com;

    location / {
        proxy_pass http://frontend_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /api/ {
        proxy_pass http://192.227.4.3/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF
nginx -t && service nginx restart
echo "Gojo Selesai!"
```

### script noomr 17 geto
```
#!/bin/bash
cat << 'EOF' > /etc/nginx/sites-available/default
upstream api_cluster {
    server 192.227.4.4 weight=3; # IP nanami
    server 192.227.4.5 weight=1; # IP todo
}

server {
    listen 80 default_server;

    set_real_ip_from 192.227.1.4;
    real_ip_header X-Forwarded-For;

    location / {
        proxy_pass http://api_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF
nginx -t && service nginx restart
echo "Geto Selesai!"

```

### script nomor 17 yaga
```bash
#!/bin/bash
mysql -u root -p'password123' -e "USE db_shibuya_k32; ALTER TABLE request_logs ADD COLUMN IF NOT EXISTS client_ip VARCHAR(50);"
echo "Kolom client_ip berhasil ditambahkan di yaga!"

```

**4. Script untuk Worker Nanami & Todo (`/root/soal17_worker.sh`)**
*(Jalankan script ini di terminal nanami dan todo)*

```
#!/bin/bash
cat << 'EOF' > /var/www/html/index.php
<?php
$host = '192.227.4.9';
$db   = 'db_shibuya_k32';
$user = 'k32';
$pass = 'password123';
$worker = gethostname();

if (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) {
    $ip_list = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
    $client_ip = trim($ip_list[0]);
} elseif (!empty($_SERVER['HTTP_X_REAL_IP'])) {
    $client_ip = $_SERVER['HTTP_X_REAL_IP'];
} else {
    $client_ip = $_SERVER['REMOTE_ADDR'];
}

$log_text = "API Request diterima dan diproses oleh worker: " . $worker;

try {
    $pdo = new PDO("mysql:host=$host;dbname=$db", $user, $pass);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    $stmt = $pdo->prepare("INSERT INTO request_logs (log_text, client_ip) VALUES (:log_text, :client_ip)");
    $stmt->bindParam(':log_text', $log_text);
    $stmt->bindParam(':client_ip', $client_ip);
    $stmt->execute();
    echo "<h1>Sukses dari $worker!</h1>";
} catch(PDOException $e) {
    echo "Gagal: " . $e->getMessage();
}
?>
EOF
echo "Skrip PHP di worker berhasil diupdate!"

```

### script soal 18 kirara 
```
#!/bin/bash

cat << 'EOF' > /var/www/html/index.php
<?php
$host = '192.227.4.9';
$db   = 'db_shibuya_k32';
$user = 'k32';
$pass = 'password123';
$worker = gethostname();

if (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) {
    $ip_list = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
    $client_ip = trim($ip_list[0]);
} elseif (!empty($_SERVER['HTTP_X_REAL_IP'])) {
    $client_ip = $_SERVER['HTTP_X_REAL_IP'];
} else {
    $client_ip = $_SERVER['REMOTE_ADDR'];
}

$log_text = "API Request diterima dan diproses oleh worker CADANGAN: " . $worker;

try {
    $pdo = new PDO("mysql:host=$host;dbname=$db", $user, $pass);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    $stmt = $pdo->prepare("INSERT INTO request_logs (log_text, client_ip) VALUES (:log_text, :client_ip)");
    $stmt->bindParam(':log_text', $log_text);
    $stmt->bindParam(':client_ip', $client_ip);
    $stmt->execute();
    echo "<h1>Sukses dari $worker!</h1>";
    echo "<p>$log_text. Log dengan IP $client_ip telah dicatat.</p>";
} catch(PDOException $e) {
    echo "Gagal: " . $e->getMessage();
}
?>
EOF

echo "Kirara siap menjadi worker cadangan!"
```

### script soal 18 geto
```
#!/bin/bash

cat << 'EOF' > /etc/nginx/sites-available/default
upstream api_cluster {
    # Worker Utama dengan Passive Health Check
    server 192.227.4.6 weight=3 max_fails=2 fail_timeout=5s; 
    server 192.227.4.7 weight=1 max_fails=2 fail_timeout=5s;
    
    # Worker Cadangan
    server 192.227.4.6 backup; # IP Kirara
}

server {
    listen 80 default_server;

    set_real_ip_from 192.227.1.4;
    real_ip_header X-Forwarded-For;

    location / {
        proxy_pass http://api_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF

nginx -t && service nginx restart
echo "Geto berhasil dikonfigurasi dengan Health Check dan Server Cadangan!"
```

### script 19 gojo
```
#!/bin/bash

# Menyuntikkan konfigurasi Nginx baru ke Gojo
cat << 'EOF' > /etc/nginx/sites-available/default
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=web_cache:10m inactive=10m use_temp_path=off;
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

upstream frontend_cluster {
    server 192.227.4.4; # IP Maki
    server 192.227.4.5; # IP Panda
}

server {
    listen 80 default_server;
    server_name jujutsu.com www.jujutsu.com;

    location / {
        proxy_pass http://frontend_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Eksekusi Cache Frontend
        proxy_cache web_cache;
        proxy_cache_valid 200 302 10m;
        add_header X-Cache-Status $upstream_cache_status;
    }

    location /api/ {
        proxy_pass http://192.227.4.3/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Eksekusi Rate Limit API
        limit_req zone=api_limit;
    }
}
EOF

# Membuat direktori cache jika belum ada, test Nginx, lalu restart
mkdir -p /var/cache/nginx
nginx -t && service nginx restart

echo "Konfigurasi Caching dan Rate Limiting di Gojo selesai!"
```

### script 20 kusukabe
```
#!/bin/sh

echo "--- 1. Memperbaiki DNS untuk Akses Internet ---"
echo "nameserver 8.8.8.8" > /etc/resolv.conf

echo "--- 2. Menginstal Apache Benchmark ---"
apk update
apk add apache2-utils

echo "--- 3. Mendaftarkan Domain ke File Hosts ---"
# Mengecek agar tidak terjadi duplikat penulisan di /etc/hosts
if ! grep -q "jujutsu.com" /etc/hosts; then
    echo "192.227.1.4 jujutsu.com www.jujutsu.com" >> /etc/hosts
    echo "Domain jujutsu.com berhasil ditambahkan ke /etc/hosts"
else
    echo "Domain jujutsu.com sudah ada di /etc/hosts"
fi

echo ""
echo "========================================================="
echo "--- 4A. PENGUJIAN BASELINE (FRONTEND: / ) ---"
echo "Target: http://jujutsu.com/"
echo "Menunggu proses benchmark selesai..."
ab -n 600 -c 30 http://jujutsu.com/

echo ""
echo "========================================================="
echo "--- 4B. PENGUJIAN RATE LIMITING (API: /api/ ) ---"
echo "Target: http://jujutsu.com/api/"
echo "Menunggu proses benchmark selesai..."
ab -n 600 -c 30 http://jujutsu.com/api/
echo "========================================================="
echo "Pengujian dari Kusakabe Selesai!"
```