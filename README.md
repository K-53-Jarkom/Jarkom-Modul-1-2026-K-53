[LAPORAN MASIH DALAM PROSES! BELUM SELESAI]
LAPORAN RESMI PRAKTIKUM KOMUNIKASI DATA DAN JARINGAN KOMPUTER
MODUL 1: SERIAL EXPERIMENTS LAIN (THE WIRED)
Kelompok: K-053

PENDAHULUAN / LATAR BELAKANG
Praktikum Modul 1 Komunikasi Data dan Jaringan Komputer mengangkat tema Serial Experiments Lain, di mana arsitektur jaringan dibangun untuk merepresentasikan sistem The Wired. Praktikum ini berfokus pada konfigurasi dasar router, manajemen antarmuka jaringan (interface), pengaturan NAT/masquerade, firewall, DHCP/DNS resolver, analisis trafik menggunakan Wireshark, konfigurasi FTP server, layanan Telnet, pemindaian port (port scanning) dengan Netcat, pengamanan akses jarak jauh menggunakan SSH Key-based Authentication, serta analisis berbagai file packet capture (pcap) untuk investigasi keamanan siber (analisis brute-force, USB HID keystroke, FTP theft, C2 traffic, SMB transfer, SMTP threat, dan dekripsi TLS).

PEMBAHASAN & LANGKAH PERCOBAAN
Soal 1 & 2: Konfigurasi Topologi & Alamat IP (GNS3)
Keterangan Topologi: Node lain bertindak sebagai Router utama yang menghubungkan internet publik via NAT/DHCP pada eth0, serta membagi jaringan ke 3 Switch (Switch 1 untuk Alice & Mika, Switch 2 untuk Chisa, dan Switch 3 untuk Knights & Eiri).

Konfigurasi Interface (/etc/network/interfaces):
```text
# Node: lain
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

# Node: alice
auto eth0
iface eth0 inet static
	address 10.190.1.2
	netmask 255.255.255.0
	gateway 10.190.1.1

# Node: mika
auto eth0
iface eth0 inet static
	address 10.190.1.3
	netmask 255.255.255.0
	gateway 10.190.1.1

# Node: chisa
auto eth0
iface eth0 inet static
	address 10.190.2.2
	netmask 255.255.255.0
	gateway 10.190.2.1

# Node: knights
auto eth0
iface eth0 inet static
	address 10.190.3.2
	netmask 255.255.255.0
	gateway 10.190.3.1

# Node: eiri
auto eth0
iface eth0 inet static
	address 10.190.3.3
	netmask 255.255.255.0
	gateway 10.190.3.1
```
Soal 3 & 4: Konektivitas Antar Entitas & Akses Internet (NAT Masquerade & DNS)
Perintah Konfigurasi pada Node lain:
```BASH
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth3 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT
```
Konfigurasi DNS pada masing-masing Client:
```BASH
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```
Pengujian Koneksi:
```BASH
ping -c 2 8.8.8.8
ping -c 2 google.com
```
Soal 5: Automasi Konfigurasi & Script Verifikasi Status (cek_status.sh)
Menambahkan Persistent Rule di /etc/network/interfaces (Node lain):
```BASH
up sysctl -w net.ipv4.ip_forward=1
up iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
Script Verifikasi /root/cek_status.sh:
```BASH
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
Eksekusi Script:
```BASH
chmod +x /root/cek_status.sh
/root/cek_status.sh
```
Soal 6: Generator Traffic & Packet Sniffing pada Node Mika
Script Traffic (/root/traffic_protocol7.sh):
```BASH
#!/bin/sh

while true; do
    ping -c 1 10.190.1.1 > /dev/null 2>&1
    nslookup google.com 8.8.8.8 > /dev/null 2>&1
    nc -z -w 1 10.190.1.1 80 > /dev/null 2>&1
    sleep 2
done
```
Menjalankan Script di Background:
```BASH
chmod +x /root/traffic_protocol7.sh
/root/traffic_protocol7.sh &
```
Wireshark Filter: dns || icmp
Soal 7: Konfigurasi FTP Server (chisa) & Manajemen Akses Pengguna
Instalasi & Konfigurasi di chisa (10.190.2.2):
```BASH
apt install -y --allow-unauthenticated vsftpd

mkdir -p /var/wired/data
chmod 777 /var/wired/data
chown -R ftp:nogroup /var/wired/data

useradd -m -s /bin/bash alice 2>/dev/null
useradd -m -s /bin/bash mika 2>/dev/null
useradd -m -s /bin/bash eiri 2>/dev/null

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
```
Pengujian Client (alice, mika, eiri): Berhasil membuktikan hak akses Read & Write untuk Alice, Read-Only untuk Mika (error 550 saat put), dan Blacklisted untuk Eiri (error 530 langsung pada saat login).
Soal 8: Transfer Dokumen Intelijen dari Knights ke FTP Chisa
Perintah Pengujian & Sniffing (tshark):
```BASH
apt update --fix-missing 
apt install -y --allow-unauthenticated ftp wget tshark 
cd /root 
wget --no-check-certificate 'https://docs.google.com/uc?export=download&id=1lFepK4wFmx55PnRki3NsHW-ivudSR0vg' -O knights_report.txt

tshark -i eth0 -f "host 10.190.2.2" -w /tmp/soal8.pcap >/dev/null 2>&1 &

ftp 10.190.2.2
# User: alice, Password: password
passive
put knights_report.txt
quit 
pkill tshark
```
Analisis & Jawaban:
Perintah FTP untuk Upload: STOR knights_report.txt
Kode Status Sukses Server: 226 Transfer complete
Port Data TCP yang Dinegosiasikan (PASV/EPSV): Port 59535 (didapat dari respons 229 Entering Extended Passive Mode (|||59535|)).

Soal 9: Pengunduhan Protokol Tujuh & Pembuktian Read-Only Mika
Perintah di Node Mika:
```BASH
cd /var/wired/data
wget --no-check-certificate 'https://docs.google.com/uc?export=download&id=1tKZu0rcti4t-fXX4jtXDSKDBWzsawfoN' -O protocol7_manifesto.txt
chmod 644 /var/wired/data/protocol7_manifesto.txt

cd /root
touch test_upload_mika.txt
ftp 10.190.2.2
# Login user: mika / password: password
get protocol7_manifesto.txt
put test_upload_mika.txt
quit
```
Hasil: Mika berhasil mengunduh file, tetapi gagal melakukan upload dengan pesan error 550 Permission denied.

Soal 10: Uji Ketahanan Latensi & ICMP Ping ke Server Chisa
Perintah Pengujian:
```BASH
ping -c 77 -s 128 -i 0.3 10.190.2.2
```
Analisis Wireshark & Statistik:
Echo Request (Knights $\rightarrow$ Chisa): Type 8, Code 0
Echo Reply (Chisa $\rightarrow$ Knights): Type 0, Code 0
Packet Loss: 0% (77 transmitted, 77 received)
Round Trip Time (RTT): Min: 0.254 ms, Avg: 0.586 ms, Max: 1.216 ms

Soal 11: Analisis Kelemahan Telnet (Plaintext & Character-at-a-Time)
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
