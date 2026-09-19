## LAPORAN RESMI PRAKTIKUM KOMUNIKASI DATA DAN JARINGAN KOMPUTER
MODUL 1: SERIAL EXPERIMENTS LAIN (THE WIRED)

##### Kelompok: K-053

### PENDAHULUAN / LATAR BELAKANG
Praktikum Modul 1 Komunikasi Data dan Jaringan Komputer mengangkat tema Serial Experiments Lain, di mana arsitektur jaringan dibangun untuk merepresentasikan sistem The Wired. Praktikum ini berfokus pada konfigurasi dasar router, manajemen antarmuka jaringan (interface), pengaturan NAT/masquerade, firewall, DHCP/DNS resolver, analisis trafik menggunakan Wireshark, konfigurasi FTP server, layanan Telnet, pemindaian port (port scanning) dengan Netcat, pengamanan akses jarak jauh menggunakan SSH Key-based Authentication, serta analisis berbagai file packet capture (pcap) untuk investigasi keamanan siber (analisis brute-force, USB HID keystroke, FTP theft, C2 traffic, SMB transfer, SMTP threat, dan dekripsi TLS).



## Laporan

### 1. Topologi & Konfigurasi Awal Interface

Untuk mempersiapkan pembangunan The Wired, Lain yang berperan sebagai Router membuat tiga Switch/Gateway: Switch 1 menuju dua entitas yaitu Alice dan Mika, Switch 2 menuju Chisa, sedangkan Switch 3 menuju Knights dan Eiri. Kelima entitas tersebut dikonfigurasi sebagai Client di GNS3.
#### Topologi
Router **Lain** membuat tiga Switch/Gateway: Switch 1 menuju dua entitas (**Alice** dan **Mika**), Switch 2 menuju **Chisa**, dan Switch 3 menuju dua entitas (**Knights** dan **Eiri**). Kelima entitas tersebut dikonfigurasi sebagai Client di GNS3, dengan prefix IP kelompok `10.190.x.x`.

**Switch 1**: Menghubungkan Alice dan Mika (Subnet 10.190.1.0/24).
**Switch 2**: Menghubungkan Chisa (Subnet 10.190.2.0/24).
**Switch 3**: Menghubungkan Knights dan Eiri (Subnet 10.190.3.0/24).

![](assets/Topologi.png)

**Lain**

```
auto eth0
iface eth0 inet dhcp
auto eth1
iface eth1 inet static
    address 10.190.1.1
    netmask 255.255.255.0
auto eth2
iface eth2 inet static
    address 10.190.2.1
    netmask 255.255.255.0
auto eth3
iface eth3 inet static
    address 10.190.3.1
    netmask 255.255.255.0
```

**Alice**

```
auto eth0
iface eth0 inet static
    address 10.190.1.2
    netmask 255.255.255.0
    gateway 10.190.1.1
```

**Mika**

```
auto eth0
iface eth0 inet static
    address 10.190.1.3
    netmask 255.255.255.0
    gateway 10.190.1.1
```

**Chisa**

```
auto eth0
iface eth0 inet static
    address 10.190.2.2
    netmask 255.255.255.0
    gateway 10.190.2.1
```

**Knights**

```
auto eth0
iface eth0 inet static
    address 10.190.3.2
    netmask 255.255.255.0
    gateway 10.190.3.1
```

**Eiri**

```
auto eth0
iface eth0 inet static
    address 10.190.3.3
    netmask 255.255.255.0
    gateway 10.190.3.1
```



### 2. Router Lain Terhubung ke Internet Publik (NAT/DHCP)

Karena menurut Lain pada saat itu The Wired masih terisolasi dari dunia luar, router Lain dikonfigurasikan agar dapat tersambung langsung ke jaringan internet publik melalui NAT/DHCP pada interface `eth0`.

```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
sysctl -w net.ipv4.ip_forward=1
```

**Verifikasi:**

```
ping -c 2 google.com
```
![alt text](assets/Screenshot%202026-09-17%20231123.png)

### 3. Konektivitas Antar-Client Melalui Routing

Setelah router Lain terhubung ke internet, seluruh entitas (Client) di bawah Switch 1, Switch 2, dan Switch 3 dapat saling terhubung dan berkomunikasi satu sama lain melalui konfigurasi routing.

Aktifkan IP Forwarding di router Lain:

```
sysctl -w net.ipv4.ip_forward=1
```

**Pembuktian — Alice (Subnet 1) ke Chisa (Subnet 2):**

```
ping -c 2 10.190.2.2
```

![](assets/Screenshot%202026-09-17%20231819.png)

**Pembuktian — Knights (Subnet 3) ke Mika (Subnet 1):**

```
ping -c 2 10.190.1.3
```
![](assets/Screenshot%202026-09-17%20231833.png)

### 4. Kemandirian Client ke Internet (NAT Masquerade + DNS)

Lain ingin agar setiap entitas (Client) memiliki kemandirian di The Wired. Firewall/iptables (NAT Masquerade) dan DNS resolver dikonfigurasikan agar setiap client dapat terhubung ke internet secara mandiri.

```
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth3 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Pada masing-masing client:

```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

**Verifikasi:**

```
ping -c 2 8.8.8.8
ping -c 2 google.com
```

![alt text](assets/Screenshot%202026-09-16%20083650.png)

### 5. Persistensi Konfigurasi Setelah Restart

Eiri tetap berupaya menanamkan kekacauan ke dalam jaringan. Untuk mengantisipasi restart tiba-tiba, seluruh konfigurasi jaringan dipastikan tidak hilang saat semua node direstart dengan menambahkan konfigurasi langsung ke `/etc/network/interfaces` pada router Lain, serta membuat script verifikasi.

**Menambahkan config di Lain:**

```
auto eth0
iface eth0 inet dhcp
auto eth1
iface eth1 inet static
    address 10.190.1.1
    netmask 255.255.255.0
auto eth2
iface eth2 inet static
    address 10.190.2.1
    netmask 255.255.255.0
auto eth3
iface eth3 inet static
    address 10.190.3.1
    netmask 255.255.255.0
up sysctl -w net.ipv4.ip_forward=1
up iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

**Script verifikasi `nano /root/cek_status.sh`:**

```sh
#!/bin/sh
echo "=========================================="
echo "         RINGKASAN INTERFACE IP           "
echo "=========================================="
ip -br a

echo ""
echo "=========================================="
echo "          STATUS TABEL NAT (IPTABLES)     "
echo "=========================================="
iptables -t nat -L -v -n
```

```
chmod +x /root/cek_status.sh
/root/cek_status.sh
```
![](assets/Screenshot%202026-09-16%20085059.png)

### 6. Packet Sniffing DNS/ICMP pada Node Mika

Mika mencurigai adanya anomali traffic pada segmen jaringannya. Generator traffic dijalankan pada node Mika, lalu packet sniffing dilakukan menggunakan Wireshark pada interface node Mika dengan display filter khusus untuk paket berprotokol DNS atau ICMP.

**`nano /root/traffic_protocol7.sh`:**

```sh
#!/bin/sh
while true; do
    ping -c 1 10.190.1.1 > /dev/null 2>&1
    nslookup google.com 8.8.8.8 > /dev/null 2>&1
    nc -z -w 1 10.190.1.1 80 > /dev/null 2>&1
    sleep 2
done
```

```
chmod +x /root/traffic_protocol7.sh
/root/traffic_protocol7.sh &
```

Display filter Wireshark: `dns || icmp`

![](assets/Screenshot%202026-09-16%20085555.png)
![](assets/Screenshot%202026-09-16%20085616.png)

File pcap no.6 [disini](File_pcap/soal_6.pcapng)

### 7. FTP Server di Chisa dengan Kebijakan Akses per User

Chisa memutuskan mendirikan FTP Server pada node miliknya dengan shared folder di `/var/wired/data`. Kebijakan akses yang diterapkan: user `alice` (read & write), user `mika` (read-only), dan user `eiri` (tanpa izin akses / blacklist).

**Konfigurasi di Chisa:**

```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apt update && apt install -y --allow-unauthenticated vsftpd

mkdir -p /var/wired/data
chmod 777 /var/wired/data
chown -R ftp:nogroup /var/wired/data

useradd -m -s /bin/ alice 2>/dev/null
useradd -m -s /bin/ mika 2>/dev/null
useradd -m -s /bin/ eiri 2>/dev/null

echo "alice:password" | chpasswd
echo "mika:password" | chpasswd
echo "eiri:password" | chpasswd

cat << 'EOF' > /etc/vsftpd.conf
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
check_shell=NO
local_root=/var/wired/data
chroot_local_user=YES
allow_writeable_chroot=YES
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO
user_config_dir=/etc/vsftpd_user_conf
EOF

echo -e "alice\nmika" > /etc/vsftpd.userlist

mkdir -p /etc/vsftpd_user_conf
echo "write_enable=NO" > /etc/vsftpd_user_conf/mika

service vsftpd restart
service vsftpd status
```

`userlist_deny=NO` bersama `userlist_file` yang hanya memuat `alice` dan `mika` membuat kedua user tersebut menjadi satu-satunya yang diizinkan login FTP, sehingga user `eiri` otomatis ditolak (blacklist secara implisit).

**Pembuktian Alice (read & write), membuat `signal_alice.txt`:**

```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apt install -y --allow-unauthenticated ftp
cd /root
touch signal_alice.txt
ftp 10.190.2.2
```

![](assets/Screenshot%202026-09-17%20235516.png)

**Pembuktian Mika (read-only):**

```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apt update --fix-missing
apt install -y --allow-unauthenticated ftp
cd /root
touch test_mika.txt
ftp 10.190.2.2
```
![alt text](assets/Screenshot%202026-09-17%20234717.png)


**Pembuktian Eiri (ditolak akses):**

```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apt update --fix-missing
apt install -y --allow-unauthenticated ftp
ftp 10.190.2.2
```
![alt text](assets/Screenshot%202026-09-17%20234753.png)


### 8. Upload FTP dari Knights ke Chisa via Akun Alice + Analisis Wireshark

Kelompok rahasia Knights perlu mengirimkan dokumen laporan intelijen ke FTP Server Chisa. Koneksi FTP client dilakukan dari node Knights ke FTP Server Chisa menggunakan akun `alice`, lalu sesi tersebut dianalisis dengan Wireshark.

Jangan lupa untuk memulai capture pada Knights

```
apt update --fix-missing
apt install -y --allow-unauthenticated ftp wget tshark
cd /root
wget --no-check-certificate '<link_file>' -O knights_report.txt

tshark -i eth0 -f "host 10.190.2.2" -w /tmp/soal8.pcap >/dev/null 2>&1 &

echo "nameserver 8.8.8.8" > /etc/resolv.conf
apt update && apt install -y ftp

ftp 10.190.2.2
# login sebagai alice
passive
put knights_report.txt
quit

pkill tshark
tshark -r /tmp/soal8.pcap -Y "ftp || ftp-data"
tshark -r /tmp/soal8.pcap -Y "ftp.response.code == 229"
```

![](assets/Screenshot%202026-09-16%20102450.png)
File pcap no 8 [disni](File_pcap/soal_8.pcapng)

**Analisis & Jawaban Soal 8:**

- Perintah FTP untuk Upload: STOR knights_report.txt (Terlihat pada baris nomor 33: Request: STOR knights_report.txt).
- Kode Status Sukses Server: 226 Transfer complete (Terlihat pada baris nomor 43: Response: 226 Transfer complete).
- Port Data TCP yang Dinegosiasikan pada Mode PASV: Saat mengetik perintah passive, koneksi berpindah ke mode Active (EPRT). Nan, untuk mendapatkan respon mode PASV, jalankan perintah berikut untuk melihat baris respon 227 Entering Passive Mode beserta portnya. 59535 (Didapat dari respon 229 Entering Extended Passive Mode (|||59535|))


### 9. Download File dari Chisa oleh Mika + Pembatasan Read-Only

Mika mengakses dokumen Protokol Tujuh dari FTP Server Chisa. Dari node Mika, file tersebut diunduh menggunakan akun `mika`, kemudian pembatasan read-only dibuktikan dengan mencoba mengunggah file baru dari akun `mika`.

**Di Chisa**  menyiapkan file:

```
cd /var/wired/data
wget --no-check-certificate '<link_file>' -O protocol7_manifesto.txt
chmod 644 /var/wired/data/protocol7_manifesto.txt
```
![alt text](assets/Screenshot%202026-09-16%20102337.png)

**Di Mika:**

```
cd /root
touch test_upload_mika.txt
ftp 10.190.2.2
# User: mika | Password: password
get protocol7_manifesto.txt
put test_upload_mika.txt
quit
```

Mika berhasil download namun gagal upload (550 Permission denied)
![alt text](assets/no9mika.png)

Upload dari akun `mika` ditolak server dengan pesan `550 Permission denied` karena konfigurasi `write_enable=NO` pada `/etc/vsftpd_user_conf/mika`, sesuai kebijakan read-only.

### 10. Uji Latensi Ping Knights → Chisa (77 Paket, 128 Byte, Interval 0.3s)

Knights melancarkan uji ketahanan koneksi ke server Chisa untuk menguji latensi jaringan The Wired.

```
ping -c 77 -s 128 -i 0.3 10.190.2.2
```

**1. Detail Header ICMP**

- Echo (Ping) Request (Knights → Chisa): **Type 8, Code 0**
- Echo (Ping) Reply (Chisa → Knights): **Type 0, Code 0**

**2. Statistik Transmisi & Packet Loss**

- Packets Transmitted: **77**
- Packets Received: **77**
- Packet Loss: **0%**

**3. Statistik Latensi Network (RTT)**

- Min: **0.219 ms**
- Avg: **0.449 ms**
- Max: **0.948 ms**

![](assets/Screenshot%202026-09-16%20102920.png)
![](assets/Screenshot%202026-09-16%20102944.png)
File pcap no.10 [disini](File_pcap/soal_10.pcapng)

### Soal 11: Analisis Kelemahan Telnet (Plaintext & Character at a Time)
Konfigurasi di Node Chisa:
```BASH
useradd -m -s /bin/bash phantom_user
echo "phantom_user:wired_ghost" | chpasswd
apt update && apt install -y telnetd openbsd-inetd
# Konfigurasi /etc/inetd.conf (aktifkan layanan telnet)
service openbsd-inetd restart
```
Pengujian dari Eiri:
```BASH
telnet 10.190.2.2
# Login: phantom_user / wired_ghost
```
Hasil Analisis Wireshark (Follow TCP Stream): Kredensial (phantom_user dan wired_ghost) terlihat jelas dalam bentuk plain text. Setiap karakter yang diketik dikirim dalam segmen TCP terpisah karena Telnet menggunakan mode character-at-a-time.

Soal 12: Port Scanning dengan Netcat & Analisis TCP Flag
Konfigurasi Layanan di Knights:
```BASH
apt update && apt install -y openssh-server apache2
service ssh start
service apache2 start
```
Eksekusi Port Scan dari Alice:
```BASH
nc -zv 10.190.3.2 22
nc -zv 10.190.3.2 80
nc -zv 10.190.3.2 7777
```
Analisis Wireshark:
Port Terbuka (22 & 80): Server merespons paket SYN dari Alice dengan flag SYN-ACK.
Port Tertutup (7777): Server merespons dengan flag RST-ACK (Reset-Acknowledgment).

Soal 13: Konfigurasi SSH Aman dengan Public Key Authentication
Konfigurasi Server (Knights):
```BASH
apt update && apt install -y openssh-server
useradd -m -s /bin/bash mika_admin
passwd mika_admin
# Ubah /etc/ssh/sshd_config: PubkeyAuthentication yes, PasswordAuthentication no
service ssh restart
```
Generate Key Pair di Mika:
```BASH
ssh-keygen -t rsa -b 2048
ssh-copy-id mika_admin@10.190.3.2
```
Pengujian & Analisis: Koneksi berhasil tanpa password. Pada Wireshark (filter ssh), terlihat proses Protocol Version Exchange dan Key Exchange (KEX). Setelah negosiasi kunci, seluruh sesi terenkripsi end-to-end, mencegah kebocoran kredensial layaknya Telnet.

Soal 14: Investigasi Brute-Force Serangan Web (wired_bruteforce.pcapng)
Kronologi & Langkah Pengerjaan:
1. Membuka dan menganalisis berkas packet capture forensik menggunakan Wireshark untuk melacak anomali trafik HTTP/TCP pada port 8080.
2. Mengidentifikasi alamat IP penyerang (172.26.7.50 atas nama Eiri) yang melakukan percobaan login berulang kali ke server target (172.26.7.100).
3. Menemukan kata sandi akun lain_admin yang berhasil dibobol, yaitu wired_pr0tocol_7, serta mendeteksi versi web server yang digunakan (Apache/2.4.62).
4. Memasukkan jawaban investigasi ke socket server validasi kelompok untuk mendapatkan flag pengerjaan.

Validasi Socket Server:
```BASH
nc [IP_Group] 3401
```
Flag: KOMJAR26{W1r3d_Brut3_ofGWOszZFdRWl2gbaXEJsvORj}

Soal 15: Investigasi Perangkat USB HID / Rubber Ducky (wired_usb_hid.pcap)
Kronologi & Langkah Pengerjaan:
1. Menganalisis paket USB packet capture untuk menyelidiki anomali perangkat Human Interface Device (HID) yang terdeteksi menancap pada sistem.
2. Mengidentifikasi atribut perangkat berdasarkan Vendor ID (0x046d milik Logitech K120 Keyboard) dan alamat USB device (7).
3. Menggunakan pustaka skrip Python berbasis pyshark untuk mengekstrak data keystroke mentah dari laporan interupsi USB.
4. Menggabungkan deretan karakter hasil ekstraksi yang membentuk string pesan rahasia: wiredprotocol7isalive2026.
5. Mengirimkan data ke socket server validasi untuk memperoleh flag.

Validasi Socket Server:
```BASH
nc [IP_Group] 3402
```
Flag: KOMJAR26{USB_K3ystr0k3_zWwnYRYEEQXKwDvQBtrYQ0mHf}

Soal 16: Investigasi Pencurian Data via FTP (wired_ftp_theft.pcap)
Kronologi & Langkah Pengerjaan:
1. Melakukan inspeksi pada file packet capture aktivitas protokol FTP untuk melacak pencurian file sensitif.
2. Mengidentifikasi IP server FTP korban (198.51.100.7), banner aplikasi (vsftpd 3.0.5), serta kredensial akun penyerang (knights_agent dengan sandi N4v1_s3cur3_2026).
3. Membaca respons perintah FTP Size (213 524288) untuk mengetahui ukuran persis file eksfiltrasi knights_payload.exe yaitu sebesar 524288 bytes.
4. Menginput hasil investigasi ke socket server kelompok guna mengklaim flag.

Validasi Socket Server:
```BASH
nc [IP_Group] 3403
```
Flag: KOMJAR26{FTP_Th3ft_R2ezQCoEsXG66qwXkkMeQTanq}

Soal 17: Investigasi Command & Control (C2) HTTP (wired_http_c2.pcap)
Kronologi & Langkah Pengerjaan:
Memeriksa file packet capture komunikasi HTTP untuk mendeteksi adanya aktivitas komunikasi Command and Control (C2) malware.
Menelusuri permintaan DNS dan HTTP GET untuk menemukan nama domain pengendali (wired-update.net), alamat IP server (203.0.113.42), nama file muatan jahat yang diunduh (navi_agent.exe), serta kode status HTTP sukses (200 OK).
Melakukan verifikasi jawaban ke socket server kelompok untuk mendapatkan flag.

Validasi Socket Server:
```BASH
nc [IP_Group] 3404
```
Flag: KOMJAR26{Navi_C2_D0wnl04d_uApWZjAmB2PSUAqwE8pqLXhom}

Soal 18: Investigasi Transfer Malware via SMB (wired_smb_transfer.pcapng)
Kronologi & Langkah Pengerjaan:
Menganalisis protokol SMBv2 di dalam file packet capture untuk melacak pergerakan lateral file berbahaya antar komputer dalam jaringan lokal.
Mengidentifikasi alamat IP pengirim (10.7.3.100), IP penerima (10.7.1.50), struktur share folder tujuan (ADMIN$ / System32), serta nama file eksekusi malware (wired_trojan_payload.exe).
Memasukkan detail investigasi ke socket server validasi untuk memunculkan flag.

Validasi Socket Server:
```BASH
nc [IP_Group] 3405
```
Flag: KOMJAR26{SMB_Tr4nsf3r_Tssoap3oiw7cU1I0Eq7fldJSv}

Soal 19: Investigasi Ancaman Pemerasan SMTP Tanpa Enkripsi (wired_smtp_threat.pcap)
Kronologi & Langkah Pengerjaan:
Melakukan Follow TCP Stream pada port SMTP di dalam file packet capture untuk membaca isi email ancaman pemerasan (extortion/ransomware).
Menemukan alamat email korban (victim@protocol7.co.jp), kebocoran sandi akun (pr0tocol_7_user), jenis ancaman ransomware privat, batas waktu tebusan selama 3 hari (72 jam), serta nomor unik pengenal klien (MailClientID: 7719980706).
Menyubmit data temuan ke socket server validasi untuk klaim flag.

Validasi Socket Server:
```BASH
nc [IP_Group] 3406
```
Flag: KOMJAR26{SMTP_Ext0rt10n_ZFLfjUJOsNM9wPysEm9m3ZIlt}

Soal 20: Dekripsi Trafik TLS dengan Keylog (wired_tls_decrypt.pcapng)
Kronologi & Langkah Pengerjaan:
Mengonfigurasi Wireshark dengan memasukkan file kunci sesi pre-master secret (keyslogfile.txt) melalui menu preferensi protokol TLS (Edit > Preferences > Protocols > TLS).
Melakukan inspeksi ulang pada trafik terenkripsi yang kini berhasil didekripsi secara transparan.
Mengidentifikasi versi protokol TLS (TLSv1.2), Server Name Indication / SNI (example.com), alamat IP server HTTPS (93.184.216.34), User-Agent (curl/7.62.0), serta metode dan jalur request HTTP (HEAD /).
Mengirimkan rangkuman hasil dekripsi ke socket server kelompok untuk mendapatkan flag terakhir.

Validasi Socket Server:
```BASH
nc [IP_Group] 3407
```
Flag: KOMJAR26{TLS_D3crypt_cTtcuYV9AqEArZEauyuYNJzim}

KESIMPULAN
Praktikum Modul 1 Komunikasi Data dan Jaringan Komputer berhasil diselesaikan dengan baik. Seluruh konfigurasi topologi jaringan The Wired di GNS3 (routing, NAT masquerade, firewall, DNS) berjalan lancar. Layanan aplikasi seperti vsftpd, telnetd, openssh-server, dan apache2 berhasil dikonfigurasikan sesuai dengan batasan hak akses keamanan. Selain itu, kemampuan analisis forensik jaringan menggunakan Wireshark, Tshark, Netcat, dan pemecahan file packet capture (pcap) terbukti sangat penting dalam mengidentifikasi anomali trafik, kerentanan protokol lama (Telnet, FTP plain text), serta investigasi insiden keamanan siber.

Kendala
Tidak ada
