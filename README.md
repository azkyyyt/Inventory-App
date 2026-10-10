# DOKUMENTASI LENGKAP & PANDUAN PENGUMPULAN TUGAS
# MINI PROJECT 3 — INVENTORY MANAGEMENT SYSTEM (PHP NATIVE)
**Pertemuan 4: Relational Database + Advanced PHP**

---

## DAFTAR ISI
1. [Tentang Proyek](#1-tentang-proyek)
2. [Desain Database & Entity-Relationship Diagram (ERD)](#2-desain-database--entity-relationship-diagram-erd)
3. [Daftar Aturan Bisnis (Business Rules Wajib)](#3-daftar-aturan-bisnis-business-rules-wajib)
4. [Fitur Utama Sistem](#4-fitur-utama-sistem)
5. [Arsitektur Keamanan Aplikasi (Security Engineering)](#5-arsitektur-keamanan-aplikasi-security-engineering)
6. [Transaksi Stok & Row-Level Locking (ACID)](#6-transaksi-stok--row-level-locking-acid)
7. [Struktur Folder & Manifest Berkas](#7-struktur-folder--manifest-berkas)
8. [Panduan Langkah Instalasi & Menjalankan Proyek](#8-panduan-langkah-instalasi--menjalankan-proyek)
9. [Akun Pengujian (Demo Credentials)](#9-akun-pengujian-demo-credentials)
10. [Checklist Evaluasi Pengumpulan (Slide 36)](#10-checklist-evaluasi-pengumpulan-slide-36)
11. [Panduan Screenshot Pengujian & Demo Fitur (Slide 38)](#11-panduan-screenshot-pengujian--demo-fitur-slide-38)

---

## 1. TENTANG PROYEK
Aplikasi **Inventory Management System (Mini Project 3)** ini dibangun menggunakan **PHP Native Modern (Berorientasi Helper & PDO)** dan **Relational Database (MySQL / MariaDB)**.

Proyek ini dirancang untuk menjawab tantangan dunia nyata di mana aplikasi pencatatan inventaris gudang **tidak cukup hanya menggunakan operasi CRUD sederhana**. Sistem ini menerapkan:
* **Integritas Referensial Ketat**: Mengunci keterkaitan antar tabel menggunakan Foreign Key constraints (`RESTRICT` & `CASCADE`).
* **Konsistensi Saldo Stok**: Mencegah stok menjadi minus menggunakan transaksi database (`BEGIN TRANSACTION`, `COMMIT`, `ROLLBACK`) dan penguncian baris (*Pessimistic Locking* `FOR UPDATE`).
* **Keamanan Berlapis**: Proteksi dari serangan SQL Injection, Cross-Site Scripting (XSS), kebocoran akses otorisasi (*Broken Access Control*), dan kebocoran kata sandi.
* **Laporan Analitik**: Memanfaatkan *SQL Aggregate Functions* (`COUNT`, `SUM`, `CASE WHEN`, `GROUP BY`) untuk menyajikan metrik inventaris secara *real-time*.

---

## 2. DESAIN DATABASE & ENTITY-RELATIONSHIP DIAGRAM (ERD)

Struktur database dirancang secara normalisasi ke dalam 7 tabel relasional yang saling mengunci konsistensi data (Slide 5–11):

```text
       ┌──────────────┐ 1             1 ┌────────────────┐
       │    users     │─────────────────│ user_profiles  │ (One-to-One: user_id UNIQUE, CASCADE)
       └──────────────┘                 └────────────────┘
              │ 1
              │
              │ N
       ┌──────────────┐ N             1 ┌────────────────┐
       │stock_movements│─────────────────│    products    │ (One-to-Many: product_id, RESTRICT)
       └──────────────┘                 └────────────────┘
              │ N                               │ N
              │ (RESTRICT)                      │
              │                                 │ M
              ▼ 1                               ▼
       ┌──────────────┐                 ┌────────────────────┐
       │    users     │                 │ product_suppliers  │ (Junction M:N, Composite PK)
       └──────────────┘                 └────────────────────┘
                                                │ M
                                                │
                                                ▼ 1
                                        ┌────────────────┐
                                        │   suppliers    │
                                        └────────────────┘
                                                ▲
                                                │
       ┌──────────────┐ 1             N         │
       │  categories  │─────────────────────────┘
       └──────────────┘ (One-to-Many: category_id, RESTRICT)
```

### Rincian Tabel & Keterkaitan Foreign Key:
1. **`users`**: Menyimpan kredensial login (`username`, `password_hash`, `role`).
2. **`user_profiles` (One-to-One)**:
   * Terhubung ke `users.id` melalui foreign key `user_id` yang bersifat `UNIQUE`.
   * Aturan: `ON UPDATE CASCADE ON DELETE CASCADE`.
3. **`categories` (One-to-Many)**:
   * Mengelompokkan jenis produk.
   * Aturan: Menghapus kategori yang masih memiliki produk akan ditolak (`ON DELETE RESTRICT`).
4. **`products`**:
   * Master data barang inventaris (`sku` UNIQUE, `name`, `price`, `stock`, `minimum_stock`).
   * Foreign Key: `category_id` $\rightarrow$ `categories.id`.
5. **`suppliers`**:
   * Master vendor penyedia barang gudang.
6. **`product_suppliers` (Many-to-Many Junction Table)**:
   * Menghubungkan produk dengan banyak supplier dan mencatat atribut harga beli dari supplier (`supplier_price`).
   * Menggunakan *Composite Primary Key* (`product_id`, `supplier_id`) agar relasi tidak terduplikasi.
   * Aturan: `ON DELETE CASCADE`.
7. **`stock_movements` (Histori Mutasi Stok)**:
   * Mencatat mutasi keluar-masuk (`IN` / `OUT`), kuantitas (`quantity`), tanggal, dan petugas pencatat (`user_id`).
   * Foreign Key: `product_id` $\rightarrow$ `products.id` dan `user_id` $\rightarrow$ `users.id`.
   * Aturan: `ON DELETE RESTRICT` untuk menjamin audit trail histori tidak hilang.

---

## 3. DAFTAR ATURAN BISNIS (BUSINESS RULES WAJIB)

Sesuai materi perkuliahan **Slide 33**, aplikasi secara ketat menegakkan 8 aturan bisnis berikut:

| No | Aturan Bisnis | Implementasi Teknis & Validasi |
|:--:|---|---|
| **1** | **SKU Harus Unik** | Kolom `sku` diberi indeks `UNIQUE` pada MySQL, divalidasi ganda via query sebelum INSERT/UPDATE. |
| **2** | **Harga & Kuantitas Non-Negatif** | Harga jual (`price` $\ge 0$), stok (`stock` $\ge 0$), minimum stok ($\ge 0$), dan kuantitas mutasi (`quantity` $> 0$) divalidasi di server (`validation.php`). |
| **3** | **Barang Keluar $\le$ Stok Tersedia** | Jumlah barang keluar tidak boleh melebihi stok yang ada di gudang. Divalidasi saat transaksi. |
| **4** | **Stok Tidak Boleh Menjadi Negatif** | Formula: $\text{Stok Akhir} = \text{Stok Awal} - \text{Qty OUT}$. Jika $\text{Stok Akhir} < 0$, transaksi otomatis dibatalkan (*Rollback*). |
| **5** | **Produk Berhistori Dilarang Dihapus** | Produk yang memiliki catatan transaksi di `stock_movements` dilindungi oleh `ON DELETE RESTRICT`. Tombol hapus akan memicu pesan peringatan integritas referensial. |
| **6** | **Halaman Internal Membutuhkan Login** | Seluruh file di folder `public/` (kecuali `login.php`) memanggil `requireAuth()` yang mengecek `$_SESSION['user_id']`. |
| **7** | **Staff Dilarang Mengakses Kelola User** | Modul `public/users/` memanggil `requireRole('admin')`. Jika role `staff` memaksa masuk via URL, sistem langsung merespons dengan **HTTP 403 Forbidden**. |
| **8** | **Sorting Hanya Menggunakan Kolom Terdaftar** | Parameter `$_GET['sort']` diperiksa terhadap array *Allowlist* (`name`, `price`, `stock`, `sku`, `id`). Input berbahaya langsung dialihkan ke default aman. |

---

## 4. FITUR UTAMA SISTEM

Sesuai spesifikasi **Slide 32**, sistem mencakup fitur:
1. **Autentikasi & Sesi**:
   * Login, logout, proteksi session hijacking, dan pesan flash notification.
2. **Otorisasi (Role-Based Access Control / RBAC)**:
   * **Admin**: Akses penuh ke seluruh menu (Dashboard, Produk, Kategori, Supplier, Transaksi Stok, dan Kelola Pengguna).
   * **Staff**: Akses operasional harian (Dashboard, Lihat & Tambah Produk, Kategori, Supplier, Transaksi Masuk/Keluar, Histori Transaksi). Ditolak dari menu Pengguna.
3. **Master Data CRUD**:
   * Kelola Kategori, Produk, Supplier, dan Pengguna secara terstruktur.
4. **Relasi Many-to-Many Produk-Supplier**:
   * Halaman khusus untuk menghubungkan dan memutuskan supplier pada produk beserta penetapan harga beli khusus.
5. **Transaksi Barang Masuk (IN) & Barang Keluar (OUT)**:
   * Formulir pencatatan barang masuk yang menambah stok secara atomik.
   * Formulir barang keluar dengan pengecekan ketersediaan stok fisik.
6. **Histori Mutasi Stok**:
   * Tabel audit log kronologis dengan filter jenis transaksi (`IN` / `OUT`) dan badge visual.
7. **Pencarian, Filter, Sorting, & Pagination**:
   * Pencarian fleksibel nama/SKU produk (`LIKE %keyword%`).
   * Filter kategori berbasis *dropdown*.
   * Pengurutan data ASC/DESC.
   * Kontrol paginasi data (10 item per halaman) dengan URL query string yang tetap terjaga.
8. **Dashboard Ringkasan & Laporan Agregat (Slide 29)**:
   * 5 Widget statistik instan: **Total Produk**, **Nilai Inventory**, **Produk Stok Rendah**, **Barang Masuk**, dan **Barang Keluar**.
   * Tabel peringatan dini barang hampir habis (`stock <= minimum_stock`).
   * Tabel laporan mutasi agregat berbasis SQL.

---

## 5. ARSITEKTUR KEAMANAN APLIKASI (SECURITY ENGINEERING)

Sesuai materi perkuliahan **Slide 16–24**, aplikasi mengimplementasikan standar keamanan web:

### A. Pencegahan SQL Injection (Slide 18–19)
Seluruh query dinamis yang menerima input pengguna menggunakan **PDO Prepared Statements** dengan parameter binding:
```php
// helpers/database & public/products/index.php
$stmt = $pdo->prepare("SELECT * FROM products WHERE name LIKE :search AND category_id = :cat");
$stmt->execute(['search' => '%' . $keyword . '%', 'cat' => $categoryId]);
```
Tidak ada satupun query SQL yang menggunakan konkatenasi string langsung (`$query = "SELECT ... " . $_GET['input']`).

### B. Pencegahan Cross-Site Scripting / XSS (Slide 20)
Setiap pencetakan data dinamis ke layar HTML dibungkus menggunakan fungsi helper encoding `e()`:
```php
// helpers/escape.php
function e(?string $value): string {
    return htmlspecialchars($value ?? '', ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}
```
Contoh penggunaan di view:
```php
<td><?= e($product['name']) ?></td>
```

### C. Password Hashing Aman (Slide 22)
Password tidak disimpan dalam bentuk teks polos (*plain text*), melainkan menggunakan algoritma hashing modern standar PHP:
* Pembuatan Hash: `password_hash($plainPassword, PASSWORD_BCRYPT)`
* Verifikasi Login: `password_verify($inputPassword, $user['password_hash'])`

### D. Sorting Allowlist Guard (Slide 16 & 33)
Parameter pengurutan dibatasi secara ketat untuk mencegah SQL Injection pada klausa `ORDER BY`:
```php
$allowedColumns = ['name' => 'p.name', 'price' => 'p.price', 'stock' => 'p.stock', 'sku' => 'p.sku', 'id' => 'p.id'];
$sortCol = array_key_exists($_GET['sort'] ?? '', $allowedColumns) ? $allowedColumns[$_GET['sort']] : 'p.id';
$sortDir = strtoupper($_GET['dir'] ?? 'ASC') === 'DESC' ? 'DESC' : 'ASC';
```

---

## 6. TRANSAKSI STOK & ROW-LEVEL LOCKING (ACID)

Sesuai materi **Slide 26–27**, transaksi barang keluar (Stock OUT) tidak boleh hanya menggunakan query UPDATE biasa karena rentan terhadap *race condition* (ketika 2 request terjadi pada milidetik yang sama sehingga stok bisa lolos menjadi minus).

Sistem menangani ini dengan **Pessimistic Locking & Database Transaction**:
```php
// public/stock/out.php
$pdo->beginTransaction();
try {
    // 1. Kunci baris data produk agar tidak dibaca/diubah proses lain sampai commit
    $stmt = $pdo->prepare("SELECT id, name, stock FROM products WHERE id = :id FOR UPDATE");
    $stmt->execute(['id' => $productId]);
    $product = $stmt->fetch();

    // 2. Validasi stok secara semantik di dalam kunci transaksi
    if ($qtyOut > (int) $product['stock']) {
        throw new Exception("Stok tidak mencukupi! Tersedia: {$product['stock']}, Diminta: {$qtyOut}");
    }

    // 3. Potong stok produk
    $updateStmt = $pdo->prepare("UPDATE products SET stock = stock - :qty WHERE id = :id");
    $updateStmt->execute(['qty' => $qtyOut, 'id' => $productId]);

    // 4. Catat histori audit trail mutasi stok
    $histStmt = $pdo->prepare("INSERT INTO stock_movements (product_id, user_id, type, quantity, notes) VALUES (:p, :u, 'OUT', :q, :n)");
    $histStmt->execute(['p' => $productId, 'u' => $userId, 'q' => $qtyOut, 'n' => $notes]);

    // 5. Simpan seluruh perubahan secara permanen (Atomik)
    $pdo->commit();
} catch (Exception $e) {
    // Jika ada kegagalan, batalkan seluruh perubahan (Rollback)
    $pdo->rollBack();
    $errorMessage = $e->getMessage();
}
```

---

## 7. STRUKTUR FOLDER & MANIFEST BERKAS

Struktur direktori disusun modular sesuai dengan rekomendasi silabus **Slide 34**:

```text
inventory-app/
├── config/
│   └── database.php          # Konfigurasi koneksi PDO MySQL & auto-initializer tabel
├── database/
│   └── schema.sql            # Skema lengkap 7 tabel DDL & seed data uji
├── helpers/
│   ├── auth.php              # Helper Session, Login, Logout, dan RBAC (requireRole)
│   ├── escape.php            # Helper Output Encoding anti-XSS: e()
│   └── validation.php        # Helper Validasi input angka, positif, format teks
├── public/
│   ├── index.php             # Router/Entry point: redirect ke dashboard atau login
│   ├── login.php             # Form login & autentikasi password_verify
│   ├── logout.php            # Hapus sesi pengguna & redirect
│   ├── dashboard.php         # Dashboard 5 widget agregat & peringatan stok minimum
│   ├── categories/           # CRUD Master Kategori (index, create, edit, delete)
│   ├── products/             # CRUD Master Produk, Search/Filter/Sort/Page, & suppliers.php
│   ├── suppliers/            # CRUD Master Supplier (index, create, edit, delete)
│   ├── stock/                # Form Barang Masuk (in.php), Keluar (out.php), Histori (history.php)
│   └── users/                # Kelola Pengguna & Profil (Admin Only, Role Guard 403)
├── tests/
│   └── verify_project.php    # Script pengujian otomatis kepatuhan silabus (8 test suite)
├── views/
│   ├── header.php            # Template atas HTML, Navbar Bootstrap 5, Menu Navigasi
│   └── footer.php            # Template bawah HTML, Footer, Script Bootstrap JS
└── README.md                 # Dokumentasi teknis & panduan tugas lengkap
```

---

## 8. PANDUAN LANGKAH INSTALASI & MENJALANKAN PROYEK

Aplikasi dapat dijalankan secara langsung di lingkungan lokal Windows menggunakan **XAMPP**:

### Langkah 1: Pindahkan Folder ke `htdocs`
Pastikan folder proyek berada di direktori web server XAMPP:
```text
C:\xampp\htdocs\inventory-app
```

### Langkah 2: Aktifkan Modul Apache & MySQL
1. Buka aplikasi **XAMPP Control Panel**.
2. Klik tombol **Start** pada modul **Apache** (Port 80/443).
3. Klik tombol **Start** pada modul **MySQL** (Port 3306).

### Langkah 3: Import Database `inventory_db`
Ada dua cara mudah untuk mengimpor database:

* **Cara A (Melalui phpMyAdmin):**
  1. Buka browser dan akses `http://localhost/phpmyadmin`.
  2. Klik menu **Import**.
  3. Pilih berkas `C:\xampp\htdocs\inventory-app\database\schema.sql`.
  4. Klik tombol **Import** di bagian bawah.

* **Cara B (Melalui Command Line / Terminal):**
  Buka terminal PowerShell dan jalankan:
  ```powershell
  C:\xampp\mysql\bin\mysql.exe -u root < "C:\xampp\htdocs\inventory-app\database\schema.sql"
  ```

*(Catatan: Aplikasi juga dilengkapi fitur otomatis membuat tabel dan seed data jika database terhubung dalam keadaan kosong).*

### Langkah 4: Akses Aplikasi di Browser
Buka browser favorit Anda dan akses URL berikut:
👉 **`http://localhost/inventory-app/public/`**

Atau jika Anda menggunakan PHP Built-in Server dari terminal:
```powershell
cd C:\xampp\htdocs\inventory-app
php -S localhost:8000 -t public
```
Akses di: `http://localhost:8000`

### Langkah 5: Jalankan Pengujian Otomatis (Opsional)
Untuk membuktikan bahwa seluruh fungsi, helper, dan struktur file memenuhi 100% kriteria Slide 34:
```powershell
php tests/verify_project.php
```
Hasil audit akan menampilkan: `FINAL RESULT: 8 / 8 AUDIT CHECKS PASSED (100%)`.

---

## 9. AKUN PENGUJIAN (DEMO CREDENTIALS)

Database telah dilengkapi dengan akun bawaan untuk menguji otorisasi multi-role:

| Peran (Role) | Username | Password | Hak Akses Fitur |
|:---:|:---:|:---:|---|
| **Administrator** | `admin` | `admin123` | **Akses Penuh:** Dashboard, Master Produk, Kategori, Supplier, Transaksi Masuk/Keluar, Histori, dan Administrasi Pengguna (`/users/`). |
| **Staff Gudang** | `staff` | `staff123` | **Operasional Terbatas:** Dashboard, Lihat & Tambah Produk, Kategori, Supplier, Transaksi Masuk/Keluar, Histori. **Dilarang mengakses menu Pengguna** (`HTTP 403 Forbidden`). |

---

## 10. CHECKLIST EVALUASI PENGUMPULAN (SLIDE 36)

Tabel berikut membuktikan secara teknis bahwa seluruh 9 kriteria kelulusan dari **Slide 36 (Checklist Pengumpulan)** telah terpenuhi secara utuh:

| No | Kriteria Checklist (Slide 36) | Status | File Implementasi | Bukti & Mekanisme Teknis |
|:--:|---|:---:|---|---|
| **1** | **Relasi dan foreign key berjalan** | `[x] LENGKAP` | `database/schema.sql` | 7 tabel terhubung menggunakan foreign key dengan aturan `CASCADE` (pada profil & relasi supplier) dan `RESTRICT` (pada kategori, produk, mutasi stok). |
| **2** | **SKU unik dan validasi server aktif** | `[x] LENGKAP` | `public/products/create.php`, `helpers/validation.php` | Kolom `sku` berindeks `UNIQUE`. Validasi server memblokir SKU ganda, harga negatif, dan kuantitas $\le 0$. |
| **3** | **Login, logout, session, dan role berjalan** | `[x] LENGKAP` | `public/login.php`, `public/logout.php`, `helpers/auth.php` | Autentikasi menggunakan session aman dan `password_verify()`. Role `admin` dan `staff` ditegakkan lewat `requireRole()`. |
| **4** | **Tidak ada query concatenation dari input user** | `[x] LENGKAP` | Seluruh berkas di `public/` | 100% query SQL yang memproses input pengguna menggunakan PDO Prepared Statements dengan parameter placeholder berformat `:param`. |
| **5** | **Output HTML menggunakan escaping** | `[x] LENGKAP` | `helpers/escape.php`, seluruh views | Seluruh pencetakan string data pengguna diapit oleh fungsi `e()`, mengonversi karakter berbahaya (`<`, `>`, `"`, `'`, `&`) menjadi HTML entities aman. |
| **6** | **Search, filter, sorting, pagination berjalan** | `[x] LENGKAP` | `public/products/index.php` | Pencarian teks `LIKE %...%`, filter kategori, sorting kolom aman via *Allowlist*, dan paginasi data 10 per halaman dengan query URL persisten. |
| **7** | **Stok tidak negatif dan histori tersimpan** | `[x] LENGKAP` | `public/stock/out.php`, `database/schema.sql` | Pengecekan kuantitas keluar terhadap saldo stok fisik mencegah stok minus. Setiap mutasi langsung dicatat ke tabel `stock_movements`. |
| **8** | **Transaksi menggunakan commit/rollback** | `[x] LENGKAP` | `public/stock/in.php`, `public/stock/out.php` | Dibungkus dalam blok `try { $pdo->beginTransaction(); ... $pdo->commit(); } catch { $pdo->rollBack(); }` dan penguncian baris `SELECT ... FOR UPDATE`. |
| **9** | **Dashboard menampilkan laporan agregat** | `[x] LENGKAP` | `public/dashboard.php` | Menampilkan 5 widget ringkasan (`COUNT`, `SUM`) dan tabel laporan mutasi gabungan berbasis *Aggregate Subquery & GROUP BY* (Slide 29). |

---

## 11. PANDUAN SCREENSHOT PENGUJIAN & DEMO FITUR (SLIDE 38)

Sesuai instruksi **Slide 38**, siapkan berkas tangkapan layar (*screenshot*) berikut untuk melengkapi laporan tugas Anda:

### 📸 Screenshot 1: Halaman Login & Keamanan Password Hashing
* **URL**: `http://localhost/inventory-app/public/login.php`
* **Objek**: Formulir login yang rapi dengan kartu informasi akun demo (`admin` dan `staff`).

### 📸 Screenshot 2: Dashboard Ringkasan & Laporan Agregat (Slide 29)
* **URL**: `http://localhost/inventory-app/public/dashboard.php`
* **Objek**:
  1. 5 Widget kartu ringkasan: Total Produk, Nilai Inventory, Stok Rendah, Total Barang Masuk, Total Barang Keluar.
  2. Tabel Peringatan Produk Stok Rendah (`stock <= minimum_stock`).
  3. Tabel Laporan Rincian Pergerakan Barang Agregat.

### 📸 Screenshot 3: Katalog Produk (Search, Filter, Sorting, Pagination)
* **URL**: `http://localhost/inventory-app/public/products/index.php`
* **Objek**:
  1. Pengujian kolom pencarian kata kunci nama/SKU.
  2. Pengujian filter kategori barang.
  3. Tombol pengurutan naik/turun (*Sort Direction*).
  4. Komponen tombol halaman (*Pagination 1, 2, ...*) di bagian bawah tabel.

### 📸 Screenshot 4: Relasi Many-to-Many Produk ↔ Supplier
* **URL**: Klik tombol **Supplier** pada salah satu baris produk (`public/products/suppliers.php?id=1`).
* **Objek**: Tabel daftar supplier yang memasok produk terpilih beserta form penambahan relasi supplier dan harga beli baru.

### 📸 Screenshot 5: Pengujian Kasus Validasi Error (Slide 38)
* **URL**: `http://localhost/inventory-app/public/products/create.php`
* **Aksi**: Kosongkan nama produk, masukkan SKU yang sudah terdaftar, atau isi harga negatif.
* **Objek**: Kotak pesan peringatan merah (*Alert Danger*) yang menolak input dan mencantumkan daftar kesalahan validasi dari server.

### 📸 Screenshot 6: Transaksi Barang Masuk (IN) & Barang Keluar (OUT)
* **Barang Masuk**: `public/stock/in.php` $\rightarrow$ Bukti penambahan stok secara sukses.
* **Barang Keluar (Error Guard)**: `public/stock/out.php` $\rightarrow$ Masukkan jumlah melebihi stok yang ada (misal: 9999). Ambil screenshot pesan penolakan: *"Jumlah barang keluar melebihi stok yang tersedia. Stok tidak boleh menjadi negatif."*
* **Histori Transaksi**: `public/stock/history.php` $\rightarrow$ Tabel audit trail mutasi dengan badge hijau (IN) dan merah (OUT).

### 📸 Screenshot 7: Pengujian Kasus Akses Terlarang / Role Guard 403 (Slide 38)
* **Aksi**:
  1. Logout dari akun admin.
  2. Login menggunakan akun staff (`staff` / `staff123`).
  3. Coba ketikkan URL administrasi pengguna secara langsung di browser: `http://localhost/inventory-app/public/users/index.php`.
* **Objek**: Tampilan halaman penolakan dengan kode status **HTTP 403 Forbidden** (*"Akses ditolak: Halaman ini hanya dapat diakses oleh Administrator"*).

### 📸 Screenshot 8: Uji Integritas Referensial (Delete Constraint)
* **Aksi**: Login kembali sebagai admin, lalu coba hapus produk yang telah memiliki histori transaksi mutasi stok.
* **Objek**: Pesan kesalahan sistem: *"Gagal menghapus: Produk ini sudah memiliki histori transaksi stok (Referential Integrity Constraint)."*

### 📸 Screenshot 9: Hasil Eksekusi Audit Suite Otomatis
* **Terminal**: Jalankan perintah `php tests/verify_project.php`.
* **Objek**: Output terminal yang menampilkan `FINAL RESULT: 8 / 8 AUDIT CHECKS PASSED (100%)`.
