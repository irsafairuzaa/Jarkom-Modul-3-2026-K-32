# Jarkom-Modul-3-2026-K-32

Laporan resmi Praktikum Modul 3 Komunikasi Data & Jaringan Komputer 2026
**Curtain Protocol: Shibuya Incident**

| | |
|---|---|
| Kelompok | K-32 |
| Prefix IP | `192.227` |
| Domain | `k32.com` |
| Anggota | Anargya Shafa Setiyadi Putra - 5027251081 |

---

## Daftar Isi

- [Topologi](#topologi)
- [Pembagian Alamat IP](#pembagian-alamat-ip)
- [Struktur Repository](#struktur-repository)
- [Catatan Lingkungan GNS3](#catatan-lingkungan-gns3)
- [Soal 1 – Routing dasar, IP forwarding & NAT](#soal-1--routing-dasar-ip-forwarding--nat)
- [Soal 2 – DHCP Server tengen](#soal-2--dhcp-server-tengen)
- [Soal 3 – DHCP Relay jichi](#soal-3--dhcp-relay-jichi)
- [Soal 4 – Dua pool DHCP di Network 2](#soal-4--dua-pool-dhcp-di-network-2)
- [Soal 5 – Short lease, long lease, DNS & domain-search](#soal-5--short-lease-long-lease-dns--domain-search)
- [Soal 6 – Fixed address kugisaki](#soal-6--fixed-address-kugisaki)
- [Soal 7 – Verifikasi konfigurasi seluruh client](#soal-7--verifikasi-konfigurasi-seluruh-client)
- [Soal 8 – Packet capture DORA](#soal-8--packet-capture-dora)
- [Soal 9 – Simulasi gangguan relay](#soal-9--simulasi-gangguan-relay)
- [Soal 10 – Autostart setelah restart](#soal-10--autostart-setelah-restart)
- [Kendala & Solusi](#kendala--solusi)

---

## Topologi

![Topologi](assets/topologi.png)

Topologi terdiri atas satu router (**jichi**) yang menghubungkan NAT (internet) dan empat jaringan:

| Network | Peran | Interface jichi | Node |
|---|---|---|---|
| WAN | Internet (NAT) | `eth0` | NAT1 |
| Network 1 | DMZ | `eth1` | utahime, shoko, gojo |
| Network 2 | Divisi A (DHCP relay) | `eth2` | itadori, fushiguro, zenin, okkotsu |
| Network 3 | Divisi B (DHCP relay) | `eth3` | ino, kusakabe, miwa, kugisaki |
| Network 4 | Data Center | `eth4` | tengen, geto, maki, panda, nanami, todo, kirara, yaga, inumaki |

Image Docker yang digunakan:
- `ardhptr21/debinet:latest` untuk router dan seluruh server
- `ardhptr21/alpinet:latest` untuk seluruh client

Topologi (32 node dan 31 link) dibuat otomatis memakai script Python melalui REST API GNS3, yaitu [`scripts/gns3_jujutsu_topology.py`](scripts/gns3_jujutsu_topology.py).

---

## Pembagian Alamat IP

Semua subnet memakai mask `/24` (`255.255.255.0`).

### Router

| Interface | Alamat | Keterangan |
|---|---|---|
| jichi `eth0` | DHCP dari NAT (`192.168.122.x`) | WAN |
| jichi `eth1` | `192.227.1.1` | Gateway Network 1 |
| jichi `eth2` | `192.227.2.1` | Gateway Network 2 |
| jichi `eth3` | `192.227.3.1` | Gateway Network 3 |
| jichi `eth4` | `192.227.4.1` | Gateway Network 4 |

### Host statis

| Node | Peran | IP | Gateway |
|---|---|---|---|
| utahime | DNS Master | `192.227.1.2` | `192.227.1.1` |
| shoko | DNS Slave | `192.227.1.3` | `192.227.1.1` |
| gojo | Public Reverse Proxy | `192.227.1.4` | `192.227.1.1` |
| inumaki | Database Slave | `192.227.4.2` | `192.227.4.1` |
| geto | Internal Load Balancer | `192.227.4.3` | `192.227.4.1` |
| maki | Worker frontend | `192.227.4.4` | `192.227.4.1` |
| panda | Worker frontend | `192.227.4.5` | `192.227.4.1` |
| nanami | Worker API utama | `192.227.4.6` | `192.227.4.1` |
| todo | Worker API utama | `192.227.4.7` | `192.227.4.1` |
| kirara | Worker API cadangan | `192.227.4.8` | `192.227.4.1` |
| yaga | Database Master | `192.227.4.9` | `192.227.4.1` |
| tengen | DHCP Server | `192.227.4.10` | `192.227.4.1` |

### Client (DHCP)

MAC address setiap client ditetapkan secara tetap oleh script `dhcp.sh`, supaya DHCP server dapat mengenali kelompok client melalui *class* berbasis MAC.

| Client | MAC | Kebijakan | Rentang / Alamat | Lease |
|---|---|---|---|---|
| itadori | `02:00:00:00:a1:01` | Pool 1 (Dev) | `192.227.2.10 – .35` | 600 s |
| fushiguro | `02:00:00:00:a1:02` | Pool 1 (Dev) | `192.227.2.10 – .35` | 600 s |
| zenin | `02:00:00:00:a2:01` | Pool 2 (Support) | `192.227.2.70 – .95` | 600 s |
| okkotsu | `02:00:00:00:a2:02` | Pool 2 (Support) | `192.227.2.70 – .95` | 600 s |
| ino | `02:00:00:00:b3:01` | Short lease | `192.227.3.10 – .89` | 1800 s |
| kusakabe | `02:00:00:00:b3:02` | Short lease | `192.227.3.10 – .89` | 1800 s |
| miwa | `02:00:00:00:b3:03` | Long lease | `192.227.3.100 – .199` | 43200 s |
| kugisaki | `02:00:00:00:b3:04` | Fixed address | `192.227.3.99` | 600 s |

---

## Struktur Repository

```
.
├── README.md
├── assets/                         # screenshot hasil pengujian
└── scripts/
    ├── gns3_jujutsu_topology.py    # pembuat topologi otomatis via API GNS3
    ├── server/soal1.sh             # IP statis jichi & seluruh server
    ├── jichi/
    │   ├── soal3.sh                # DHCP relay
    │   ├── soal9.sh                # simulasi gangguan relay (stop/start)
    │   └── soal10.sh               # autostart
    ├── tengen/
    │   ├── soal2.sh … soal7.sh     # konfigurasi isc-dhcp-server bertahap
    │   ├── cek7.sh                 # pemeriksa kebijakan pool & bentrok alamat
    │   ├── soal8.sh                # tcpdump DORA
    │   └── soal10.sh               # autostart
    └── client/dhcp.sh              # permintaan DHCP dari client
```

Di dalam node, seluruh script disimpan di `/root` sesuai ketentuan praktikum. Salinan cadangannya disimpan di `/etc/network/`.

> Output terminal pada laporan ini sebagian dipersingkat agar fokus pada baris yang menjadi bukti. Output lengkap dapat dilihat pada screenshot di folder `assets/`.

---

## Catatan Lingkungan GNS3

Dua perilaku node Docker di GNS3 berpengaruh langsung pada cara pengerjaan:

1. **Hanya `/etc/network` yang persisten.** Saat node di-restart atau project dibuka ulang, container dibuat ulang. Akibatnya IP yang dipasang dengan perintah `ip`, paket yang diinstal, isi `/root`, dan file lease ikut hilang. Karena itu setiap script dicadangkan ke `/etc/network/`, dan setiap script mengecek serta menginstal ulang paket bila belum ada.
2. **Tidak ada systemd.** Perintah `systemctl enable` tidak berlaku. Saat container start, GNS3 menjalankan `ifup -a` (versi busybox) terhadap `/etc/network/interfaces`, dan mekanisme inilah yang dipakai untuk autostart di soal 10.

Jika project dibuka ulang sebelum autostart terpasang, urutan pemulihan manualnya:

```bash
# 1. jichi (selalu pertama, karena semua jalur melewati jichi)
cp -n /etc/network/*.sh /root/ ; bash /root/soal1.sh ; bash /root/soal3.sh
# 2. setiap server
cp -n /etc/network/*.sh /root/ ; bash /root/soal1.sh
# 3. tengen
bash /root/soal7.sh
# 4. setiap client
cp -n /etc/network/dhcp.sh /root/ ; sh /root/dhcp.sh
```

---

## Soal 1 – Routing dasar, IP forwarding & NAT

### Tujuan
Memulihkan konektivitas dasar. jichi mendapat WAN dari NAT melalui `eth0` dan menjadi gateway Network 1–4 melalui `eth1`–`eth4`. IPv4 forwarding dan source NAT diaktifkan, lalu seluruh host infrastruktur diberi IP statis.

### Langkah
Satu script yang sama, [`scripts/server/soal1.sh`](scripts/server/soal1.sh), dijalankan di jichi dan di 12 server. Script membaca `hostname` untuk menentukan perannya.

- **Di jichi**, script:
  - meminta IP DHCP dari NAT di `eth0`;
  - memasang `192.227.N.1/24` di `eth1`–`eth4`;
  - mengaktifkan `net.ipv4.ip_forward=1`;
  - menambahkan aturan `MASQUERADE` untuk `192.227.0.0/16` yang keluar lewat `eth0`.
- **Di server**, script memasang IP sesuai tabel, menambahkan default route ke gateway subnetnya, dan mengisi nameserver `192.168.122.1` (dipakai untuk persiapan instalasi paket).

```bash
bash /root/soal1.sh
cp /root/soal1.sh /etc/network/soal1.sh
```

Inti konfigurasi di jichi:

```bash
for n in 1 2 3 4; do
  ip addr replace 192.227.$n.1/24 dev eth$n
  ip link set eth$n up
done
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -s 192.227.0.0/16 -o eth0 -j MASQUERADE
```

### Hasil Uji

**jichi**: interface, routing, forwarding, dan NAT.

```
root@jichi:~# ip -br addr; ip route; cat /proc/sys/net/ipv4/ip_forward; iptables -t nat -S POSTROUTING
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UNKNOWN        192.168.122.2/24 fe80::42:dfff:fe69:7000/64
eth1             UNKNOWN        192.227.1.1/24 fe80::42:dfff:fe69:7001/64
eth2             UNKNOWN        192.227.2.1/24 fe80::42:dfff:fe69:7002/64
eth3             UNKNOWN        192.227.3.1/24 fe80::42:dfff:fe69:7003/64
eth4             UNKNOWN        192.227.4.1/24 fe80::42:dfff:fe69:7004/64
default via 192.168.122.1 dev eth0
192.168.122.0/24 dev eth0 proto kernel scope link src 192.168.122.2
192.227.1.0/24 dev eth1 proto kernel scope link src 192.227.1.1
192.227.2.0/24 dev eth2 proto kernel scope link src 192.227.2.1
192.227.3.0/24 dev eth3 proto kernel scope link src 192.227.3.1
192.227.4.0/24 dev eth4 proto kernel scope link src 192.227.4.1
1
-P POSTROUTING ACCEPT
-A POSTROUTING -s 192.227.0.0/16 -o eth0 -j MASQUERADE
```

**Contoh server (panda)**: ping ke gateway dan ke internet.

```
eth0             UNKNOWN        192.227.4.5/24
Gateway: 192.227.4.1
--- 192.227.4.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2006ms
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
PING google.com (172.217.194.139) 56(84) bytes of data.
--- google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
```

**Kesimpulan:**
- Setiap interface jichi sudah memegang alamat gateway yang sesuai.
- `ip_forward` bernilai `1`, dan aturan MASQUERADE aktif.
- Host di Network 4 berhasil menjangkau gateway, `8.8.8.8`, dan `google.com`. Hasil yang sama didapat di seluruh server.

![Soal 1 - jichi](assets/soal1-jichi.png)
![Soal 1 - server](assets/soal1-server.png)

---

## Soal 2 – DHCP Server tengen

### Tujuan
Mengonfigurasi `isc-dhcp-server` di tengen dengan alamat `192.227.4.10`. Network 4 dideklarasikan sebagai subnet lokal tanpa pool dinamis, sedangkan Network 2 dan Network 3 diberi scope DHCP.

### Langkah
1. **Menyesuaikan IP tengen ke `192.227.4.10`.** Pembagian awal memberi tengen `.4.2` dan inumaki `.4.10`, sehingga keduanya ditukar:

    ```bash
    sed -i -E 's/tengen\)( +)IP=\$P\.4\.2 /tengen)\1IP=$P.4.10 /; s/inumaki\)( +)IP=\$P\.4\.10 /inumaki)\1IP=$P.4.2 /' /root/soal1.sh
    ip addr flush dev eth0
    bash /root/soal1.sh
    cp /root/soal1.sh /etc/network/soal1.sh
    ```

2. **Menginstal dan mengonfigurasi DHCP server** dengan [`scripts/tengen/soal2.sh`](scripts/tengen/soal2.sh):
    - `INTERFACESv4="eth0"` di `/etc/default/isc-dhcp-server`;
    - `subnet 192.227.4.0` dideklarasikan **kosong** (tanpa `range`). Deklarasi ini wajib ada supaya dhcpd mengenali subnet interface-nya sendiri. Tanpanya, dhcpd gagal dengan error *"No subnet declaration for eth0"*;
    - scope Network 2 dan Network 3, masing-masing dengan `range`, `routers`, dan `broadcast-address`.

Potongan `dhcpd.conf`:

```
subnet 192.227.4.0 netmask 255.255.255.0 {
}
subnet 192.227.2.0 netmask 255.255.255.0 {
    range 192.227.2.10 192.227.2.100;
    option routers 192.227.2.1;
    option broadcast-address 192.227.2.255;
}
subnet 192.227.3.0 netmask 255.255.255.0 {
    range 192.227.3.10 192.227.3.100;
    option routers 192.227.3.1;
    option broadcast-address 192.227.3.255;
}
```

### Hasil Uji

IP tengen setelah penyesuaian:

```
root@tengen:~# bash /root/soal1.sh
eth0             UNKNOWN        192.227.4.10/24
Gateway: 192.227.4.1
--- 192.227.4.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2272ms
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2218ms
--- google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2218ms
```

Validasi konfigurasi dan status service:

```
===== Validasi konfigurasi =====
Internet Systems Consortium DHCP Server 4.4.3-P1
Config file: /etc/dhcp/dhcpd.conf
Database file: /var/lib/dhcp/dhcpd.leases
PID file: /var/run/dhcpd.pid
Launching IPv4 server only.
Starting ISC DHCPv4 server: dhcpd.
===== Status service =====
Status of ISC DHCPv4 server: dhcpd is running.
root  491  0.0  0.0  9520  6920 ?  Ss  07:59  0:00 /usr/sbin/dhcpd -4 -q -cf /etc/dhcp/dhcpd.conf eth0
```

**Kesimpulan:** `dhcpd -t` tidak menampilkan error apa pun, termasuk *"No subnet declaration"*, dan service berjalan di `eth0`. Pesan `invoke-rc.d: policy-rc.d denied execution of start` saat instalasi adalah perilaku normal di container: service memang tidak otomatis dinyalakan saat diinstal, sehingga script menyalakannya sendiri.

![Soal 2](assets/soal2.png)

---

## Soal 3 – DHCP Relay jichi

### Tujuan
Broadcast DHCP dari Network 2 dan 3 tidak bisa menyeberang router. Karena itu jichi dikonfigurasi sebagai DHCP relay menuju tengen (`192.227.4.10`), memakai `eth2` dan `eth3`.

### Langkah
[`scripts/jichi/soal3.sh`](scripts/jichi/soal3.sh) menginstal `isc-dhcp-relay`, lalu menjalankan:

```bash
dhcrelay -4 -id eth2 -id eth3 -iu eth4 192.227.4.10
```

- `-id eth2`, `-id eth3`: interface *downstream*, yaitu sisi client (Network 2 dan 3).
- `-iu eth4`: interface *upstream*, yaitu sisi DHCP server (Network 4).

`eth4` wajib didaftarkan sebagai upstream. Tanpanya, request memang terkirim ke tengen, tetapi balasan dari tengen tidak diterima relay sehingga tidak pernah sampai ke client.

Di sisi client, [`scripts/client/dhcp.sh`](scripts/client/dhcp.sh) menghapus IP lama, memasang MAC tetap, lalu meminta lease dengan `udhcpc`.

### Hasil Uji

**jichi**: relay aktif dengan peran interface yang benar.

```
Requesting: eth2 as upstream: N downstream: Y
Requesting: eth3 as upstream: N downstream: Y
Requesting: eth4 as upstream: Y downstream: N
Internet Systems Consortium DHCP Relay Agent 4.4.3-P1
===== Konfigurasi relay =====
SERVERS="192.227.4.10"
INTERFACES=""
OPTIONS="-id eth2 -id eth3 -iu eth4"
===== Proses relay =====
root  502  0.0  0.0  5988  3460 ?  Ss  10:23  0:00 dhcrelay -4 -id eth2 -id eth3 -iu eth4 192.227.4.10
===== Cek jalur ke tengen =====
2 packets transmitted, 2 received, 0% packet loss, time 1140ms
```

**itadori (Network 2)**: mendapat lease melalui relay.

```
itadori:~# sh /root/dhcp.sh
udhcpc: started, v1.37.0
udhcpc: broadcasting discover
udhcpc: broadcasting select for 192.227.2.10, server 192.227.4.10
udhcpc: lease of 192.227.2.10 obtained from 192.227.4.10, lease time 600
===== Hasil lease =====
eth0             UNKNOWN        192.227.2.10/24
default via 192.227.2.1 dev eth0 metric 83254
192.227.2.0/24 dev eth0 proto kernel scope link src 192.227.2.10
--- 192.227.2.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2320ms
```

**Kesimpulan:** client di subnet lain mendapat lease dari `192.227.4.10` dengan gateway yang sesuai. Client Network 3 juga mendapat lease melalui relay yang sama, seperti ditunjukkan pada hasil uji soal 5–7.

![Soal 3 - relay](assets/soal3-relay.png)
![Soal 3 - client](assets/soal3-client.png)

---

## Soal 4 – Dua pool DHCP di Network 2

### Tujuan
Memisahkan alokasi tim Dev (itadori, fushiguro) ke Pool 1 `192.227.2.10–.35` dan tim Support (zenin, okkotsu) ke Pool 2 `192.227.2.70–.95`. Rentang `.36–.69` tidak boleh dialokasikan.

### Langkah
Seluruh client berada di subnet yang sama, sehingga digunakan **DHCP class berbasis MAC address**. Masalahnya, MAC container dapat berubah saat node dibuat ulang. Karena itu `dhcp.sh` memasang MAC tetap (`ip link set eth0 address ...`) sebelum meminta lease.

Konfigurasi di tengen ([`scripts/tengen/soal4.sh`](scripts/tengen/soal4.sh)):

```
class "pool1" { match hardware; }
subclass "pool1" 1:02:00:00:00:a1:01;   # itadori
subclass "pool1" 1:02:00:00:00:a1:02;   # fushiguro

class "pool2" { match hardware; }
subclass "pool2" 1:02:00:00:00:a2:01;   # zenin
subclass "pool2" 1:02:00:00:00:a2:02;   # okkotsu

subnet 192.227.2.0 netmask 255.255.255.0 {
    option routers 192.227.2.1;
    option broadcast-address 192.227.2.255;
    pool {
        range 192.227.2.10 192.227.2.35;
        allow members of "pool1";
    }
    pool {
        range 192.227.2.70 192.227.2.95;
        allow members of "pool2";
    }
}
```

- `match hardware` mencocokkan field hardware milik client. Prefiks `1:` pada `subclass` menandakan tipe hardware Ethernet.
- Rentang `.36–.69` tidak dimasukkan ke pool mana pun, sehingga otomatis tidak pernah dialokasikan.

### Hasil Uji

Validasi konfigurasi:

```
===== Validasi konfigurasi =====
Internet Systems Consortium DHCP Server 4.4.3-P1
Stopping ISC DHCPv4 server: dhcpd.
Launching IPv4 server only.
Starting ISC DHCPv4 server: dhcpd.
Status of ISC DHCPv4 server: dhcpd is running.
```

**itadori (Pool 1)**:

```
udhcpc: lease of 192.227.2.10 obtained from 192.227.4.10, lease time 600
    link/ether 02:00:00:00:a1:01 brd ff:ff:ff:ff:ff:ff
eth0             UNKNOWN        192.227.2.10/24 fe80::ff:fe00:a101/64
default via 192.227.2.1 dev eth0 metric 83254
```

**okkotsu (Pool 2)**:

```
udhcpc: lease of 192.227.2.71 obtained from 192.227.4.10, lease time 600
    link/ether 02:00:00:00:a2:02 brd ff:ff:ff:ff:ff:ff
eth0             UNKNOWN        192.227.2.71/24 fe80::ff:fe00:a202/64
default via 192.227.2.1 dev eth0 metric 83267
```

Lease yang tercatat di tengen (`/var/lib/dhcp/dhcpd.leases`):

```
lease 192.227.2.70 {
  hardware ethernet 02:00:00:00:a2:01;     # zenin
lease 192.227.2.71 {
  hardware ethernet 02:00:00:00:a2:02;     # okkotsu
```

**Kesimpulan:** itadori dan fushiguro mendapat alamat di Pool 1 (`.2.10`, `.2.11`), sedangkan zenin dan okkotsu mendapat alamat di Pool 2 (`.2.70`, `.2.71`). Tidak ada alamat di rentang `.36–.69` yang dialokasikan.

![Soal 4](assets/soal4.png)

---

## Soal 5 – Short lease, long lease, DNS & domain-search

### Tujuan
Di Network 3, ino dan kusakabe mendapat short lease 1800 detik (pool `.3.10–.89`), dan miwa mendapat long lease 43200 detik (pool `.3.100–.199`). Client juga menerima DNS `192.227.1.2` dan `192.227.1.3`, serta domain-search `k32.com`.

### Langkah
Pola class berbasis MAC yang sama dipakai lagi. Setiap pool diberi `default-lease-time` dan `max-lease-time` masing-masing ([`scripts/tengen/soal5.sh`](scripts/tengen/soal5.sh)):

```
class "short-lease" { match hardware; }
subclass "short-lease" 1:02:00:00:00:b3:01;   # ino
subclass "short-lease" 1:02:00:00:00:b3:02;   # kusakabe
class "long-lease" { match hardware; }
subclass "long-lease" 1:02:00:00:00:b3:03;    # miwa

subnet 192.227.3.0 netmask 255.255.255.0 {
    option routers 192.227.3.1;
    option broadcast-address 192.227.3.255;
    option domain-name-servers 192.227.1.2, 192.227.1.3;
    option domain-search "k32.com";
    pool {
        range 192.227.3.10 192.227.3.89;
        allow members of "short-lease";
        default-lease-time 1800;
        max-lease-time 1800;
    }
    pool {
        range 192.227.3.100 192.227.3.199;
        allow members of "long-lease";
        default-lease-time 43200;
        max-lease-time 43200;
    }
}
```

Di sisi client, `udhcpc` dijalankan dengan `-O search -O domain`. Tanpa opsi itu, `udhcpc` tidak meminta opsi 119 (domain-search) ke server, sehingga baris `search` tidak akan muncul di `resolv.conf`.

### Hasil Uji

**ino (short lease)**:

```
udhcpc: lease of 192.227.3.10 obtained from 192.227.4.10, lease time 1800
    link/ether 02:00:00:00:b3:01 brd ff:ff:ff:ff:ff:ff
eth0             UNKNOWN        192.227.3.10/24 fe80::ff:fe00:b301/64
default via 192.227.3.1 dev eth0 metric 83270
----- /etc/resolv.conf -----
search k32.com
nameserver 192.227.1.2
nameserver 192.227.1.3
--- 192.227.3.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
```

Lease yang tercatat di tengen. Selisih `starts` dan `ends` menunjukkan durasi lease (waktu dalam UTC):

```
lease 192.227.3.10 {
  starts 3 2026/10/14 10:50:38;
  ends 3 2026/10/14 11:20:38;
  hardware ethernet 02:00:00:00:b3:01;     # ino      -> 30 menit
lease 192.227.3.11 {
  starts 3 2026/10/14 10:51:01;
  ends 3 2026/10/14 11:21:01;
  hardware ethernet 02:00:00:00:b3:02;     # kusakabe -> 30 menit
lease 192.227.3.100 {
  starts 3 2026/10/14 10:51:10;
  ends 3 2026/10/14 22:51:10;
  hardware ethernet 02:00:00:00:b3:03;     # miwa     -> 12 jam
```

| Client | IP | Durasi lease | Pool |
|---|---|---|---|
| ino | `192.227.3.10` | 30 menit | short ✅ |
| kusakabe | `192.227.3.11` | 30 menit | short ✅ |
| miwa | `192.227.3.100` | 12 jam | long ✅ |

**Kesimpulan:** lease time dan pool berbeda sesuai kelas client, dan opsi DNS serta domain-search diterima oleh client. DNS `192.227.1.2/.3` baru dapat dipakai untuk resolusi nama setelah DNS server di utahime dan shoko dikonfigurasi.

![Soal 5 - ino](assets/soal5-ino.png)
![Soal 5 - leases](assets/soal5-leases.png)

---

## Soal 6 – Fixed address kugisaki

### Tujuan
kugisaki selalu mendapat alamat `192.227.3.99` berdasarkan MAC address-nya.

### Langkah
Alamat `.3.99` berada di celah antara pool short (`.10–.89`) dan pool long (`.100–.199`), sehingga berada di luar seluruh pool dinamis. Deklarasi `host` ditambahkan ([`scripts/tengen/soal6.sh`](scripts/tengen/soal6.sh)):

```
host kugisaki {
    hardware ethernet 02:00:00:00:b3:04;
    fixed-address 192.227.3.99;
    option routers 192.227.3.1;
    option domain-name-servers 192.227.1.2, 192.227.1.3;
    option domain-search "k32.com";
}
```

### Hasil Uji

Validasi di tengen:

```
===== Validasi konfigurasi =====
Internet Systems Consortium DHCP Server 4.4.3-P1
Starting ISC DHCPv4 server: dhcpd.
Status of ISC DHCPv4 server: dhcpd is running.
===== Reservasi kugisaki =====
host kugisaki {
    hardware ethernet 02:00:00:00:b3:04;
    fixed-address 192.227.3.99;
    option routers 192.227.3.1;
    option domain-name-servers 192.227.1.2, 192.227.1.3;
    option domain-search "k32.com";
}
```

**kugisaki**, permintaan pertama:

```
kugisaki:~# sh /root/dhcp.sh
udhcpc: broadcasting select for 192.227.3.99, server 192.227.4.10
udhcpc: lease of 192.227.3.99 obtained from 192.227.4.10, lease time 600
    link/ether 02:00:00:00:b3:04 brd ff:ff:ff:ff:ff:ff
eth0             UNKNOWN        192.227.3.99/24 fe80::ff:fe00:b304/64
default via 192.227.3.1 dev eth0 metric 83280
search k32.com
nameserver 192.227.1.2
nameserver 192.227.1.3
--- 192.227.3.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2246ms
```

**kugisaki**, renew lease:

```
kugisaki:~# udhcpc -i eth0 -n -q -O search -O domain
udhcpc: broadcasting select for 192.227.3.99, server 192.227.4.10
udhcpc: lease of 192.227.3.99 obtained from 192.227.4.10, lease time 600
eth0             UNKNOWN        192.227.3.99/24 fe80::ff:fe00:b304/64
```

**Kesimpulan:** kugisaki menerima alamat reservasi `192.227.3.99`, baik pada permintaan pertama maupun setelah renew. Fixed address tidak dicatat di `dhcpd.leases`; ini memang perilaku normal isc-dhcp-server untuk reservasi.

![Soal 6](assets/soal6.png)

---

## Soal 7 – Verifikasi konfigurasi seluruh client

### Tujuan
Memastikan setiap client menerima mask `/24`, gateway, broadcast, DNS `192.227.1.2/.3`, dan domain-search yang benar. Lease juga dicocokkan dengan `dhcpd.leases` untuk memastikan tidak ada alamat yang bentrok atau berada di luar kebijakan pool.

### Langkah
1. Saat pemeriksaan, ditemukan bahwa Network 2 masih memakai DNS `192.168.122.1` tanpa domain-search. Opsi Network 2 disamakan dengan Network 3 menggunakan [`scripts/tengen/soal7.sh`](scripts/tengen/soal7.sh).
2. Setiap client meminta lease ulang, lalu konfigurasinya ditampilkan dengan:

    ```sh
    ip addr show eth0 | grep -E "ether|inet " ; ip route ; cat /etc/resolv.conf
    ```

3. Dibuat script pemeriksa [`scripts/tengen/cek7.sh`](scripts/tengen/cek7.sh), yang:
    - membaca status terakhir setiap IP di `dhcpd.leases`;
    - mencocokkan MAC setiap lease aktif dengan pool yang seharusnya;
    - mendeteksi IP atau MAC yang terduplikasi;
    - memastikan `.3.99` tidak ikut terpakai oleh pool dinamis.

### Hasil Uji

Opsi per subnet setelah diperbarui:

```
subnet 192.227.2.0 netmask 255.255.255.0 {
    option routers 192.227.2.1;
    option broadcast-address 192.227.2.255;
    option domain-name-servers 192.227.1.2, 192.227.1.3;
    option domain-search "k32.com";
subnet 192.227.3.0 netmask 255.255.255.0 {
    option routers 192.227.3.1;
    option broadcast-address 192.227.3.255;
    option domain-name-servers 192.227.1.2, 192.227.1.3;
    option domain-search "k32.com";
```

Contoh client Divisi A (**okkotsu**) dan Divisi B (**ino**):

```
===== okkotsu =====
    link/ether 02:00:00:00:a2:02 brd ff:ff:ff:ff:ff:ff
    inet 192.227.2.71/24 brd 192.227.2.255 scope global eth0
default via 192.227.2.1 dev eth0 metric 83267
search k32.com
nameserver 192.227.1.2
nameserver 192.227.1.3

===== ino =====
    link/ether 02:00:00:00:b3:01 brd ff:ff:ff:ff:ff:ff
    inet 192.227.3.10/24 brd 192.227.3.255 scope global eth0
default via 192.227.3.1 dev eth0 metric 83270
search k32.com
nameserver 192.227.1.2
nameserver 192.227.1.3
```

Pencocokan lease dengan kebijakan pool (`bash /root/cek7.sh` di tengen):

```
IP               MAC                 KLIEN      STATUS    POOL           HASIL
192.227.2.10     02:00:00:00:a1:01   itadori    active    .2.10-.2.35    OK
192.227.2.11     02:00:00:00:a1:02   fushiguro  active    .2.10-.2.35    OK
192.227.2.70     02:00:00:00:a2:01   zenin      active    .2.70-.2.95    OK
192.227.2.71     02:00:00:00:a2:02   okkotsu    active    .2.70-.2.95    OK
192.227.3.10     02:00:00:00:b3:01   ino        active    .3.10-.3.89    OK
192.227.3.11     02:00:00:00:b3:02   kusakabe   active    .3.10-.3.89    OK
192.227.3.100    02:00:00:00:b3:03   miwa       active    .3.100-.3.199  OK

Bentrok alamat : TIDAK ADA (setiap IP & MAC aktif unik)
Fixed address  : .3.99 tidak dipakai pool dinamis (khusus kugisaki)
KESIMPULAN     : semua lease sesuai kebijakan pool
```

**Kesimpulan:** semua client menerima mask `/24`, broadcast `.255`, gateway sesuai subnet, DNS `192.227.1.2/.3`, dan `search k32.com`. Ketujuh lease dinamis berada di pool yang benar, tanpa bentrok alamat, dan kugisaki memegang `.3.99` lewat reservasi.

> Lease short (30 menit) dapat kedaluwarsa bila pemeriksaan dilakukan terlalu lama setelah client meminta lease. Dalam kasus itu, client perlu menjalankan `dhcp.sh` ulang sebelum `cek7.sh`.

![Soal 7 - client](assets/soal7-client.png)
![Soal 7 - cek7](assets/soal7-cek.png)

---

## Soal 8 – Packet capture DORA

### Tujuan
Membuktikan proses DHCP berjalan sesuai protokol, dengan urutan DHCPDISCOVER → DHCPOFFER → DHCPREQUEST → DHCPACK. Dari paket yang tertangkap juga diidentifikasi alamat server, alamat yang diberikan, lease time, dan field `giaddr`.

### Langkah
[`scripts/tengen/soal8.sh`](scripts/tengen/soal8.sh) dijalankan di tengen dan menunggu 4 paket DHCP:

```bash
tcpdump -i eth0 -n -vv -c 4 -l 'udp port 67 or udp port 68' 2>&1 | tee /root/soal8_capture.txt
```

Selama tcpdump menunggu, itadori (Network 2) meminta lease baru dengan `sh /root/dhcp.sh`.

### Hasil Uji

```
22:35:21.666226 IP 192.227.4.1.67 > 192.227.4.10.67: BOOTP/DHCP, Request from 02:00:00:00:a1:01, length 300, hops 1, xid 0x981a035b
          Gateway-IP 192.227.2.1
          Client-Ethernet-Address 02:00:00:00:a1:01
            DHCP-Message (53), length 1: Discover
22:35:21.666632 IP 192.227.4.10.67 > 192.227.2.1.67: BOOTP/DHCP, Reply, length 300, hops 1, xid 0x981a035b
          Your-IP 192.227.2.10
          Gateway-IP 192.227.2.1
            DHCP-Message (53), length 1: Offer
            Server-ID (54), length 4: 192.227.4.10
            Lease-Time (51), length 4: 600
            Subnet-Mask (1), length 4: 255.255.255.0
            Default-Gateway (3), length 4: 192.227.2.1
            Domain-Name-Server (6), length 8: 192.227.1.2,192.227.1.3
            BR (28), length 4: 192.227.2.255
            Unknown (119), length 9: 3.107.51.50.3.99.111.109.0
22:35:21.679161 IP 192.227.4.1.67 > 192.227.4.10.67: BOOTP/DHCP, Request from 02:00:00:00:a1:01, length 300, hops 1, xid 0x981a035b
          Gateway-IP 192.227.2.1
            DHCP-Message (53), length 1: Request
            Requested-IP (50), length 4: 192.227.2.10
            Server-ID (54), length 4: 192.227.4.10
22:35:21.680220 IP 192.227.4.10.67 > 192.227.2.1.67: BOOTP/DHCP, Reply, length 300, hops 1, xid 0x981a035b
          Your-IP 192.227.2.10
          Gateway-IP 192.227.2.1
            DHCP-Message (53), length 1: ACK
            Server-ID (54), length 4: 192.227.4.10
            Lease-Time (51), length 4: 600
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```

| # | Pesan | Arah |
|---|---|---|
| 1 | Discover | `192.227.4.1` (relay) → `192.227.4.10` |
| 2 | Offer | `192.227.4.10` → `192.227.2.1` (giaddr) |
| 3 | Request | `192.227.4.1` (relay) → `192.227.4.10` |
| 4 | ACK | `192.227.4.10` → `192.227.2.1` (giaddr) |

| Field | Nilai | Sumber |
|---|---|---|
| Alamat server | `192.227.4.10` | `Server-ID (54)` |
| Alamat yang diberikan | `192.227.2.10` | `Your-IP` |
| Lease time | `600` detik | `Lease-Time (51)` |
| giaddr | `192.227.2.1` | `Gateway-IP` |

### Analisis
- Keempat paket memiliki `xid 0x981a035b` yang sama, sehingga merupakan satu transaksi DHCP.
- Discover diterima jichi melalui **eth2** (`192.227.2.1`). Sebagai relay, jichi mengisi `giaddr = 192.227.2.1` dan meneruskannya ke tengen dari interface `eth4` (`192.227.4.1`), dengan `hops 1`.
- Dari `giaddr` itu tengen mengetahui bahwa client berasal dari `192.227.2.0/24`, lalu memilih alamat dari pool Network 2. Offer dan ACK dikirim ke `giaddr`, kemudian diteruskan jichi ke client.
- Nilai `giaddr` sesuai dengan interface jichi yang menerima request dari subnet client.
- Opsi `Unknown (119)` adalah domain-search yang dikodekan dalam ASCII: `107 51 50` = `k32` dan `99 111 109` = `com`, sehingga isinya `k32.com`.

![Soal 8](assets/soal8.png)

---

## Soal 9 – Simulasi gangguan relay

### Tujuan
Menghentikan DHCP relay, mencatat gejala di client, lalu memulihkan relay dan membuktikan bahwa client kembali mendapat alamat dari pool yang benar dengan gateway semula.

### Langkah
[`scripts/jichi/soal9.sh`](scripts/jichi/soal9.sh) mempunyai dua mode:

```bash
bash /root/soal9.sh stop    # hentikan dhcrelay
bash /root/soal9.sh start   # jalankan ulang relay (soal3.sh)
```

Setelah relay dihentikan, itadori meminta lease baru. Relay kemudian dipulihkan dan itadori meminta lease lagi.

### Hasil Uji

**1. Relay dihentikan (jichi):**

```
root@jichi:~# bash /root/soal9.sh stop
===== Status relay setelah dihentikan =====
dhcrelay TIDAK berjalan (relay mati)
```

**2. Gejala di client (itadori):**

```
itadori:~# sh /root/dhcp.sh
udhcpc: started, v1.37.0
udhcpc: broadcasting discover
udhcpc: broadcasting discover
udhcpc: broadcasting discover
udhcpc: broadcasting discover
udhcpc: broadcasting discover
udhcpc failed to get a DHCP lease
udhcpc: no lease, failing
===== Hasil lease =====
    link/ether 02:00:00:00:a1:01 brd ff:ff:ff:ff:ff:ff
eth0             UNKNOWN        fe80::ff:fe00:a101/64
ping: usage error: Destination address required
```

**3. Relay dipulihkan (jichi):**

```
root@jichi:~# bash /root/soal9.sh start
Requesting: eth2 as upstream: N downstream: Y
Requesting: eth3 as upstream: N downstream: Y
Requesting: eth4 as upstream: Y downstream: N
===== Proses relay =====
root  532  0.0  0.0  5988  3456 ?  Ss  22:54  0:00 dhcrelay -4 -id eth2 -id eth3 -iu eth4 192.227.4.10
===== Cek jalur ke tengen =====
2 packets transmitted, 2 received, 0% packet loss, time 1112ms
```

**4. Client mendapat alamat kembali (itadori):**

```
itadori:~# sh /root/dhcp.sh
udhcpc: broadcasting select for 192.227.2.10, server 192.227.4.10
udhcpc: lease of 192.227.2.10 obtained from 192.227.4.10, lease time 600
eth0             UNKNOWN        192.227.2.10/24 fe80::ff:fe00:a101/64
default via 192.227.2.1 dev eth0 metric 83254
--- 192.227.2.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2284ms
```

| Kondisi | Relay | Client |
|---|---|---|
| Gangguan | `dhcrelay` mati | Discover terkirim 5 kali tanpa balasan Offer; tanpa IPv4 dan tanpa gateway |
| Pulih | `dhcrelay` aktif kembali | `192.227.2.10` (Pool 1), gateway `192.227.2.1` |

**Kesimpulan:** DHCPDISCOVER adalah broadcast dan tidak dapat melewati router. Selama relay mati, DHCP server tetap berjalan tetapi tidak menerima request apa pun dari Network 2. Setelah relay dipulihkan, client langsung mendapat alamat yang sama dari pool yang benar, karena kebijakan class di tengen tidak berubah.

![Soal 9](assets/soal9.png)

---

## Soal 10 – Autostart setelah restart

### Tujuan
DHCP server di tengen dan DHCP relay di jichi harus menyala otomatis setelah node di-restart, dan seluruh kebijakan DHCP harus tetap berlaku.

### Langkah
Karena tidak ada systemd (lihat [Catatan Lingkungan GNS3](#catatan-lingkungan-gns3)), autostart dibangun dengan cara berikut ([`scripts/jichi/soal10.sh`](scripts/jichi/soal10.sh)):

1. **`/etc/network/autostart.sh`** memasang IP (`soal1.sh`), lalu menyalakan layanan sesuai node:
    - jichi menjalankan `soal3.sh` (relay);
    - tengen menunggu koneksi internet tersedia (diperlukan bila paket harus diinstal ulang), lalu menjalankan `soal7.sh` (konfigurasi DHCP lengkap dari soal 4–7).
2. **Hook boot di `/etc/network/interfaces`.** File ini dibaca GNS3 lewat `ifup -a` setiap container start:

    ```
    auto lo
    iface lo inet manual
        up setsid /etc/network/autostart.sh > /var/log/autostart.log 2>&1 < /dev/null &
    ```

3. `update-rc.d isc-dhcp-relay defaults` (jichi) dan `update-rc.d isc-dhcp-server defaults` (tengen), sebagai cara standar Debian untuk mengaktifkan service saat boot.

Isi `autostart.sh`:

```bash
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
    for i in $(seq 1 60); do ping -c1 -W1 8.8.8.8 >/dev/null 2>&1 && break; sleep 2; done
    bash /root/soal7.sh
    ;;
esac
echo "===== autostart selesai $(date) ====="
```

#### Mengapa `inet manual`, bukan `inet loopback`
Percobaan pertama memakai `iface lo inet loopback`, dan setelah restart autostart **tidak berjalan**:

```
root@jichi:~# ps -o lstart= -p 1; ip -br addr; ip route | grep default; ps aux | grep [d]hcrelay; tail -5 /var/log/autostart.log
Wed Oct 14 23:11:21 2026
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UNKNOWN        fe80::42:dfff:fe69:7000/64
eth1             UNKNOWN        fe80::42:dfff:fe69:7001/64
eth2             UNKNOWN        fe80::42:dfff:fe69:7002/64
eth3             UNKNOWN        fe80::42:dfff:fe69:7003/64
eth4             UNKNOWN        fe80::42:dfff:fe69:7004/64
tail: cannot open '/var/log/autostart.log' for reading: No such file or directory
```

Dengan metode `loopback`, busybox `ifup` mencoba menambahkan `127.0.0.1` ke `lo`. Alamat itu sudah ada, sehingga langkah tersebut gagal, interface dianggap gagal naik, dan perintah `up` (hook autostart) tidak pernah dijalankan. Setelah diganti menjadi `inet manual`, busybox tidak melakukan konfigurasi alamat dan langsung menjalankan perintah `up`.

```bash
sed -i 's/^iface lo inet loopback$/iface lo inet manual/' /etc/network/interfaces
```

### Hasil Uji

Setelah jichi dan tengen di-restart (Stop, lalu Start), tanpa menjalankan perintah apa pun secara manual:

**jichi:**

```
Wed Oct 14 23:13:00 2026
eth0             UNKNOWN        192.168.122.121/24
eth1             UNKNOWN        192.227.1.1/24 fe80::42:dfff:fe69:7001/64
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth2             UNKNOWN        192.227.2.1/24 fe80::42:dfff:fe69:7002/64
eth3             UNKNOWN        192.227.3.1/24 fe80::42:dfff:fe69:7003/64
eth4             UNKNOWN        192.227.4.1/24 fe80::42:dfff:fe69:7004/64
default via 192.168.122.1 dev eth0 metric 87240
root  130  0.0  0.0  5988  3328 ?  Ss  23:13  0:00 dhcrelay -4 -id eth2 -id eth3 -iu eth4 192.227.4.10
```

**tengen:**

```
Wed Oct 14 23:13:12 2026
eth0             UNKNOWN        192.227.4.10/24 fe80::42:94ff:fe19:5e00/64
default via 192.227.4.1 dev eth0
Status of ISC DHCPv4 server: dhcpd is running.
...
===== autostart selesai Wed Oct 14 23:13:35 UTC 2026 =====
```

Proses relay mendapat **PID baru (130) dengan waktu start 23:13**, sama dengan waktu boot container. Sebelum restart, PID-nya 532 dengan waktu start 22:54. Ini membuktikan relay dinyalakan oleh autostart, bukan dijalankan manual.

Kebijakan DHCP setelah restart. Client meminta lease ulang, lalu `cek7.sh` dijalankan di tengen:

```
IP               MAC                 KLIEN      STATUS    POOL           HASIL
192.227.2.10     02:00:00:00:a1:01   itadori    active    .2.10-.2.35    OK
192.227.2.70     02:00:00:00:a2:01   zenin      active    .2.70-.2.95    OK
192.227.3.11     02:00:00:00:b3:02   kusakabe   active    .3.10-.3.89    OK
192.227.3.100    02:00:00:00:b3:03   miwa       active    .3.100-.3.199  OK

Bentrok alamat : TIDAK ADA (setiap IP & MAC aktif unik)
Fixed address  : .3.99 tidak dipakai pool dinamis (khusus kugisaki)
KESIMPULAN     : semua lease sesuai kebijakan pool
```

| Kebijakan | Client | Hasil setelah restart |
|---|---|---|
| Pool 1 | itadori | `192.227.2.10` ✅ |
| Pool 2 | zenin | `192.227.2.70` ✅ |
| Short lease | kusakabe | `192.227.3.11` ✅ |
| Long lease | miwa | `192.227.3.100` ✅ |
| Fixed address | kugisaki | `192.227.3.99` ✅ |

**Kesimpulan:** setelah restart, relay dan DHCP server menyala sendiri dengan IP dan gateway yang benar, dan seluruh kebijakan DHCP tetap berlaku.

> File `dhcpd.leases` ikut kosong setelah container dibuat ulang. Tabel `cek7.sh` baru terisi setelah client meminta lease kembali.

![Soal 10 - restart](assets/soal10-restart.png)
![Soal 10 - cek](assets/soal10-cek.png)

---

## Kendala & Solusi

| Kendala | Penyebab | Solusi |
|---|---|---|
| Error *Permission denied (privilege Project.Allocate is required)* di GNS3 | Akun yang dipakai tidak memiliki hak akses ke project | Login dengan akun kelompok yang memiliki akses ke project |
| Node menyala tanpa IPv4 (hanya `fe80::`) | `/etc/network/interfaces` masih berupa template bawaan yang seluruhnya dikomentari | IP dipasang melalui `soal1.sh`, dan dijalankan otomatis oleh autostart |
| Server tidak bisa menginstal paket setelah GNS3 dibuka ulang | jichi belum dikonfigurasi, sehingga gateway Network 4 tidak ada | Selalu konfigurasi jichi lebih dulu; autostart di tengen menunggu koneksi internet sebelum menginstal |
| Client `broadcasting discover` tanpa balasan pada pengujian soal 3 | jichi belum aktif kembali setelah restart, dan relay belum menandai interface upstream | Relay dijalankan dengan `-id eth2 -id eth3 -iu eth4` |
| `search k32.com` tidak muncul di client | `udhcpc` tidak meminta opsi 119 secara default | `udhcpc` dijalankan dengan `-O search -O domain` |
| Paste script panjang di PuTTY menjadi acak | Buffer terminal tidak sanggup menampung paste besar | Paste satu blok per kali, dan tunggu prompt kembali sebelum paste blok berikutnya |
| Autostart tidak berjalan setelah restart | `iface lo inet loopback` gagal di busybox `ifup`, sehingga hook `up` dilewati | Diganti menjadi `iface lo inet manual` |
| Tabel `cek7.sh` kosong setelah restart | File lease ikut kosong karena container dibuat ulang | Client diminta mengambil lease ulang sebelum pemeriksaan |
