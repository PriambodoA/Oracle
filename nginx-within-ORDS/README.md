# Dokumentasi Implementasi NGINX dan ORDS pada Oracle Autonomous Database

## 1. Latar Belakang Pendekatan
Pendekatan menggunakan NGINX sebagai *reverse proxy* di depan Oracle REST Data Services (ORDS) dilakukan untuk mengatasi berbagai keterbatasan bawaan pada ORDS, antara lain:
*   **Ketiadaan Fitur SSL Termination:** ORDS *standalone* kurang optimal untuk menangani *SSL termination* skala produksi. NGINX mengambil alih peran enkripsi dan dekripsi SSL/TLS secara terpusat, sehingga mengurangi beban komputasi di sisi *backend* ORDS maupun Database.
*   **Reverse Proxy & Load Balancing:** NGINX bertindak sebagai *reverse proxy* cerdas yang dapat mendistribusikan *traffic* masuk ke beberapa *node* ORDS sekaligus (*High Availability*).
*   **Proteksi Keamanan Tambahan:** NGINX memfasilitasi penerapan *Security Headers* standar industri, restriksi ukuran unggahan (upload), dan pemblokiran akses ke *path* administratif ORDS yang sensitif dari internet/jaringan publik.

## 2. Kebutuhan Konfigurasi Network (Whitelisting & Firewall)
Untuk memastikan komunikasi berjalan lancar dan aman antara *client*, *instance* NGINX, dan *backend* ORDS (Oracle Autonomous/ExaCC), berikut adalah konfigurasi *Network Security Group* (NSG) atau *Firewall* yang perlu di-*whitelist*:

| Source (Asal) | Destination (Tujuan) | Port | Protokol | Deskripsi |
| :--- | :--- | :--- | :--- | :--- |
| End-User / Client | NGINX Instance (`10.119.36.50`) | `80`, `443` | TCP | Akses HTTP (untuk di-*redirect*) dan HTTPS dari klien ke server portal NGINX. |
| NGINX Instance (`10.119.36.50`) | ORDS Node 1 (`10.119.32.246`) | `443` | TCP | Akses *proxy* dari NGINX ke *backend* ORDS ExaCC Node 1. |
| NGINX Instance (`10.119.36.50`) | ORDS Node 2 (`10.119.32.247`) | `443` | TCP | Akses *proxy* dari NGINX ke *backend* ORDS ExaCC Node 2. |
| NGINX Instance (`10.119.36.50`) | ORDS Node 3 (`10.119.32.248`) | `443` | TCP | Akses *proxy* dari NGINX ke *backend* ORDS ExaCC Node 3. |

*Catatan untuk Database Link:* Apabila di dalam arsitektur Autonomous Database ini mengimplementasikan komunikasi *Database Link* ke sistem database lain, pastikan port listener standar Oracle (biasanya TCP `1521` atau `1522` / TCPS `2484`) diizinkan (*whitelisted*) antar-VCN (Virtual Cloud Network) atau IP *node* Database yang bersangkutan.

## 3. Penjelasan Konfigurasi NGINX (`proxy-prod.conf`)
File `proxy-prod.conf` berfungsi sebagai pengatur rute dan penjaga keamanan jaringan dari sisi luar menuju aplikasi APEX/ORDS. Berikut penjelasan detail dari tiap blok konfigurasinya:

### A. Load Balancing (Upstream Backend)
*   Menggunakan *block* `upstream ords_backend` untuk mendefinisikan tiga alamat IP *backend* ExaCC (`10.119.32.246`, `10.119.32.247`, `10.119.32.248`) pada port `443` [cite: 1].
*   Menggunakan metode algoritma `ip_hash` agar *session* dari IP *client* yang sama akan selalu diarahkan ke *backend* ORDS yang sama (mencegah *session loss*) [cite: 1].
*   Dilengkapi dengan parameter *health* yaitu `max_fails=3` dan `fail_timeout=10s` untuk *failover* otomatis apabila ada *node* ORDS yang *down* [cite: 1].
*   Mengaktifkan `keepalive 32` guna memelihara koneksi persisten (*connection pooling*) sehingga meminimalkan latensi pembuatan koneksi baru [cite: 1].

### B. Redirect HTTP ke HTTPS
*   Mendengarkan *port* `80` (HTTP) dan secara otomatis memaksa *redirect* (kode HTTP `301`) seluruh koneksi ke port HTTPS (443) menggunakan `$host$request_uri` [cite: 1]. Ini memastikan tidak ada trafik *clear-text* yang masuk [cite: 1].

### C. Konfigurasi Server Utama (HTTPS) & Limitasi
*   Server mendengarkan *port* `443` dengan protokol `ssl` dan `http2` untuk performa modern [cite: 1].
*   `server_tokens off;` diaktifkan untuk menghilangkan informasi versi NGINX pada respon HTTP (guna menghindari celah sekuriti) [cite: 1].
*   Menetapkan `client_max_body_size 50m;` untuk membatasi ukuran unggahan file maksimal menjadi 50MB dari *client* [cite: 1].

### D. Keamanan SSL dan Security Headers
*   Memasukkan lokasi sertifikat SSL menggunakan *fullchain* (`/etc/nginx/ssl/fullchain.pem`) dan *private key* (`/etc/nginx/ssl/privkey.pem`) [cite: 1].
*   Membatasi versi SSL hanya pada `TLSv1.2` dan `TLSv1.3`, beserta pemilihan spesifikasi `ciphers` yang disarankan (`HIGH:!aNULL:!MD5`) [cite: 1].
*   Menyisipkan *Security Headers* yang sangat krusial [cite: 1]:
    *   `X-Frame-Options "SAMEORIGIN"`: Menghalau serangan *Clickjacking* [cite: 1].
    *   `X-Content-Type-Options "nosniff"`: Menghalau upaya *MIME sniffing* dari *browser* [cite: 1].
    *   `X-XSS-Protection "1; mode=block"`: Proteksi bawaan dari *Cross-Site Scripting* [cite: 1].
    *   `Strict-Transport-Security`: Memaksa komunikasi di kemudian hari harus tetap melalui HTTPS (*HSTS*) [cite: 1].

### E. Proxy SSL Backend & Penanganan Header
*   NGINX diinstruksikan untuk tidak memvalidasi ulang sertifikat milik ExaCC *backend* internal (`proxy_ssl_verify off;`) namun tetap memakai protokol enkripsi yang kuat (TLSv1.2 & v1.3) [cite: 1].
*   Saat meneruskan koneksi (*proxy_pass*), NGINX membawa *Standard Proxy Headers* (`Host`, `X-Real-IP`, `X-Forwarded-For`, dll.) agar *backend* ORDS mengenali IP dan skema protokol awal (HTTPS) *user*, bukan IP milik NGINX [cite: 1].

### F. Routing Berdasarkan Path (Lokasi)
1.  **Proteksi Path Administratif:** NGINX diatur untuk menolak (`deny all; return 403;`) setiap permintaan menuju *path* sensitif ORDS seperti `/ords/sql-developer`, `/ords/apex_admin`, `/ords/metadata-catalog`, `/ords/internal`, dll [cite: 1]. Hal ini menutup akses *tools* internal dan *developer* dari jaringan luar [cite: 1].
2.  **Reverse Proxy ORDS Utama:** Me-*routing* `/ords/` ke `https://ords_backend/ords/` yang juga disokong dengan dukungan untuk *WebSocket* dan HTTP/1.1 (`Upgrade`, `Connection "upgrade"`) dan pembatasan waktu koneksi (*timeout*) [cite: 1].
3.  **Static Resources (`/i/`):** Semua aset statis (*image*, CSS, JS) di bawah folder `/i/` APEX diarahkan ke ORDS dengan penambahan instruksi `expires 7d;` dan `Cache-Control` [cite: 1]. Konfigurasi ini menyuruh *browser* menyimpan file *cache* selama seminggu guna meningkatkan kecepatan prapemuatan aplikasi [cite: 1].
4.  **Health Check Endpoint:** Rute `/healthz` yang merespon status HTTP `200 "ok"` disediakan khusus untuk alat pemantauan status layanan eksternal (contoh: *Load Balancer* Cloud), dengan `access_log off;` agar tidak membanjiri log file [cite: 1].
5.  **Auto-Redirect Root (*Landing Page*):** Jika *user* sekadar mengakses `https://10.119.36.50/`, NGINX akan otomatis me-lempar (`return 301`) pengguna ke URL spesifik aplikasi APEX *Bank Jakarta Portal* di alamat `/ords/apexdbuat/r/bjktapp/bank-jakarta-portal` [cite: 1].
