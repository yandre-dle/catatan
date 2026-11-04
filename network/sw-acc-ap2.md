# Hardening Switch Akses SW-ACC-AP2 (Cisco Catalyst 2960)

Catatan ini berisi langkah-langkah mengamankan (*hardening*) switch akses **SW-ACC-AP2**, sebuah Cisco Catalyst 2960 dengan IOS 15.x. Switch akses adalah switch yang langsung terhubung ke perangkat pengguna (PC, printer, *access point*), sehingga paling sering menjadi sasaran serangan dari dalam jaringan: DHCP palsu, ARP *spoofing*, *MAC flooding*, *VLAN hopping*, dan *loop*.

- [Asumsi dan Variabel](#asumsi-dan-variabel)
- [Persiapan Sebelum Konfigurasi](#persiapan-sebelum-konfigurasi)
- [1. Identitas dan Banner](#1-identitas-dan-banner)
- [2. Akun dan Password](#2-akun-dan-password)
- [3. SSH dan Akses Manajemen](#3-ssh-dan-akses-manajemen)
- [4. Matikan Layanan yang Tidak Dipakai](#4-matikan-layanan-yang-tidak-dipakai)
- [5. VLAN, VTP, dan Interface Manajemen](#5-vlan-vtp-dan-interface-manajemen)
- [6. Uplink Trunk](#6-uplink-trunk)
- [7. Port Akses Pengguna](#7-port-akses-pengguna)
- [8. Port Access Point](#8-port-access-point)
- [9. Port yang Tidak Dipakai](#9-port-yang-tidak-dipakai)
- [10. Spanning Tree](#10-spanning-tree)
- [11. DHCP Snooping, DAI, dan IP Source Guard](#11-dhcp-snooping-dai-dan-ip-source-guard)
- [12. Waktu, Logging, dan SNMPv3](#12-waktu-logging-dan-snmpv3)
- [13. Simpan dan Verifikasi](#13-simpan-dan-verifikasi)
- [Checklist Ringkas](#checklist-ringkas)

## Asumsi dan Variabel

Sesuaikan nilai berikut dengan jaringan Anda sebelum menyalin perintah:

| Variabel | Contoh | Keterangan |
|---|---|---|
| Hostname | `SW-ACC-AP2` | Nama switch |
| VLAN data pengguna | `10` | PC / printer |
| VLAN voice | `20` | Telepon IP (opsional) |
| VLAN AP / wireless | `30` | Manajemen *access point* |
| VLAN manajemen | `99` | IP switch, SSH, SNMP |
| VLAN native trunk | `998` | VLAN kosong, tidak dipakai host |
| VLAN *blackhole* | `999` | Untuk port yang tidak dipakai |
| IP switch | `10.10.99.12/24` | Di VLAN 99 |
| Gateway | `10.10.99.1` | Di switch distribusi/router |
| Jaringan admin | `10.10.99.0/24` | Satu-satunya yang boleh SSH/SNMP |
| Server NTP / syslog | `10.10.99.1` / `10.10.99.20` | |
| Uplink ke distribusi | `Gi0/1`, `Gi0/2` | |
| Port pengguna | `Fa0/1 - 18` | |
| Port access point | `Fa0/19 - 20` | |
| Port tidak dipakai | `Fa0/21 - 24` | |

> Catatan:
> Penamaan interface di atas untuk 2960 klasik (24 port FastEthernet + 2 GigabitEthernet). Pada 2960-X / 2960-XR / 2960-Plus, nama interface biasanya `GigabitEthernet1/0/1` dst. Cek dengan `show ip interface brief`.

## Persiapan Sebelum Konfigurasi

1. **Cek versi IOS dan *feature set*.** SSH dan SNMPv3 dengan enkripsi butuh image **crypto (K9)**. DAI dan IP Source Guard butuh *feature set* **LAN Base** (tidak tersedia di LAN Lite).

   ```terminal
   SW-ACC-AP2# show version
   ```

   Cari tulisan `K9` pada nama image (misalnya `c2960-lanbasek9-mz...`).

2. **Backup konfigurasi saat ini.**

   ```terminal
   SW-ACC-AP2# copy running-config flash:backup-sebelum-hardening.cfg
   SW-ACC-AP2# copy running-config tftp:
   ```

3. **Pasang jaring pengaman.** Jika konfigurasi lewat SSH/telnet, jadwalkan reload otomatis supaya switch kembali ke konfigurasi lama bila kita terkunci di luar. **Jangan** `write memory` sebelum semuanya terverifikasi.

   ```terminal
   SW-ACC-AP2# reload in 15
   ```

   Jika semua sudah berjalan normal, batalkan:

   ```terminal
   SW-ACC-AP2# reload cancel
   ```

4. Sebaiknya kerjakan lewat **kabel console**, terutama untuk langkah 3 (SSH/ACL) dan langkah 11 (DHCP snooping/DAI).

Semua perintah di bawah dijalankan dari mode konfigurasi global (`configure terminal`).

## 1. Identitas dan Banner

```cisco
hostname SW-ACC-AP2
banner login ^
PERINGATAN: Hanya untuk pengguna yang berwenang.
Semua aktivitas dicatat dan dipantau.
^
```

Banner tidak boleh berisi informasi sensitif (nama perusahaan, model perangkat, IP).

## 2. Akun dan Password

```cisco
service password-encryption
security passwords min-length 12

enable algorithm-type scrypt secret <PASSWORD-ENABLE>
username admin privilege 15 algorithm-type scrypt secret <PASSWORD-ADMIN>

login block-for 120 attempts 3 within 60
login on-failure log
login on-success log
```

- `algorithm-type scrypt` (type 9) tersedia di IOS 15.2(x)E ke atas. Jika perintah ditolak, gunakan `enable secret <PASSWORD>` dan `username admin privilege 15 secret <PASSWORD>` (type 5).
- **Jangan** pakai `enable password` atau `username ... password` karena enkripsinya (type 7) mudah dibongkar.
- `login block-for` memblokir login selama 120 detik jika gagal 3 kali dalam 60 detik (perlindungan *brute force*).

Aktifkan AAA lokal:

```cisco
aaa new-model
aaa authentication login default local
aaa authorization exec default local
```

> Peringatan:
> Setelah `aaa new-model`, console dan VTY langsung meminta username/password lokal. Pastikan `username admin` sudah dibuat **sebelum** perintah ini.

Amankan console:

```cisco
line con 0
 exec-timeout 5 0
 logging synchronous
 login authentication default
```

## 3. SSH dan Akses Manajemen

Buat kunci RSA dan aktifkan SSH versi 2:

```cisco
ip domain-name lab.local
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
```

Batasi siapa yang boleh SSH dengan ACL:

```cisco
ip access-list standard ACL-MGMT
 permit 10.10.99.0 0.0.0.255
 deny   any log

line vty 0 15
 access-class ACL-MGMT in
 transport input ssh
 exec-timeout 10 0
 login authentication default
```

`transport input ssh` mematikan telnet, yang mengirim password dalam bentuk teks biasa.

## 4. Matikan Layanan yang Tidak Dipakai

```cisco
no ip http server
no ip http secure-server
no service pad
no ip source-route
no ip domain-lookup
no service config
service tcp-keepalives-in
service tcp-keepalives-out
```

- Web UI (`ip http`) adalah celah yang sering dieksploitasi. Kelola switch lewat SSH saja.
- `no ip domain-lookup` mencegah switch "hang" mencari DNS ketika salah ketik perintah.
- `tcp-keepalives` menutup sesi SSH yang menggantung.

**CDP/LLDP** membocorkan informasi perangkat (model, IOS, IP) ke siapa pun yang terhubung. Di switch akses, **matikan per port pengguna** (lihat langkah 7), tetapi **biarkan aktif di uplink dan port AP**: AP Cisco memakai CDP untuk negosiasi daya PoE.

## 5. VLAN, VTP, dan Interface Manajemen

Gunakan VTP *transparent* supaya database VLAN switch ini tidak bisa ditimpa oleh switch lain (serangan/kecelakaan VTP):

```cisco
vtp mode transparent

vlan 10
 name DATA
vlan 20
 name VOICE
vlan 30
 name AP-MGMT
vlan 99
 name MGMT
vlan 998
 name NATIVE-UNUSED
vlan 999
 name BLACKHOLE
```

Pindahkan manajemen dari VLAN 1 ke VLAN 99, lalu matikan VLAN 1:

```cisco
interface Vlan99
 description Manajemen SW-ACC-AP2
 ip address 10.10.99.12 255.255.255.0
 no ip redirects
 no ip proxy-arp
 no ip unreachables
 no shutdown

interface Vlan1
 no ip address
 shutdown

ip default-gateway 10.10.99.1
```

2960 adalah switch layer 2, jadi gateway diatur dengan `ip default-gateway`, bukan `ip route`.

## 6. Uplink Trunk

```cisco
interface range GigabitEthernet0/1 - 2
 description UPLINK ke SW-DIST
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 998
 switchport trunk allowed vlan 10,20,30,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
```

- `switchport nonegotiate` mematikan DTP sehingga perangkat lain tidak bisa "memaksa" port menjadi trunk.
- Native VLAN diganti ke VLAN kosong (998) dan `allowed vlan` hanya berisi VLAN yang diperlukan, untuk mencegah *VLAN hopping* (*double tagging*).
- 2960 hanya mendukung 802.1Q, jadi tidak perlu `switchport trunk encapsulation dot1q`.
- Native VLAN dan `allowed vlan` **harus sama** di sisi switch distribusi.

Opsional (jika didukung versi IOS): tag juga native VLAN di semua trunk.

```cisco
vlan dot1q tag native
```

## 7. Port Akses Pengguna

```cisco
interface range FastEthernet0/1 - 18
 description USER
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 3
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport port-security aging time 1440
 switchport port-security aging type inactivity
 spanning-tree portfast
 spanning-tree bpduguard enable
 storm-control broadcast level 5.00
 storm-control multicast level 5.00
 storm-control action trap
 ip dhcp snooping limit rate 15
 no cdp enable
 no lldp transmit
 no lldp receive
 no shutdown
```

Penjelasan:

- `switchport mode access` + `nonegotiate`: port tidak akan pernah menjadi trunk.
- **Port security**: membatasi jumlah MAC per port (mencegah *MAC flooding* dan switch/hub liar). `maximum 3` cukup untuk PC + telepon IP (telepon terhitung di VLAN voice). Tanpa telepon IP, hapus `voice vlan` dan gunakan `maximum 1` atau `2`.
- `violation restrict`: frame dari MAC asing dibuang dan dicatat, port tetap hidup. Gunakan `shutdown` jika ingin lebih ketat.
- `spanning-tree portfast`: port langsung aktif (cocok untuk host). Pada IOS 15.2(2)E ke atas, perintahnya `spanning-tree portfast edge`.
- **BPDU guard**: port langsung *err-disabled* jika ada yang mencolokkan switch, mencegah *loop* dan serangan STP.
- **Storm control**: membatasi badai broadcast/multicast hingga 5% bandwidth.

Pulihkan otomatis port yang *err-disabled* setelah 5 menit:

```cisco
errdisable recovery cause bpduguard
errdisable recovery cause psecure-violation
errdisable recovery cause dhcp-rate-limit
errdisable recovery cause arp-inspection
errdisable recovery interval 300
```

## 8. Port Access Point

Konfigurasi port AP tergantung mode AP-nya.

**AP mode *local* (semua trafik ditunnel ke WLC)**: cukup port akses di VLAN AP.

```cisco
interface range FastEthernet0/19 - 20
 description AP
 switchport mode access
 switchport access vlan 30
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 storm-control broadcast level 5.00
 storm-control action trap
 ip dhcp snooping limit rate 15
 no shutdown
```

**AP mode *FlexConnect* / autonomous (SSID di-*bridge* ke VLAN lokal)**: port harus trunk. Native VLAN di sini **sengaja** diisi VLAN AP (30) karena AP membutuhkan VLAN manajemen tanpa tag. Ini pengecualian dari aturan native VLAN di langkah 6.

```cisco
interface range FastEthernet0/19 - 20
 description AP-FLEX
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 30
 switchport trunk allowed vlan 10,20,30
 spanning-tree portfast trunk
 spanning-tree bpduguard enable
 storm-control broadcast level 5.00
 storm-control action trap
 ip dhcp snooping limit rate 50
 no shutdown
```

Pada IOS 15.2(2)E ke atas, `spanning-tree portfast trunk` diganti `spanning-tree portfast edge trunk`. CDP tetap aktif di port AP (untuk PoE).

## 9. Port yang Tidak Dipakai

Semua port kosong dimatikan dan dipindah ke VLAN *blackhole*, sehingga mencolokkan kabel ke port tersebut tidak memberi akses apa pun:

```cisco
interface range FastEthernet0/21 - 24
 description UNUSED
 switchport mode access
 switchport access vlan 999
 switchport nonegotiate
 no cdp enable
 shutdown
```

Saat port akan dipakai, ubah konfigurasinya mengikuti langkah 7, jangan sekadar `no shutdown`.

## 10. Spanning Tree

```cisco
spanning-tree mode rapid-pvst
spanning-tree loopguard default
spanning-tree portfast bpduguard default
spanning-tree extend system-id
```

- Rapid PVST+ mempercepat konvergensi.
- `loopguard default` mencegah *loop* bila uplink berhenti menerima BPDU (misalnya kabel fiber satu arah).
- `portfast bpduguard default` mengaktifkan BPDU guard otomatis di semua port portfast.
- Switch akses **tidak boleh** menjadi root bridge. Pastikan root berada di switch distribusi (`spanning-tree vlan 10,20,30,99 root primary` di sana), dan pasang `spanning-tree guard root` di port distribusi yang mengarah ke switch ini.

## 11. DHCP Snooping, DAI, dan IP Source Guard

**DHCP snooping** memblokir server DHCP liar. Hanya uplink (sudah diberi `ip dhcp snooping trust` di langkah 6) yang boleh mengirim balasan DHCP.

```cisco
ip dhcp snooping
ip dhcp snooping vlan 10,20,30
no ip dhcp snooping information option
```

`no ip dhcp snooping information option` diperlukan bila server/relay DHCP menolak paket berisi *option 82* (gejala: klien tidak mendapat IP setelah snooping aktif).

**Dynamic ARP Inspection (DAI)** mencegah ARP *spoofing*/*man-in-the-middle* dengan mencocokkan ARP terhadap tabel DHCP snooping:

```cisco
ip arp inspection vlan 10,20,30
ip arp inspection validate src-mac dst-mac ip
```

> Peringatan:
> Host dengan **IP statis** (printer, server, AP dengan IP statis) tidak ada di tabel DHCP snooping, sehingga ARP-nya akan dibuang DAI. Buat ARP ACL untuk host tersebut sebelum mengaktifkan DAI:
>
> ```cisco
> arp access-list ARP-STATIS
>  permit ip host 10.10.10.50 mac host aaaa.bbbb.cccc
> ip arp inspection filter ARP-STATIS vlan 10
> ```

**IP Source Guard** (opsional, paling ketat) memblokir trafik dari IP yang tidak sesuai tabel DHCP snooping. Pasang di port pengguna:

```cisco
interface range FastEthernet0/1 - 18
 ip verify source
```

DAI dan IP Source Guard butuh *feature set* LAN Base. Jika perintah ditolak, cek `show version` dan template SDM (`show sdm prefer`).

Aktifkan langkah ini secara bertahap (snooping dulu, cek `show ip dhcp snooping binding`, baru DAI, lalu IP Source Guard), dan lakukan di luar jam kerja.

## 12. Waktu, Logging, dan SNMPv3

Waktu yang akurat penting agar log bisa dipakai untuk investigasi:

```cisco
clock timezone WIB 7 0
ntp authentication-key 1 md5 <KUNCI-NTP>
ntp authenticate
ntp trusted-key 1
ntp server 10.10.99.1 key 1
ntp source Vlan99

service timestamps log datetime msec localtime show-timezone
service timestamps debug datetime msec localtime show-timezone
```

Kirim log ke server syslog dan simpan juga di memori:

```cisco
logging buffered 64000 informational
logging console critical
logging trap informational
logging source-interface Vlan99
logging host 10.10.99.20
```

Catat setiap perubahan konfigurasi (siapa mengubah apa):

```cisco
archive
 log config
  logging enable
  notify syslog contenttype plaintext
  hidekeys
```

Gunakan SNMPv3 (terautentikasi dan terenkripsi). Hapus community SNMPv1/v2c bawaan:

```cisco
no snmp-server community public
no snmp-server community private

ip access-list standard ACL-SNMP
 permit host 10.10.99.20
 deny   any log

snmp-server group NMS v3 priv access ACL-SNMP
snmp-server user nmsuser NMS v3 auth sha <PASSWORD-AUTH> priv aes 128 <PASSWORD-PRIV>
snmp-server location <LOKASI>
snmp-server contact <KONTAK-ADMIN>
snmp-server enable traps port-security
snmp-server enable traps storm-control
snmp-server host 10.10.99.20 version 3 priv nmsuser
```

## 13. Simpan dan Verifikasi

Jalankan perintah verifikasi berikut. Jika semua normal, batalkan reload dan simpan:

```terminal
SW-ACC-AP2# show running-config
SW-ACC-AP2# show ip interface brief
SW-ACC-AP2# show interfaces status
SW-ACC-AP2# show interfaces trunk
SW-ACC-AP2# show vlan brief
SW-ACC-AP2# show ip ssh
SW-ACC-AP2# show port-security
SW-ACC-AP2# show spanning-tree summary
SW-ACC-AP2# show ip dhcp snooping
SW-ACC-AP2# show ip dhcp snooping binding
SW-ACC-AP2# show ip arp inspection statistics
SW-ACC-AP2# show errdisable recovery
SW-ACC-AP2# show ntp status
SW-ACC-AP2# show logging

SW-ACC-AP2# reload cancel
SW-ACC-AP2# copy running-config startup-config
```

Uji juga dari sisi pengguna:

- Login SSH dari jaringan admin berhasil; dari jaringan lain ditolak; telnet ditolak.
- PC di port pengguna mendapat IP DHCP dan bisa mengakses jaringan.
- AP mendapat daya PoE, bergabung ke WLC, dan klien wireless mendapat IP.
- Mencolokkan switch kecil ke port pengguna membuat port *err-disabled* (BPDU guard).

## Checklist Ringkas

| No | Item | Perintah cek |
|---|---|---|
| 1 | Backup konfigurasi lama tersimpan | `dir flash:` |
| 2 | Password memakai `secret` (type 9/5), tidak ada `password` biasa | `show run \| include password\|secret` |
| 3 | AAA lokal dan `login block-for` aktif | `show login` |
| 4 | Hanya SSH v2, VTY dibatasi ACL | `show ip ssh`, `show run \| section line vty` |
| 5 | HTTP server mati | `show run \| include http` |
| 6 | VTP transparent | `show vtp status` |
| 7 | Manajemen di VLAN 99, VLAN 1 shutdown | `show ip interface brief` |
| 8 | Trunk: nonegotiate, native VLAN 998, allowed VLAN terbatas | `show interfaces trunk` |
| 9 | Port akses: mode access, port-security, portfast, BPDU guard | `show port-security` |
| 10 | Port tidak dipakai: VLAN 999 dan shutdown | `show interfaces status` |
| 11 | Rapid PVST+, loopguard, switch ini bukan root | `show spanning-tree summary` |
| 12 | DHCP snooping, DAI (dan IP Source Guard) aktif | `show ip dhcp snooping`, `show ip arp inspection` |
| 13 | NTP sinkron, log terkirim ke syslog | `show ntp status`, `show logging` |
| 14 | SNMPv3 saja, tanpa community `public`/`private` | `show snmp user` |
| 15 | Konfigurasi disimpan | `copy running-config startup-config` |

Referensi:
- [Cisco Guide to Harden Cisco IOS Devices](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/13608-21.html)
- [Catalyst 2960 Software Configuration Guide](https://www.cisco.com/c/en/us/support/switches/catalyst-2960-series-switches/products-installation-and-configuration-guides-list.html)
