# WA Worker - Rimender Presensi BPS

Aplikasi rimender presensi pulang otomatis/manual melalui WhatsApp Bot (aku lebih suka menyebutnya WAWorker).

---

## 🚀 Fitur Utama

1. **Arsitektur Microservices**:
   - **Python Flask Backend / Crawler** ([src/flask/app.py](file:///c:/laragon/www/waworker-rimender-presensi/src/flask/app.py)): Mengelola dashboard UI, scheduler otomatisasi, logika parsing data presensi, dan pengelolaan data lokal.
   - **Node.js WhatsApp Bot** ([src/node/index.js](file:///c:/laragon/www/waworker-rimender-presensi/src/node/index.js)): Bertanggung jawab atas konektivitas WhatsApp, sinkronisasi kontak kustom, caching Linked Identifier (LID), dan pengiriman pesan.
2. **Sistem Login SSO & Caching Sesi BPS**:
   - Mendukung login SSO Keycloak BPS dengan PKCE flow.
   - Bypass proteksi WAF F5 BIG-IP dengan penanganan cookie khusus.
   - Caching token dan cookie di `data/session.json` untuk mencegah blokir akibat deteksi *multiple login* (re-login berkali-kali). Otomatis melakukan relogin/refresh jika sesi kedaluwarsa.
3. **Parser Protobuf Tanpa gRPC Frame (Raw Protobuf)**:
   - Komunikasi data presensi BPS memanfaatkan Protocol Buffers.
   - Mengambil data presensi dengan menyusun payload biner mentah secara dinamis dan mengekstrak *timestamp* secara presisi langsung dari byte tag respons.
4. **Scheduler Pengingat Presensi Otomatis**:
   - Berjalan sebagai thread latar belakang untuk memonitor waktu pengiriman reminder.
   - **Aturan Hari Jumat (`j` Suffix)**: Jam pengiriman yang memiliki suffix `j` (misalnya `16:05j` atau `20:00j`) secara otomatis akan **ditambahkan 30 menit** khusus pada hari Jumat (menjadi `16:35` dan `20:30`) untuk menyesuaikan dengan jam kepulangan hari Jumat. Offset ini dicatat di log terminal, dan tampilan dashboard normal akan menyembunyikan akhiran `j` serta menampilkan waktu hasil kalkulasi.
   - **Placeholder Jam Dinamis (`{jam}`)**: Isi template pesan dapat disisipkan tag `{jam}` (atau `{JAM}`) yang secara otomatis digantikan dengan waktu pengiriman riil (HH:MM).
   - **Jitter Delay**: Pengiriman pesan bulk menggunakan interval jeda acak (diatur di `config.json`) untuk mencegah deteksi spam WhatsApp.
5. **Sistem Pengecualian Harian (Daily Exclude)**:
   - Pegawai tertentu dapat dikecualikan dari daftar pengiriman reminder hari ini dengan menekan tombol silang (`✕`) di dashboard atau via CLI.
   - Daftar pengecualian akan di-reset secara otomatis pada pergantian hari (tengah malam).
6. **Log Real-time Console Page**:
   - Halaman khusus console log dengan visualisasi bergaya terminal macOS gelap yang dapat diakses di port Node.js (`http://localhost:3012`).
   - Pembaruan log secara real-time via Server-Sent Events (SSE).
7. **Integrasi System Tray (Windows)**:
   - Aplikasi dapat dijalankan di latar belakang Windows System Tray menggunakan menu tray modular.
   - Tombol close CMD dinonaktifkan untuk mencegah ketidaksengajaan penutupan aplikasi, dan console dapat dimunculkan/disembunyikan melalui menu System Tray.
8. **Command Line Interface (CLI)**:
   - Skrip interaktif & argumen [cli.py](file:///c:/laragon/www/waworker-rimender-presensi/cli.py) untuk mengelola data, memicu crawling, memicu reminder manual, hingga mengonfigurasi pengaturan langsung dari terminal.

---

## 📂 Struktur Direktori

Berikut adalah struktur folder dan file setelah proses refaktorisasi:

```text
waworker-rimender-presensi/
├── data/                         # Direktori penyimpanan data persistence & cache
│   ├── auth_info_baileys/        # Kredensial & sesi WhatsApp (Baileys)
│   ├── config.json               # Konfigurasi toggle crawl/reminder otomatis & waktu pengiriman
│   ├── contacts.json             # Caching data nama kontak kustom
│   ├── exclusions.json           # Caching pengecualian harian (jika di-persisted)
│   ├── inbox.json                # Riwayat pesan chat WhatsApp masuk
│   ├── lid_map.json              # Pemetaan Linked Identifier (LID) WhatsApp ke JID asli
│   ├── messages.json             # Template pesan pengingat terjadwal & pesan default
│   ├── pegawai.json              # Hasil konversi data pegawai aktif dari pegawai.csv
│   └── session.json              # Cache cookies WAF F5 & Token SSO Keycloak BPS
├── src/
│   ├── flask/                    # Sumber kode Backend Python (Flask)
│   │   ├── app.py                # Server Flask Utama, Dashboard Routing, & Scheduler
│   │   └── crawler.py            # Logika Login SSO BPS, Request Protobuf, & Crawl Presensi
│   └── node/                     # Sumber kode WhatsApp Bot (Node.js)
│       ├── chat.html             # UI Log Terminal Real-time
│       └── index.js              # Logika Utama Baileys Bot & API Gateway
├── static/                       # File statis untuk dashboard Flask
│   ├── script.js                 # Logika Frontend & Event Handler Dashboard
│   └── style.css                 # Gaya visual Modern Dark UI Dashboard
├── cli.py                        # Command Line Interface (CLI)
├── r.bat                         # Launcher utama aplikasi via System Tray (CMD Conhost)
├── tray_runner.js                # Pengelola aplikasi Windows System Tray (systray2)
├── package.json                  # Dependensi & script Node.js
├── requirements.txt              # Dependensi Python
├── .env                          # Konfigurasi Environment (Kredensial & Port)
└── pegawai.csv                   # Data awal daftar pegawai (NIP, Nama, WA, Prefix)
```

---

## ⚙️ Persyaratan & Instalasi

### Prerequisites
- Python 3.9 ke atas
- Node.js v16 ke atas
- OS Windows (untuk dukungan integrasi System Tray)

### Langkah Instalasi

1. **Clone repositori dan masuk ke direktori proyek**:
   ```bash
   cd waworker-rimender-presensi
   ```

2. **Instal dependensi Python**:
   ```bash
   pip install -r requirements.txt
   ```

3. **Instal dependensi Node.js**:
   ```bash
   npm install
   ```

4. **Siapkan Konfigurasi Environment (`.env`)**:
   Buat file `.env` di root direktori dengan contoh isi berikut:
   ```env
   CLIENT_ID=03340-bomonitor-28t
   CLIENT_SECRET=your_client_secret_here

   SSO_USERNAME=your_sso_username
   SSO_PASSWORD=your_sso_password

   FLASK_PORT=5012
   NODE_PORT=3012
   ```

5. **Siapkan Data Pegawai**:
   Letakkan file `pegawai.csv` di root direktori. Skrip akan otomatis mengonversinya menjadi format JSON di `data/pegawai.json` pada saat startup pertama kali. Format CSV:
   ```csv
   nip,nama,wa,prefix
   199xxxxxxxxx,Nama Pegawai,08xxxxxxxxxx,Pak/Bu
   ```

---

## 🚀 Menjalankan Aplikasi

Aplikasi dapat dijalankan dengan dua cara:

### A. Menggunakan Launcher Windows System Tray (Direkomendasikan)
Jalankan file batch [r.bat](file:///c:/laragon/www/waworker-rimender-presensi/r.bat):
- Klik ganda `r.bat` atau panggil lewat terminal:
  ```cmd
  r.bat
  ```
- Launcher akan otomatis mematikan proses lama (Node/Python) yang bertabrakan, membuka konsol Windows klasik (`conhost.exe`), dan meluncurkan server.
- Jendela konsol akan bersembunyi secara otomatis setelah 1 detik ke dalam System Tray Windows.
- Menu klik kanan System Tray menyediakan kontrol untuk menampilkan/menyembunyikan konsol terminal serta keluar dari seluruh proses aplikasi.
- Halaman dashboard (`http://localhost:5012`) akan terbuka secara otomatis di browser default Anda.

### B. Menjalankan Server secara Manual (Untuk Development / Debugging)

Jika ingin memantau log secara langsung di terminal terpisah:

1. **Jalankan WhatsApp Bot Helper (Node.js)**:
   ```bash
   npm start
   ```
   *Buka browser di `http://localhost:3012` untuk melihat halaman konsol log real-time.*

2. **Jalankan Flask Backend (Python)**:
   ```bash
   python src/flask/app.py
   ```
   *Buka browser di `http://localhost:5012` untuk memantau dashboard presensi.*

---

## 🛠️ Panduan Command Line Interface (CLI)

Anda bisa mengendalikan fungsionalitas server langsung dari terminal menggunakan [cli.py](file:///c:/laragon/www/waworker-rimender-presensi/cli.py).

### 1. Mode Interaktif
Jalankan tanpa parameter tambahan untuk masuk ke menu interaktif berbasis teks:
```bash
python cli.py
```

### 2. Mode Argumentasi (CLI Perintah Langsung)
Berikut adalah daftar perintah CLI yang tersedia:

- **Cek Status Konektivitas WhatsApp**:
  ```bash
  python cli.py status
  ```
- **Jalankan Crawl Presensi Manual Sekarang**:
  ```bash
  python cli.py crawl
  ```
- **Tampilkan Tabel Rekap Hasil Crawl Terakhir**:
  ```bash
  python cli.py results
  ```
- **Kirim Reminder Massal ke Seluruh Pegawai yang Belum Pulang**:
  ```bash
  python cli.py rimend-all
  ```
- **Kirim Reminder Individu menggunakan NIP**:
  ```bash
  python cli.py rimend-individual <NIP>
  ```
- **Kecualikan Pegawai dari Pengingat Hari Ini**:
  ```bash
  # Mengecualikan pegawai (tidak dikirimi pesan hari ini)
  python cli.py exclude <NIP> --status on

  # Memasukkan kembali pegawai ke daftar pengingat
  python cli.py exclude <NIP> --status off
  ```
- **Kirim Pesan Uji Coba Ke Nomor Tertentu**:
  ```bash
  python cli.py send-test <NOMOR_WA> --message "Pesan Uji Coba Kustom"
  ```
- **Upload / Ganti Data Pegawai via CSV**:
  ```bash
  python cli.py upload-csv <PATH_FILE_CSV>
  ```
- **Melihat & Mengubah Konfigurasi Otomatisasi**:
  ```bash
  # Melihat konfigurasi aktif
  python cli.py config

  # Mengaktifkan/menonaktifkan crawl & reminder otomatis
  python cli.py config --crawl on --rimend off
  ```

---

## 🧪 Detail Teknis & Aturan Logika

- **Batas Jam Pulang Presensi BPS**:
  - Hari Senin s.d. Kamis: Pegawai dihitung sudah melakukan presensi pulang dengan sah jika jam pulang tercatat `≥ 16:00`.
  - Hari Jumat: Pegawai dihitung sudah melakukan presensi pulang dengan sah jika jam pulang tercatat `≥ 16:30`.
  - Pegawai dengan status dinas luar atau cuti (seperti `CB1`, `DL`, `CT`) secara otomatis ditandai `Rimend: Tidak` dan dikecualikan.
- **Logika Suffix `j` Jumat**:
  - Untuk setiap entri waktu di config `rimender_times` (misal: `["16:05j", "18:00", "20:00j"]`), scheduler Python Flask akan memeriksa hari aktif saat ini.
  - Pada hari Jumat, jam dengan akhiran `j` akan ditambah 30 menit. Jam `16:05j` dipicu pukul `16:35` dan `20:00j` dipicu pukul `20:30`.
  - Scheduler mencatat aktivitas log ini ke konsol terminal: `[Friday Rule] Menambahkan offset 30 menit untuk jam {jam_asal} -> {jam_baru}`.
- **Penanganan `{jam}` di Template Pesan**:
  - Sebelum dikirimkan lewat API, baik Python (`app.py`) maupun Node.js (`index.js`) akan mengganti tag `{jam}` atau `{JAM}` di isi pesan dengan jam riil pengiriman pesan saat itu (misal: "16:35").
