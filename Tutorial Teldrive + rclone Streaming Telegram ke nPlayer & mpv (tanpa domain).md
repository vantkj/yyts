# Tutorial Teldrive + rclone: Streaming Telegram ke nPlayer & mpv (tanpa domain)

Oct 30, 2026 · @sultangg

## Gambaran dan kebutuhan

File disimpan di channel privat Telegram milikmu, Teldrive di VPS yang mengaturnya, dan nPlayer atau mpv memutarnya lewat WebDAV memakai IP VPS, tanpa domain.

Alurnya:

1. Kamu upload video lewat web Teldrive atau rclone, lalu Teldrive memecahnya jadi chunk dan menyimpannya di channel Telegram.
2. rclone di VPS membaca isi Teldrive dan menyajikannya sebagai server WebDAV di port 8081.
3. nPlayer (iPhone) atau mpv (PC/Android) membuka `http://IP_VPS:8081` dan memutar langsung, termasuk seek.

Yang kamu butuhkan:

- VPS Ubuntu 22.04 atau 24.04 dengan IP publik, minimal 1 vCPU dan 2 GB RAM, lokasi dekat kamu (misalnya Singapura atau Jepang).
- Bandwidth VPS yang besar, karena setiap kali menonton, videonya lewat VPS.
- Akun Telegram dan satu channel privat kosong khusus penyimpanan. Buat dulu di aplikasi Telegram.
- Pemutar: nPlayer di iPhone atau mpv di PC/Android.
- Waktu sekitar 45 menit.

Sebelum mulai:

- File lama di Telegram tidak ikut terbaca. Teldrive menyimpan file sebagai chunk dengan databasenya sendiri, jadi file yang sudah ada harus diupload ulang lewat Teldrive.
- Dokumen ini disusun dari dokumentasi resmi Teldrive dan rclone, dan belum diuji langsung. Tiap langkah punya cara cek, jadi berhenti dan perbaiki kalau hasilnya tidak sesuai.
- Ganti semua tulisan huruf besar seperti `IP_VPS`, `USERMU`, dan `PASSKUAT` dengan nilai milikmu.

## Langkah 1: Siapkan VPS

Login ke VPS lewat SSH dari PC (atau aplikasi SSH di HP), lalu perbarui sistem dan sinkronkan jam. Login Teldrive gagal kalau jam server meleset, jadi langkah jam ini wajib.

```bash
ssh root@IP_VPS
apt update && apt upgrade -y
apt install -y ufw curl openssl nano
timedatectl set-ntp true
timedatectl status
```

Cek: baris `System clock synchronized: yes` muncul di hasil `timedatectl status`.

Aktifkan firewall dasar. Buka port SSH dulu sebelum mengaktifkan, kalau tidak kamu bisa terkunci dari VPS sendiri:

```bash
ufw allow 22/tcp
ufw --force enable
ufw status
```

Port lain (8081 untuk WebDAV) dibuka nanti di Langkah 7.

## Langkah 2: Install Docker

```bash
curl https://get.docker.com | sh
docker --version
docker compose version
```

Cek: kedua perintah versi menampilkan nomor versi, bukan error `command not found`.

## Langkah 3: Pasang PostgreSQL

Teldrive menyimpan indeks file (nama, folder, letak chunk di Telegram) di PostgreSQL. Image `pgroonga` dipakai karena dokumentasi Teldrive mensyaratkan ekstensi itu.

Buat folder kerja dan password database. Password dibuat dari hex supaya tidak ada karakter khusus yang merusak alamat koneksi nanti:

```bash
mkdir -p ~/teldrive/db && cd ~/teldrive/db
openssl rand -hex 16
```

Catat hasilnya sebagai `PASSWORD_DB`. Lalu buat file compose:

```bash
nano docker-compose.yml
```

Isi dengan ini, ganti `PASSWORD_DB`:

```yaml
services:
  postgres:
    image: groonga/pgroonga:latest-alpine-17
    container_name: postgres_db
    restart: always
    networks:
      - postgres
    environment:
      - POSTGRES_USER=teldrive
      - POSTGRES_PASSWORD=PASSWORD_DB
      - POSTGRES_DB=postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data

networks:
  postgres:
    external: true

volumes:
  postgres_data:
    external: true
```

Simpan di nano dengan Ctrl+O, Enter, lalu keluar dengan Ctrl+X. Jalankan:

```bash
docker network create postgres
docker volume create postgres_data
docker compose up -d
docker ps
docker logs postgres_db 2>&1 | tail -n 5
```

Cek: `postgres_db` berstatus `Up`, dan log menampilkan `database system is ready to accept connections`. Port database sengaja tidak dibuka ke luar, jadi hanya container di jaringan `postgres` yang bisa mengaksesnya.

## Langkah 4: Konfigurasi dan jalankan Teldrive

Buat dua nilai acak, satu untuk `JWT_SECRET` dan satu untuk `ENCRYPTION_KEY`. Jalankan perintah ini dua kali dan catat keduanya:

```bash
cd ~/teldrive
openssl rand -hex 32
openssl rand -hex 32
```

Buat file konfigurasi:

```bash
nano config.toml
```

Isi, dengan `PASSWORD_DB` dari Langkah 3 dan username Telegram-mu tanpa tanda `@`:

```toml
[db]
data-source = "postgres://teldrive:PASSWORD_DB@postgres/postgres"

[jwt]
allowed-users = ["username_telegram_kamu"]
secret = "JWT_SECRET"

[tg]
ntp = true

[tg.uploads]
encryption-key = "ENCRYPTION_KEY"
```

`allowed-users` membatasi siapa yang boleh login ke Teldrive-mu, `ntp = true` menyinkronkan waktu otomatis, dan kunci enkripsi bersifat opsional tapi disarankan oleh dokumentasi.

Buat file compose untuk Teldrive:

```bash
nano docker-compose.yml
```

```yaml
services:
  teldrive:
    image: ghcr.io/tgdrive/teldrive
    container_name: teldrive
    restart: always
    volumes:
      - ./config.toml:/config.toml
      - ./storage.db:/storage.db
    ports:
      - "127.0.0.1:8080:8080"
    networks:
      - postgres

networks:
  postgres:
    external: true
```

Dokumentasi menyebut image `ghcr.io/tgdrive/teldrive`, dua file yang di-mount, dan port 8080. Penulis menambahkan `127.0.0.1:` pada port supaya halaman Teldrive tidak terbuka ke internet.

Jalankan:

```bash
touch storage.db
docker compose up -d
docker logs --tail 30 teldrive
curl -I http://127.0.0.1:8080
```

Cek: container `teldrive` berstatus `Up`, log tanpa pesan error, dan `curl` mengembalikan respons HTTP.

Penting: salin `config.toml` (berisi secret dan kunci enkripsi) ke tempat aman di luar VPS sekarang juga. Dokumentasi memperingatkan untuk mencadangkan kredensial database, signing key, dan kunci enkripsi. Kalau kunci enkripsi hilang, file terenkripsi tidak bisa dibuka lagi.

## Langkah 5: Login pertama dan upload

Halaman Teldrive hanya terbuka di dalam VPS (`127.0.0.1`), jadi aksesnya lewat SSH tunnel dari PC-mu. Jalankan di PC (bukan di VPS) dan biarkan jendelanya terbuka:

```bash
ssh -L 8080:127.0.0.1:8080 root@IP_VPS
```

Lalu buka `http://localhost:8080` di browser PC dan lakukan ini berurutan:

1. Login dengan akun Telegram-mu (nomor HP, kode login, dan password 2FA kalau aktif).
2. Buka **Settings** di dalam Teldrive dan sinkronkan channel.
3. Masih di Settings, pilih channel privat yang kamu buat sebagai **default channel**.
4. Buat folder (misalnya `Anime`, `Film`) lalu upload satu video kecil untuk tes, lewat tombol upload atau drag and drop.

Cek: video tampil di daftar file Teldrive dan bisa diputar dari sana. Kalau tampilan login tidak bergerak, penyebab paling sering adalah jam server, ulangi `timedatectl status` dari Langkah 1.

Jangan hapus atau ubah isi channel itu langsung dari aplikasi Telegram. Isinya adalah potongan file yang diatur Teldrive, dan menghapusnya dari Telegram membuat file di Teldrive rusak.

Tunnel ini hanya perlu dibuka saat kamu mengatur atau upload lewat web. Streaming ke nPlayer dan mpv tidak membutuhkannya.

## Langkah 6: Pasang rclone dan hubungkan ke Teldrive

rclone biasa tidak mengenal Teldrive. Dokumentasi Teldrive menyediakan skrip pemasang rclone yang sudah mendukungnya. Skrip ini dari pihak ketiga, jadi kalau mau lebih hati-hati, buka dulu isinya di `https://instl.vercel.app/rclone` sebelum dijalankan. Jalankan di VPS:

```bash
curl -sSL instl.vercel.app/rclone | bash
rclone version
which rclone
rclone config file
```

Catat dua hal dari hasilnya: lokasi program rclone (`which rclone`, dipakai di Langkah 7) dan lokasi file konfigurasi (`rclone config file`).

**Ambil access\_token.** rclone login ke Teldrive memakai token dari cookie sesi web-mu. Di PC, dengan tunnel dan halaman Teldrive dari Langkah 5 masih terbuka:

1. Tekan F12 untuk membuka DevTools.
2. Buka tab **Application** (Chrome) atau **Storage** (Firefox), lalu **Cookies** dan pilih `http://localhost:8080`.
3. Salin nilai cookie bernama `access_token`.

Isi file konfigurasi rclone di VPS (`nano` pada path hasil `rclone config file`, biasanya `/root/.config/rclone/rclone.conf`):

```ini
[teldrive]
type = teldrive
api_host = http://localhost:8080
access_token = ISI_NILAI_COOKIE
chunk_size = 500M
upload_concurrency = 4
```

Nilai `chunk_size` dan `upload_concurrency` di atas adalah nilai bawaan, `chunk_size` maksimal 2 GB. Opsi `encrypt_files` bawaannya mati. Kalau mau file terenkripsi, tambahkan `encrypt_files = true` sebelum upload pertama lewat rclone.

Tes koneksi:

```bash
rclone lsd teldrive:
rclone ls teldrive:
```

Cek: folder dan video tes dari Langkah 5 muncul di daftar. Kalau muncul error 401 atau unauthorized, tokennya salah salin atau sudah kedaluwarsa, ulangi pengambilan cookie.

Upload banyak file sekarang bisa lewat rclone, dengan progres:

```bash
rclone copy /path/lokal/Anime teldrive:/Anime -P
```

## Langkah 7: Sajikan lewat WebDAV memakai IP

Buat username dan password untuk WebDAV. Pakai hex supaya aman dipakai di URL mpv:

```bash
openssl rand -hex 6
openssl rand -hex 12
```

Catat hasil pertama sebagai `USERMU` dan hasil kedua sebagai `PASSKUAT`. Tes dulu secara manual di VPS:

```bash
rclone serve webdav teldrive: --addr 0.0.0.0:8081 --user USERMU --pass PASSKUAT --read-only
```

Biarkan jalan, buka jendela SSH kedua ke VPS, dan jalankan:

```bash
curl -u USERMU:PASSKUAT http://127.0.0.1:8081/
```

Cek: keluar daftar folder-mu (XML atau HTML), bukan error 401. Setelah itu hentikan perintah di jendela pertama dengan Ctrl+C. Opsi `--read-only` membuat server ini hanya bisa dibaca, jadi pemutar tidak bisa menghapus atau menimpa file.

**Buka port di firewall.** Kalau IP internet rumah atau HP-mu relatif tetap, izinkan hanya IP itu:

```bash
ufw allow from IP_KAMU to any port 8081 proto tcp
```

Kalau IP-mu berubah-ubah, buka untuk semua dan andalkan password kuat:

```bash
ufw allow 8081/tcp
```

Kalau penyedia VPS-mu punya firewall sendiri di panelnya (security group), buka port 8081 di sana juga.

**Jalankan otomatis sebagai service** supaya tetap hidup setelah logout atau reboot. Ganti path rclone dengan hasil `which rclone` dari Langkah 6:

```bash
nano /etc/systemd/system/rclone-webdav.service
```

```ini
[Unit]
Description=rclone WebDAV untuk Teldrive
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/rclone serve webdav teldrive: --addr 0.0.0.0:8081 --user USERMU --pass PASSKUAT --read-only --config /root/.config/rclone/rclone.conf
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
chmod 600 /etc/systemd/system/rclone-webdav.service
systemctl daemon-reload
systemctl enable --now rclone-webdav
systemctl status rclone-webdav --no-pager
```

Cek: status `active (running)`. Log bisa dilihat dengan `journalctl -u rclone-webdav -f`.

**Risiko tanpa HTTPS.** Alamat `http://IP_VPS:8081` mengirim username dan password tanpa enkripsi, jadi bisa terbaca oleh jaringan yang dilewati (Wi-Fi umum, ISP). Pakai password panjang yang unik, batasi lewat IP kalau bisa, dan jangan bagikan alamatnya.

**Alternatif lebih aman tanpa domain: Tailscale.** Lalu lintas terenkripsi dan port 8081 tidak perlu dibuka ke internet. Pasang di VPS:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
tailscale ip -4
```

Pasang aplikasi Tailscale di HP atau PC dengan akun yang sama. Di service rclone, ganti `--addr 0.0.0.0:8081` dengan `--addr 100.x.x.x:8081` (IP Tailscale VPS hasil perintah terakhir), tambahkan `tailscaled.service` pada baris `After=`, lalu jalankan `systemctl daemon-reload && systemctl restart rclone-webdav`. Aturan firewall untuk port 8081 tidak diperlukan, dan di pemutar kamu memakai `http://100.x.x.x:8081`.

## Langkah 8: Putar di nPlayer dan mpv

**nPlayer (iPhone/iPad).** Nama menu bisa sedikit berbeda antar versi aplikasi.

1. Buka nPlayer, masuk ke bagian jaringan (Network), lalu tambah server baru (+).
2. Pilih jenis **WebDAV**.
3. Isi alamat server `IP_VPS`, port `8081`, protokol HTTP, serta username `USERMU` dan password `PASSKUAT`. Kalau memakai Tailscale, alamatnya IP Tailscale VPS.
4. Simpan, ketuk server itu, telusuri folder, dan ketuk video untuk memutar.

**mpv di PC.** Jalankan dari terminal. Spasi pada nama file ditulis `%20`:

```bash
mpv "http://USERMU:PASSKUAT@IP_VPS:8081/Anime/Judul/Episode%2001.mkv"
```

Untuk melihat path yang benar, buka `http://IP_VPS:8081` di browser dan login, lalu telusuri foldernya dan salin tautan filenya. Atau lihat dari VPS dengan `rclone lsf teldrive:/Anime`.

**mpv di Android.** Pakai fitur membuka alamat jaringan di aplikasi mpv-android, lalu tempel URL yang sama dengan format di atas.

**Tes akhir:**

1. Putar video tes dari Langkah 5, dan pastikan gambar dan suara jalan.
2. Loncat ke tengah video. Seek harus berfungsi, dan jeda beberapa detik saat lompat jauh masih wajar.
3. Putar dari jaringan seluler dan dari Wi-Fi untuk memastikan firewall dan alamatnya benar.

Pemutaran pertama bisa terasa agak lambat karena Teldrive mengambil chunk dari Telegram dulu. Kecepatannya bergantung pada bandwidth VPS dan pembatasan dari Telegram.

## Perawatan, keamanan, dan troubleshooting

**Cadangkan database tiap minggu.** Indeks file ada di database, jadi simpan cadangannya di luar VPS. Cadangan manual:

```bash
docker exec postgres_db pg_dump -U teldrive postgres > ~/teldrive-db-$(date +%F).sql
```

Otomatis tiap Minggu pukul 03.00 (jalankan `crontab -e`, tambahkan baris ini, lalu salin file `.sql` secara berkala ke PC dengan `scp`):

```bash
0 3 * * 0 docker exec postgres_db pg_dump -U teldrive postgres > /root/teldrive-db-$(date +\%F).sql
```

**Update Teldrive.** Cadangkan database dan `config.toml` dulu, lalu:

```bash
cd ~/teldrive
docker compose pull
docker compose up -d
```

**Keamanan.**

- Pakai password WebDAV yang panjang dan unik, dan jangan bagikan alamat serta kredensialnya.
- Biarkan port 8080 (Teldrive) dan port Postgres tertutup. Hanya 8081 (atau tidak ada, kalau memakai Tailscale) yang dibuka.
- Jalankan `apt update && apt upgrade -y` secara berkala.
- Simpan salinan terpisah untuk file yang penting. Dokumentasi Teldrive memperingatkan bahwa akun atau file di Telegram bisa saja tidak tersedia, dan pemakaian harus mengikuti ketentuan dan batas Telegram.

**Masalah yang sering muncul:**

| Gejala | Kemungkinan penyebab | Perbaikan |
| --- | --- | --- |
| Login Teldrive hanya berputar | Jam VPS tidak sinkron | `timedatectl set-ntp true`, lalu cek `timedatectl status` |
| `rclone lsd teldrive:` error 401 atau unauthorized | `access_token` salah salin atau kedaluwarsa | Ambil cookie baru, perbarui `rclone.conf`, lalu `systemctl restart rclone-webdav` |
| nPlayer atau mpv tidak bisa konek | Service mati, port tertutup, atau IP salah | `systemctl status rclone-webdav`, `ufw status`, cek firewall di panel VPS |
| Container `teldrive` restart terus | `config.toml` salah atau Postgres belum jalan | `docker logs teldrive` dan `docker ps` |
| File lama tidak muncul | Hanya file yang diupload lewat Teldrive yang terbaca | Upload ulang lewat Teldrive atau `rclone copy` |
| Video tersendat | Bandwidth VPS kecil, lokasi jauh, atau pembatasan Telegram | Pakai VPS yang lebih dekat dengan bandwidth lebih besar, dan bandingkan dengan file lain |

**Sumber (dokumentasi yang dibuka untuk menyusun panduan ini):**

- [Teldrive: Prerequisites](https://teldrive-docs.pages.dev/docs/getting-started/prerequisites)
- [Teldrive: Installation](https://teldrive-docs.pages.dev/docs/getting-started/installation)
- [Teldrive: Usage](https://teldrive-docs.pages.dev/docs/getting-started/usage)
- [Teldrive: rclone guide](https://teldrive-docs-2h3.pages.dev/docs/guides/rclone)
- [Repo Teldrive](https://github.com/tgdrive/teldrive)
