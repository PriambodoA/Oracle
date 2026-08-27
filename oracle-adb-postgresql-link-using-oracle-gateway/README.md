# Dokumen Implementasi: Database Link dari Oracle Autonomous Database (ExaCC) ke PostgreSQL

Dokumen ini menjelaskan langkah-langkah teknis untuk mengimplementasikan *Database Link* dari Oracle Autonomous Database (ADB) pada platform Exadata Cloud@Customer (ExaCC) on-premise ke PostgreSQL menggunakan Oracle Database Gateway.

## 1. Arsitektur Solusi

Karena implementasi ini sepenuhnya berada di lingkungan *on-premise* (ExaCC, Database Gateway, dan PostgreSQL), alur koneksinya adalah sebagai berikut:

**Oracle Autonomous Database (ExaCC)** < **Oracle Database Gateway (ODBC)** < **PostgreSQL**

- **Oracle Autonomous Database (ExaCC)**: Berperan sebagai sumber (*source*), tempat di mana *database link* dibuat dan *query* dieksekusi.
- **Oracle Database Gateway**: Berperan sebagai perantara (*middleware*) yang menerjemahkan *query* Oracle (PL/SQL) menjadi *query* yang dapat dipahami oleh PostgreSQL. Koneksi ini menggunakan mekanisme *Heterogeneous Services* (HS).
- **PostgreSQL**: Berperan sebagai target *database*.

---

## 2. Konfigurasi Jaringan (Network Whitelisting)

Untuk memastikan komunikasi berjalan lancar antar instans, konfigurasi *firewall* atau *Security Group* harus disesuaikan. Berikut adalah tabel *whitelist* jaringan yang diperlukan:

| Source (Asal) | Target (Tujuan) | Port Default | Protokol | Keterangan |
| :--- | :--- | :--- | :--- | :--- |
| Oracle ADB (ExaCC) Nodes | Oracle Database Gateway | `1521` (TCP) | Oracle Net | Koneksi dari ExaCC ke Listener Database Gateway. (Port bisa disesuaikan jika menggunakan port kustom). |
| Oracle Database Gateway | PostgreSQL Server | `25432` (TCP) | PostgreSQL | Koneksi ODBC dari server Gateway ke PostgreSQL. |

---

## 3. Konfigurasi PostgreSQL

Pada sisi PostgreSQL, kita perlu membuat *user* khusus untuk *database link* dan mengizinkan koneksi dari IP Oracle Database Gateway.

### 3.1 Membuat User
Login ke PostgreSQL dan jalankan perintah berikut:
```sql
CREATE ROLE oracle_dblink_user WITH LOGIN PASSWORD 'StrongPassword123';
GRANT CONNECT ON DATABASE target_db TO oracle_dblink_user;
GRANT USAGE ON SCHEMA public TO oracle_dblink_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO oracle_dblink_user;
```

---

## 4. Konfigurasi Oracle Database Gateway

Langkah ini mencakup instalasi *driver* ODBC dan konfigurasi *Heterogeneous Services* pada server Oracle Database Gateway.

### 4.1 Instalasi Driver ODBC PostgreSQL
Pada server Gateway (diasumsikan menggunakan Oracle Linux/RHEL):
```bash
sudo yum install unixODBC postgresql-odbc
```

### 4.2 Konfigurasi ODBC (`/etc/odbc.ini`)
Tambahkan *Data Source Name* (DSN) untuk PostgreSQL:
```ini
[PG_DSN]
Description = PostgreSQL via ODBC
Driver      = /usr/lib64/psqlodbc.so
Database    = target_db
Servername  = <IP PostgreSQL>
Port        = 25432
UserName    = oracle_dblink_user
Password    = StrongPassword123
```
> *Test koneksi odbc menggunakan perintah:* `isql -v PG_DSN`

### 4.3 Konfigurasi Gateway Initialization File
Masuk ke direktori Oracle Gateway (`$ORACLE_HOME/hs/admin`) dan buat file `initPGSQL.ora`:
```ini
# This is a sample agent init file that contains the HS parameters that are
# needed for an ODBC Agent.

# initDWH.ora - DG4ODBC initialization file
HS_FDS_CONNECT_INFO = PG_DSN
HS_FDS_TRACE_LEVEL = 0
HS_FDS_SHAREABLE_NAME = /usr/lib64/psqlodbc.so
HS_LANGUAGE = AMERICAN_AMERICA.WE8ISO8859P9
# Environment variables
set ODBCINI=/etc/odbc.ini
```
### 4.4 Konfigurasi `tnsnames.ora`
Meskipun database link utama dibuat di Autonomous Database, terkadang entri tnsnames.ora pada server Gateway diperlukan untuk keperluan pengujian lokal (seperti perintah tnsping atau koneksi uji menggunakan SQL*Plus dari server Gateway).
Edit file $ORACLE_HOME/network/admin/tnsnames.ora di server Database Gateway dan tambahkan konfigurasi berikut:
```text
PGSQL_GW =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = <IP Database Gateway>)(PORT = 1521))
    (CONNECT_DATA =
      (SID = PGSQL)
    )
    (HS = OK)
  )
```
> **Pengujian Lokal dari Server Gateway:**
> Anda dapat menguji apakah *gateway* merespons dengan benar dari server Gateway itu sendiri menggunakan perintah:
> ```bash
> tnsping PGSQL_GW
> 
> ```
> 
> 

---

### 4.5 Konfigurasi `listener.ora`
Edit file `$ORACLE_HOME/network/admin/listener.ora` di server Gateway untuk mendengarkan permintaan *Heterogeneous*:
```text
SID_LIST_LISTENER_GW =
  (SID_LIST =
    (SID_DESC =
      (SID_NAME = PGSQL)
      (ORACLE_HOME = /path/to/gateway/oracle_home)
      (PROGRAM = dg4odbc)
    )
  )

LISTENER_GW =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = <IP Database Gateway>)(PORT = 1521))
    )
  )
```
Mulai ulang *listener*:
```bash
lsnrctl stop LISTENER_GW
lsnrctl start LISTENER_GW
```

---

## 5. Konfigurasi Oracle Autonomous Database (ExaCC)

Karena berada di infrastruktur ExaCC, Anda memiliki akses penuh ke fitur *database link*. 

### 5.1 Membuat Database Link
Login ke Oracle Autonomous Database (ExaCC) menggunakan SQL*Plus, SQL Developer, atau Database Actions sebagai *user* dengan *privilege* `CREATE DATABASE LINK`.

Jalankan perintah berikut:

```sql
CREATE DATABASE LINK pg_link 
CONNECT TO "oracle_dblink_user" IDENTIFIED BY "StrongPassword123" 
USING '(DESCRIPTION=
        (ADDRESS=(PROTOCOL=TCP)(HOST=<IP Database Gateway>)(PORT=1521))
        (CONNECT_DATA=(SID=PGSQL))
        (HS=OK)
      )';
```
*Catatan: Parameter `(HS=OK)` sangat penting karena memberi tahu Oracle bahwa ini adalah koneksi Heterogeneous Services.*

### 5.2 Pengujian Database Link
Lakukan *query* sederhana untuk memverifikasi koneksi:
```sql
-- Cek tanggal dari PostgreSQL
SELECT * FROM "some_table"@pg_link;
```
> **Penting**: Nama objek pada PostgreSQL bersifat *case-sensitive*. Pastikan Anda mengapit nama tabel atau kolom menggunakan tanda kutip ganda (`" "`) jika di PostgreSQL dibuat dengan huruf kecil/besar spesifik.
