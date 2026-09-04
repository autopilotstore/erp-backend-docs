# PRD — REST API Doctype User (Frappe Core)

> Dokumen spesifikasi pemanggilan REST API untuk **User** (akun login) di ERPNext (Frappe),
> diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Core (Frappe)
- **Doctype:** `User`
- **Versi API:** `/api/resource/...` (API v1)
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

---

Dokumentasi REST API untuk doctype **User** (bawaan Frappe, `frappe.core`), mengikuti pola yang sama dengan
[Supplier](./prd_supplier.md), [Customer](./prd_customer.md), [Contact](./prd_contact.md),
[Address](./prd_address.md), [Lead](./prd_lead.md), dan [Employee](./prd_employee.md). Autentikasi
(OAuth 2.0) dan penanganan error umum berlaku sama (lihat [prd_supplier.md §3](./prd_supplier.md) /
[`prd_oauth.md`](../prd_oauth.md) dan [prd_supplier.md §6](./prd_supplier.md)).

> **Peran User:** User adalah **akun login** sistem (email + roles + user_type), bukan master
> *party* seperti Customer/Supplier. Relasi antar-akun & hak akses direpresentasikan lewat
> **Role**, **Role Profile**, **User Type**, dan **User Permission** (lihat §2.2).

### Ruang lingkup — User

| # | Endpoint | Metode | Keterangan |
|---|---|---|---|
| 1 | `/api/resource/User` | POST | Buat User baru (CREATE) |
| 2 | `/api/resource/User/{name}` | GET | Ambil detail 1 User (READ) |
| 3 | `/api/resource/User` | GET | Daftar User (READ list) |
| 4 | `/api/resource/User/{name}` | PUT | Ubah User (UPDATE) |
| 5 | `/api/resource/User/{name}` | PUT | Non-aktifkan User (`enabled=0`) — pengganti DELETE |
| 6 | `/api/resource/Role` | GET | Daftar Role (dropdown, §5.1) |
| 7 | `/api/resource/Role Profile` | GET | Daftar Role Profile (dropdown, §5.2) |
| 8 | `/api/resource/User Type` | GET | Daftar User Type (opsi `user_type`, §5.3) |
| 9 | `/api/resource/User Permission` | CRUD | Atur User Permission (pembatasan data, §4.6) |
| 10 | `/api/method/frappe.core.page.permission_manager.permission_manager.update` | POST | Override permission role (tanpa desk, §8.4) |
| 11 | `/api/method/frappe.client.get_count` | GET | Total record User sesuai filter — untuk pagination / lazy loading (§4.2) |

> Sesuai keputusan, dokumen ini **tidak** membahas export/import User, serta **tidak** membahas
> alur password/verifikasi email (dilewati sementara — default `send_welcome_email` tetap berjalan,
> lihat §1.1 poin 8). Setup jabatan + permission (tanpa desk) didokumentasikan lengkap di §8.

---

## 1. Ringkasan field & data wajib — User

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `email` | Data (Email) | Email user. `reqd: 1`. **Sekaligus menjadi `name`** (autoname, di-lowercase). |
| 🔴 **WAJIB** | `first_name` | Data | Nama depan. `reqd: 1`. |
| 🟢 Opsional | `middle_name`, `last_name` | Data | Nama tengah/belakang. `full_name` dihitung dari ketiganya. |
| 🟢 Opsional | `username` | Data (unique) | Username unik. Jika kosong, di-generate dari `first_name` (`scrub`). |
| 🟢 Opsional | `mobile_no` | Data (unique) | Nomor HP (unik). Bisa dipakai login bila setting `allow_login_using_mobile_number`. |
| 🟢 Opsional | `gender`, `language`, `time_zone`, `birth_date`, `phone`, `user_image`, dll. | — | Data profil tambahan. |
| 🟠 **Permlevel 1** | `roles` | Table (`Has Role`) | Daftar Role user. Hanya bisa ditulis user ber-role **System Manager**. |
| 🟠 **Permlevel 1** | `role_profiles` | Table MultiSelect (`User Role Profile`) | Bundel Role Profile. Saat save, roles dari profile **diperluas** ke `roles`. |
| 🟠 **Permlevel 1** | `user_type` | Link → User Type | `System User` / `Website User` / tipe custom. **Dihitung otomatis** bila tidak dikirim. |
| 🟠 **Permlevel 1** | `module_profile`, `api_key`, `api_secret`, `block_modules` | — | Profil modul & API keys (umumnya tidak dipakai frontend). |
| 🟢 Opsional (khusus) | `new_password` | Password | Set password saat CREATE. **Tidak dilewati dalam scope saat ini** (§1.1 poin 8). |
| ⚪ **Otomatis — jangan dikirim** | `name`, `enabled` | — | `name` = email (autoname). `enabled` default `1`. |
| ⚪ **Read-only — jangan dikirim** | `full_name`, `last_login`, `last_ip`, `last_active`, `api_key`, `api_secret` | — | `full_name` dihitung; sisanya diisi sistem. |

### 1.1 Catatan penting — User

1. **`name` = `email`** — kunci dokumen User adalah email (lowercase). Semua operasi
   GET/PUT/non-aktif memakai `name` (= email). Contoh: `budi@perusahaan.co.id`.
2. **`full_name` read-only** — dihitung dari `first_name + middle_name + last_name`. Jangan dikirim.
3. **`user_type` otomatis** (`set_system_user`) — bila tidak dikirim (atau dikirim `System User`),
   backend menghitung: ada role dengan `desk_access=1` → `System User`; jika tidak → `Website User`.
   `Administrator` selalu `System User`; `Guest` selalu `Website User` (tidak bisa diubah).
   **Perbedaan `System User` vs `Website User`** (panduan proyek):
   - **`System User`** → user **internal perusahaan** (karyawan/pengelola data, mis. Sales Manager,
     Kasir, Admin). Terhitung sebagai *system user*: punya akses desk (backend) secara default dan
     diikutkan pada dropdown/filter sistem yang mengecualikan website user. Meski proyek **tidak
     memakai UI desk**, user internal tetap dibuat `System User` (lewat role ber-`desk_access=1`)
     agar berperilaku seperti pengguna sistem.
   - **`Website User`** → user **eksternal** (contact person supplier/customer, portal). Tidak
     punya akses desk dan tidak terhitung sebagai *system user*. Role portal bawaan `Supplier` /
     `Customer` sudah ber-`desk_access=0` (lihat §8.9).
   > **Penting:** `user_type` **tidak menentukan** hak ubah data — akses baca/tulis data ditentukan
   > oleh **permission role** pada tiap doctype (bukan `user_type`). `Website User` yang memegang
   > role ber-`write` tetap bisa mengubah data lewat API; bedanya hanya pada akses desk dan konteks
   > "system user" di sebagian method/dropdown.
4. **`roles` & `role_profiles` permlevel 1** — hanya bisa diubah oleh user dengan akses permlevel-1
   pada doctype User, praktis role **`System Manager`**. Membuat User pun praktis butuh
   `System Manager` (karena field permlevel-1 diisi).
5. **`role_profiles` memperluas ke `roles`** (`populate_role_profile_roles`) — saat save, `roles`
   disaring ke gabungan roles seluruh profile lalu di-append. **Aturan praktis:** kirim `roles`
   langsung ATAU `role_profiles`, jangan campur tanpa sadar (bisa menimpa).
   Field `role_profile_name` (Link) **deprecated** — gunakan `role_profiles`.
6. **User tanpa roles** = boleh dibuat (hanya peringatan *"no roles enabled"*), tetapi **tidak punya
   akses apa pun** (403 di semua resource) dan `user_type` jadi `Website User`.
7. **Standard users** — `Administrator` dan `Guest` **tidak bisa** dinonaktifkan, direname, atau
   dihapus. Non-aktifkan user lain pakai `enabled=0` (§4.5).
8. **Password / welcome email (dilewati sementara)** — field `new_password` (Password) dipakai
   untuk set password saat CREATE; jika tidak dikirim, default `send_welcome_email=1` membuat
   backend mengirim email berisi link set password (butuh Email Account outgoing; bila tidak ada,
   muncul peringatan `OutgoingEmailError`). Keputusan saat ini: **tidak dibahas**; alur login
   frontend ditentukan belakangan.
9. **Role yang dibutuhkan** — baca daftar & dropdown User: role dengan hak baca `User` (mis.
   `System Manager`, `Employee` utk dirinya sendiri); buat/ubah/permission: **`System Manager`**;
   kelola User Permission/Role/Role Profile: **`System Manager`**.
10. **Throttle pembuatan user** — `throttle_user_creation()` membatasi pembuatan User per jam
    (default `throttle_user_limit` = 60/jam). Pembuatan massal bisa kena error `Throttled`.
11. **`roles` / `role_profiles` — wajib diisi salah satu** (aturan frontend): CREATE/UPDATE User
    **harus** menyertakan minimal satu dari `roles` atau `role_profiles`. Bila keduanya kosong,
    frontend memblokir dengan pesan *"User harus memiliki minimal satu role."* (Catatan teknis:
    Frappe sebenarnya mengizinkan user tanpa roles — hanya peringatan *"no roles enabled"* — tapi
    user tanpa roles tidak punya akses apa pun, sehingga aturan ini diberlakukan.) Hindari mengirim
    **keduanya sekaligus**: `role_profiles` akan menimpa/mereset `roles` (lihat poin 5).
12. **`roles` bisa multi** — child table `roles` menerima banyak baris; satu user boleh punya
    banyak role sekaligus (akses = gabungan permission semua role-nya). Contoh payload **multi
    roles** dengan role bawaan ERPNext (`Sales User` + `Accounts User`) ada di §4.1.
13. **User eksternal (contact person supplier/customer) → `Website User`** — akun untuk pihak
    luar dibuat dengan role portal bawaan `Supplier` / `Customer` (keduanya ber-`desk_access=0`),
    sehingga `user_type` otomatis `Website User`. Tautkan akun ke Contact-nya via field `user`
    (Link → User). Contoh alur lengkap: §8.9.

---

## 2. Relasi User dengan doctype lain

| Relasi | Cara diwakili | Catatan dari kode |
|---|---|---|
| **Role** | child table `roles` (`Has Role`) | Assign lewat `roles` di body POST/PUT. Role dengan `desk_access=1` → `System User`. Role `disabled=1` otomatis dibuang dari user. |
| **Role Profile** | child table `role_profiles` (`User Role Profile`) | `populate_role_profile_roles()` memperluas roles profile ke `roles`. Boleh multi. |
| **User Type** | `user_type` (Link → `User Type`) | `System User`/`Website User`/custom. Dihitung otomatis bila kosong. |
| **Module Profile** | `module_profile` (Link → `Module Profile`) | Diekspansi ke child table `block_modules`. |
| **User Permission** | doctype terpisah `User Permission` | `user`, `allow` (DocType), `for_value` (Dynamic Link). Dikelola via `/api/resource/User Permission` (§4.6). Saat User dihapus, User Permission ikut terhapus. |
| **Contact** | field `user` di Contact | `on_update()` → `create_contact()` **otomatis membuat/memperbarui Contact** dari User (first/last name, gender, email, phone, mobile_no). Saat User dihapus, `Contact.user` di-unset. |
| **Employee** | `Employee.user_id` (Link → User) — arah terbalik | Lihat [prd_employee.md §1.1](./prd_employee.md): saat Employee disimpan dengan `user_id`, nama/DOB/gender/image disinkron ke User + role `Employee` ditambahkan. |
| **Lead / dll.** | `lead_owner`, `allocated_to`, owner | User dipakai sebagai owner/assignee lintas doctype. |

**Catatan relasi User ↔ Contact:**

- Tautan disimpan di **field `user` pada dokumen Contact** (kolom `tabContact.user`, Link → User),
  nilainya = `name` User (= email). **Arah tautan searah Contact → User** — doctype User **tidak**
  punya field yang menunjuk ke Contact (source of truth ada di `tabContact.user`).
- `create_contact()` (dipicu `on_update` User) otomatis **membuat** Contact baru bila email belum
  ada, atau **meng-update** Contact yang email-nya sama (lihat §4.1). Tautan otomatis `Contact.user`
  terisi hanya bila email User = email **primer** Contact.
- Saat User dihapus, `Contact.user` di-unset (`on_trash`).
- **Cara verifikasi via API:**
  - Dari sisi Contact: `GET /api/resource/Contact/{name}` → baca field `user`.
  - Dari sisi User (mencari Contact yang ter-link):
    ```bash
    curl -G "https://site-anda.com/api/resource/Contact" \
      -H 'Authorization: Bearer <access_token>' \
      --data-urlencode 'fields=["name","full_name","email_id","user"]' \
      --data-urlencode 'filters=[["user","=","budi@perusahaan.co.id"]]' \
      --data-urlencode 'limit_page_length=1'
    ```

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Detail lengkap setup OAuth Client, alur mendapatkan/memperbarui/mencabut token ada di
**[`prd_oauth.md`](../prd_oauth.md)**.

> ⚠️ Sebagian besar operasi di dokumen ini (buat/ubah User, Role, Role Profile, User Permission,
> override permission) **wajib memakai token milik user ber-role `System Manager`** (atau
> `Administrator`). Token OAuth user biasa akan ditolak (403). Lihat §8.1.

---

## 4. CRUD — Doctype User

### 4.1 CREATE — `POST /api/resource/User`

**Langkah 0 — Pre-check (disarankan sebelum POST)**

Karena `name` = email dan email harus unik, cek dulu apakah email sudah dipakai:

```bash
curl -G "https://site-anda.com/api/resource/User" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","enabled"]' \
  --data-urlencode 'filters=[["name","=","budi@perusahaan.co.id"]]' \
  --data-urlencode 'limit_page_length=1'
```

- `data` kosong (`[]`) → lanjut POST.
- `data` terisi → blokir dengan pesan *"Email sudah terdaftar sebagai user."* (tanpa pre-check,
  backend melempar `DuplicateEntryError`).

**Payload minimum (wajib) + roles (bisa multi):**

```json
{
  "email": "budi@perusahaan.co.id",
  "first_name": "Budi",
  "roles": [
    { "role": "Sales User" },
    { "role": "Accounts User" }
  ]
}
```

**Contoh request:**

```bash
curl -X POST https://site-anda.com/api/resource/User \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "email": "budi@perusahaan.co.id",
    "first_name": "Budi",
    "roles": [
      { "role": "Sales User" },
      { "role": "Accounts User" }
    ]
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "data": {
    "name": "budi@perusahaan.co.id",
    "owner": "Administrator",
    "creation": "2026-08-20 10:00:00.000000",
    "modified": "2026-08-20 10:00:00.000000",
    "enabled": 1,
    "email": "budi@perusahaan.co.id",
    "first_name": "Budi",
    "full_name": "Budi",
    "username": "budi",
    "user_type": "System User",
    "roles": [
      { "name": "abc123", "role": "Sales User" },
      { "name": "def456", "role": "Accounts User" }
    ],
    "role_profiles": []
  }
}
```

> `name` = `budi@perusahaan.co.id` → simpan nilai ini (dipakai GET/PUT/non-aktif).
> Contoh ini memakai **multi roles** (role bawaan ERPNext: `Sales User` + `Accounts User`).
> `user_type` dihitung otomatis: `System User` (internal) karena role bawaan `Sales User`
> ber-`desk_access=1`; untuk user eksternal/portal gunakan role tanpa desk access →
> `Website User` (lihat §1.1 poin 3).

**Efek samping otomatis (penting):**
- **Auto-create Contact** — `on_update()` menjalankan `create_contact()`: membuat/memperbarui
  Contact (first/last name, gender, email, phone, mobile_no) yang ter-link ke User via field
  `user`. Frontend tidak perlu membuat Contact manual untuk akun User.
- **Welcome email** — bila `send_welcome_email` (default 1) dan tidak ada `new_password`,
  backend mengirim email berisi link set password (lihat §1.1 poin 8).

**Menautkan User ke Contact yang sudah ada (email yang sama):**

Jika `email` yang dipakai untuk User **sudah terdaftar** di sebuah Contact (di child table
`email_ids`), `create_contact()` **tidak membuat Contact baru** — ia menemukan Contact tersebut
(`get_contact_name`) dan **meng-update-nya** (sinkron `first_name`/`last_name`/`gender`, tambah
`phone`/`mobile_no` bila belum ada). Dengan kata lain: **tidak ada duplikat Contact** selama email
cocok.

> **Cara kerjanya di kode:**
> - Pencocokan memakai child table **`email_ids`** (`tabContact Email`) — cocok dengan email apa pun,
>   termasuk yang bukan primer.
> - Tautan otomatis `Contact.user` diisi lewat `set_user()`, yang memakai **email primer**
>   (`email_id`) Contact:
>   - Email User = **email primer** Contact → `Contact.user` otomatis terisi (tertaut). ✅
>   - Email User hanya **non-primer** → Contact tetap di-update, tapi `Contact.user` **tidak
>     otomatis** terisi → set manual:
>     ```json
>     PUT /api/resource/Contact/{name}
>     { "user": "budi@perusahaan.co.id" }
>     ```
> - Proses ini **async** (background job `frappe.enqueue`, `enqueue_after_commit=True`) — tautan/
>   update terjadi sesaat setelah respons POST.
> - Nama depan/belakang & gender Contact **ditimpa** dengan nilai User (sinkron User → Contact).
> - Untuk user eksternal (contact person supplier/customer), contoh alur lengkap ada di §8.9.

### 4.2 READ (satu record) — `GET /api/resource/User/{name}`

```bash
curl -X GET "https://site-anda.com/api/resource/User/budi@perusahaan.co.id" \
  -H 'Authorization: Bearer <access_token>'
```

Respons `data` berisi seluruh field User (termasuk `roles[]`, `role_profiles[]`, `user_type`).

**Total record count — `GET /api/method/frappe.client.get_count`**

Untuk kebutuhan pagination / lazy loading (mis. menampilkan "900 dari 1000 User"), ambil
**total record yang cocok dengan filter** lewat method whitelisted `get_count`. Respons list
(`GET /api/resource/User`) **tidak** menyertakan total — hitung terpisah dengan endpoint ini.

```bash
curl -G "https://site-anda.com/api/method/frappe.client.get_count" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=User' \
  --data-urlencode 'filters=[["enabled","=",1]]'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": 900
}
```

> Parameter: `doctype` (wajib), `filters` (opsional, format sama seperti READ list).
> Respons berupa `message` (bukan `data`) = jumlah record yang cocok. Count menghormati filter
> **permission user** (angka sesuai hak akses user) dan akurat selama tidak mengirim param
> `limit`. Nilai ini dipakai untuk menghitung total halaman saat lazy loading di §4.3.

### 4.3 READ (daftar) — `GET /api/resource/User`

Parameter pilihan: `fields`, `filters`, `order_by`, `limit_start`, `limit_page_length`.

> **Lazy loading / pagination:** `limit_page_length` = jumlah record per halaman,
> `limit_start` = index awal (offset). Contoh ambil **per 50 data**:

```bash
# Halaman 1 — record 1–50
curl -G "https://site-anda.com/api/resource/User" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","full_name","email","enabled","user_type"]' \
  --data-urlencode 'filters=[["enabled","=",1]]' \
  --data-urlencode 'order_by=full_name asc' \
  --data-urlencode 'limit_start=0' \
  --data-urlencode 'limit_page_length=50'
```

> Halaman berikutnya naikkan `limit_start` kelipatan 50 (`50`, `100`, dst.) dengan
> `limit_page_length=50` tetap. Jumlah halaman = `total / 50`, dengan `total` diperoleh
> dari `get_count` (§4.2). `limit_page_length=0` = ambil **semua** record (tanpa LIMIT),
> seperti contoh di bawah ini.

```bash
curl -G "https://site-anda.com/api/resource/User" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","full_name","email","enabled","user_type"]' \
  --data-urlencode 'filters=[["enabled","=",1]]' \
  --data-urlencode 'order_by=full_name asc' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "budi@perusahaan.co.id",
      "full_name": "Budi",
      "email": "budi@perusahaan.co.id",
      "enabled": 1,
      "user_type": "System User"
    }
  ]
}
```

### 4.4 UPDATE — `PUT /api/resource/User/{name}`

Kirim **hanya field yang diubah**. Child table `roles` bersifat **replace-all** (kirim seluruh
daftar roles yang diinginkan). Contoh menambah role + mengganti nama:

```bash
curl -X PUT "https://site-anda.com/api/resource/User/budi@perusahaan.co.id" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "last_name": "Pratama",
    "roles": [
      { "role": "Sales User" },
      { "role": "Accounts User" },
      { "role": "Purchase User" }
    ]
  }'
```

**Contoh respons (HTTP 200):** objek `data` berisi dokumen terbaru (roles terbarui, `full_name`
dihitung ulang).

> **Catatan:**
> - `roles` **replace-all** — jangan lupa sertakan role lama yang masih diinginkan.
> - Mengirim `role_profiles` akan **menimpa/mereset `roles`** dari profil (lihat §1.1 poin 5).
> - Mengganti `user_type` ke tipe custom non-standar dapat **menimpa `roles`** (`set_system_user`).
> - Mengganti `email` lewat API **tidak** otomatis me-rename `name` — hindari rename email via
>   REST; gunakan mekanisme rename resmi bila diperlukan.

### 4.5 Non-aktifkan (disarankan) — `PUT /api/resource/User/{name}`

**Aturan:** User **tidak boleh dihapus**, hanya dinon-aktifkan. Non-aktifkan dengan `enabled=0`:

```bash
curl -X PUT "https://site-anda.com/api/resource/User/budi@perusahaan.co.id" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "enabled": 0 }'
```

**Contoh respons (HTTP 200):** objek `data` terbaru dengan `"enabled": 0`.

Untuk mengaktifkan kembali: `PUT` dengan `{ "enabled": 1 }`.

> **Catatan backend:** field `enabled` ditandai `read_only: 1` di doctype (permlevel 0). Secara
> praktik nilai ini bisa di-set lewat REST API, namun **perlu diverifikasi pada implementasi**
> (jika tertahan, gunakan `frappe.db.set_value` via kustomisasi/hook backend).
>
> **Efek non-aktif:** user tidak bisa login; sesi aktif di-logout (`check_enable_disable` →
> `logout`), notifikasi dimatikan. **Contact, roles, dan User Permission TIDAK dihapus.**
>
> ⚠️ **Jangan gunakan `DELETE /api/resource/User/{name}`** — selain tidak bisa untuk
> `Administrator`/`Guest`, `on_trash` menghapus banyak data terkait (User Permission, ToDo,
> DocShare, unset Contact.user, dll.) — melanggar aturan "jangan hapus" master data.

### 4.6 Kelola User Permission — CRUD `/api/resource/User Permission`

User Permission membatasi **record mana** yang boleh diakses user (bukan "bisa apa"). Doctype
terpisah; CRUD-nya standar:

**CREATE — batasi user hanya pada 1 Warehouse:**

```bash
curl -X POST https://site-anda.com/api/resource/User%20Permission \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "user": "budi@perusahaan.co.id",
    "allow": "Warehouse",
    "for_value": "Toko Pusat - MJ",
    "is_default": 1,
    "apply_to_all_doctypes": 1
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": {
    "name": "abc123",
    "user": "budi@perusahaan.co.id",
    "allow": "Warehouse",
    "for_value": "Toko Pusat - MJ",
    "is_default": 1,
    "apply_to_all_doctypes": 1
  }
}
```

**READ / UPDATE / DELETE** — `GET/PUT/DELETE /api/resource/User Permission/{name}` (kirim hanya
field berubah; `allow`/`for_value`/`user` bisa diganti).

> **Catatan:** `User Permission` hanya boleh dikelola role **`System Manager`**. User tanpa User
> Permission = melihat **semua** record yang diizinkan role-nya.

---

## 5. GET pendukung UI

### 5.1 GET Role (dropdown) — `GET /api/resource/Role`

```bash
curl -G "https://site-anda.com/api/resource/Role" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","role_name","disabled"]' \
  --data-urlencode 'filters=[["disabled","=",0]]' \
  --data-urlencode 'order_by=role_name asc' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    { "name": "Accounts User", "role_name": "Accounts User", "disabled": 0 },
    { "name": "Kasir", "role_name": "Kasir", "disabled": 0 },
    { "name": "Sales User", "role_name": "Sales User", "disabled": 0 }
  ]
}
```

> Untuk dropdown assign role di form User: ambil semua `disabled=0`. Kirim `name` (= `role_name`)
> ke child table `roles[].role`.

### 5.2 GET Role Profile (dropdown) — `GET /api/resource/Role Profile`

```bash
curl -G "https://site-anda.com/api/resource/Role%20Profile" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","role_profile"]' \
  --data-urlencode 'order_by=role_profile asc' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    { "name": "Kasir", "role_profile": "Kasir" },
    { "name": "Supervisor Toko", "role_profile": "Supervisor Toko" }
  ]
}
```

### 5.3 GET User Type (opsi `user_type`) — `GET /api/resource/User Type`

```bash
curl -G "https://site-anda.com/api/resource/User%20Type" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","is_standard","role"]' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    { "name": "System User", "is_standard": 1, "role": null },
    { "name": "Website User", "is_standard": 1, "role": null }
  ]
}
```

> Untuk frontend: umumnya **tidak perlu mengirim** `user_type` (otomatis, §1.1 poin 3). Dropdown
> hanya diperlukan bila ingin menetapkan tipe custom non-standar. `System User` = user internal
> perusahaan; `Website User` = user eksternal/portal (lihat §1.1 poin 3).

### 5.4 GET User (dropdown) — `GET /api/resource/User`

Dipakai untuk field Link→User di dokumen lain (mis. `lead_owner`, `user_id`), filter `enabled=1`:

```bash
curl -G "https://site-anda.com/api/resource/User" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","full_name","email"]' \
  --data-urlencode 'filters=[["enabled","=",1]]' \
  --data-urlencode 'order_by=full_name asc' \
  --data-urlencode 'limit_page_length=0'
```

---

## 6. Penanganan error umum

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | Role tidak punya akses (mis. bukan System Manager) | `{"message": "Not permitted"}` |
| 404 | Resource tidak ditemukan | `{"exc_type":"DoesNotExistError","message":"Resource Not Found"}` |
| 417 | Field wajib kosong (`reqd`) | `{"exc_type":"MandatoryError","message":"email is mandatory"}` |
| 417 | Email sudah dipakai (name duplikat) | `{"exc_type":"DuplicateEntryError","message":"... already exists"}` |
| 417 | Email tidak valid | `{"exc_type":"ValidationError","message":"Please enter a valid Email Address"}` |
| 429 | Pembuatan user melebihi throttle per jam | `{"exc_type":"ValidationError","message":"Throttled"}` |

---

## 7. Koleksi Postman

Seluruh pemanggilan pada dokumen ini mengikuti koleksi Postman yang sama:
`docs/postman/postman_erpnext_api.json` (Collection v2.1). Koleksi ini berisi seluruh modul API
ERPNext (OAuth 2.0 + Supplier + Customer + Contact + Address + Lead + Employee) dan **siap ditambah
modul baru** — termasuk folder *User & Setup Jabatan/Permission* bila dibutuhkan belakangan
(keputusan saat ini: request User didokumentasikan di dokumen ini, belum masuk koleksi).

---

## 8. Setup jabatan & permission TANPA DESK (panduan lengkap)

> Bagian ini menjelaskan cara **menyiapkan jabatan (role), user, dan permission** seluruhnya lewat
> REST API — **tanpa membuka Desk**. Cocok untuk proyek yang memakai frontend sendiri dan perlu
> proses *provisioning* akun yang terprogram dan terdokumentasi.

### 8.1 Prasyarat & konsep service provisioning

**Apa itu service provisioning?** Proses menyiapkan/mengalokasikan akun + hak akses secara
terprogram. Karena tidak memakai Desk, dibutuhkan satu "service" (Postman folder, script, atau
service backend) yang login sebagai admin dan memanggil REST API untuk membuat
Role → (Role Profile) → User → (User Permission) → mengatur permission.

**Prasyarat wajib:**
1. **Token OAuth milik user `System Manager`** (atau `Administrator`) — semua endpoint di bagian
   ini hanya untuk `System Manager` (`frappe.only_for("System Manager")`). Siapkan satu OAuth
   Client + user System Manager khusus untuk provisioning.
2. Header `Authorization: Bearer <access_token>` pada semua request (lihat §3).
3. Doctype yang dipakai role baru (mis. `Sales Invoice`, `Item`, `Customer`, `Warehouse`) sudah
   ada.

### 8.2 Konsep permission & peringatan penting

- **DocPerm vs Custom DocPerm** — permission standar doctype ada di `DocPerm`. Saat diubah,
  dibuat record **`Custom DocPerm`** yang menggantikan `DocPerm` untuk doctype tersebut.
- ⚠️ **Gotcha replace-total** (`Meta.set_custom_permissions`): jika ada `Custom DocPerm` untuk
  sebuah doctype, **seluruh** daftar permission doctype itu **diganti** dengan isi Custom DocPerm.
  Membuat Custom DocPerm untuk satu role saja → role lain kehilangan akses. **Hindari**
  `POST /api/resource/Custom DocPerm` manual kecuali mengisi semua role.
- **Permission = union (OR)** — akses user = gabungan permission semua role-nya. Tidak bisa
  "mengurangi" permission untuk satu user sementara role-nya tetap.
- **Cara aman**: gunakan method whitelisted `permission_manager.update` (§8.4) yang otomatis
  menyalin semua DocPerm standar dulu (`setup_custom_perms`) lalu mengubah satu property.

### 8.3 Toolkit API tanpa desk

| Aksi | Endpoint | Syarat |
|---|---|---|
| Buat Role | `POST /api/resource/Role` `{ "role_name": "Kasir" }` | System Manager |
| Buat Role Profile | `POST /api/resource/Role Profile` `{ "role_profile": "...", "roles": [...] }` | System Manager |
| Buat/assign User | `POST /api/resource/User` (dgn `roles`/`role_profiles`) | System Manager |
| Buat User Permission | `POST /api/resource/User Permission` | System Manager |
| **Override permission role** | `POST /api/method/frappe.core.page.permission_manager.permission_manager.update` | System Manager |
| Baca permission efektif | `...permission_manager.get_permissions?doctype=...` | System Manager |
| Baca permission standar | `...permission_manager.get_standard_permissions?doctype=...` | System Manager |
| Reset ke standar | `...permission_manager.reset` (body `{"doctype": ...}`) | System Manager |
| Tambah baris permission | `...permission_manager.add` (`parent`, `role`, `permlevel`) | System Manager |
| Hapus baris permission | `...permission_manager.remove` (`doctype`, `role`, `permlevel`) | System Manager |

### 8.4 Cara aman override permission role bawaan

Method `update` di `frappe.core.page.permission_manager.permission_manager` (whitelisted) memanggil
`update_permission_property()` yang mengeksekusi `setup_custom_perms(doctype)` — **menyalin SEMUA
DocPerm standar ke Custom DocPerm** sebelum mengubah satu property → role lain aman.

**Contoh: matikan `create` untuk role `Sales User` pada `Sales Order`:**

```bash
curl -X POST "https://site-anda.com/api/method/frappe.core.page.permission_manager.permission_manager.update" \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Sales Order",
    "role": "Sales User",
    "permlevel": 0,
    "ptype": "create",
    "value": 0
  }'
```

**Parameter `ptype`** yang tersedia: `select`, `read`, `write`, `create`, `delete`, `submit`,
`cancel`, `amend`, `report`, `export`, `import`, `share`, `print`, `email`, `if_owner`.

**Verifikasi hasil:**

```bash
curl -G "https://site-anda.com/api/method/frappe.core.page.permission_manager.permission_manager.get_permissions" \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  --data-urlencode 'doctype=Sales Order' \
  --data-urlencode 'role=Sales User'
```

**Reset ke permission standar (bila perlu):**

```bash
curl -X POST "https://site-anda.com/api/method/frappe.core.page.permission_manager.permission_manager.reset" \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Sales Order" }'
```

> ⚠️ **Efek global:** mengubah permission role bawaan (mis. `Sales User`) memengaruhi **SEMUA**
> user pemegang role tersebut. Untuk jabatan spesifik, lebih aman membuat **role custom** (studi
> kasus berikut).

### 8.5 Studi kasus A — Jabatan Kasir (role custom)

Tujuan: user kasir bisa membuat/lihat transaksi POS (mis. `Sales Invoice`) di tokonya, **tanpa**
bisa membuat `Sales Order`. Pendekatan: role custom dari nol (default tanpa akses).

**Langkah 1 — Buat role `Kasir`:**

```bash
curl -X POST https://site-anda.com/api/resource/Role \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{ "role_name": "Kasir", "desk_access": 1 }'
```

> Role baru default **tidak punya akses apa pun** (permission diatur di Langkah 2). `desk_access=1`
> dipakai agar `user_type` user-nya otomatis menjadi **`System User`** (internal perusahaan), sesuai
> panduan §1.1 poin 3. Flag ini menentukan `user_type`, bukan berarti proyek memakai UI desk.

**Langkah 2 — Beri permission yang pas (per property, per doctype):**

```bash
# baca + tulis Sales Invoice
curl -X POST "https://site-anda.com/api/method/frappe.core.page.permission_manager.permission_manager.update" \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Sales Invoice", "role": "Kasir", "permlevel": 0, "ptype": "read", "value": 1 }'
curl -X POST "https://site-anda.com/api/method/frappe.core.page.permission_manager.permission_manager.update" \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Sales Invoice", "role": "Kasir", "permlevel": 0, "ptype": "write", "value": 1 }'

# baca Item & Customer (untuk transaksi)
curl -X POST "https://site-anda.com/api/method/frappe.core.page.permission_manager.permission_manager.update" \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Item", "role": "Kasir", "permlevel": 0, "ptype": "read", "value": 1 }'
curl -X POST "https://site-anda.com/api/method/frappe.core.page.permission_manager.permission_manager.update" \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Customer", "role": "Kasir", "permlevel": 0, "ptype": "read", "value": 1 }'
```

> **Tidak** diberi permission `Sales Order` → kasir tidak bisa membuat/melihat Sales Order sama
> sekali (lebih ketat daripada sekadar mematikan `create` pada `Sales User`).

**Langkah 3 — Buat User kasir + assign role:**

```bash
curl -X POST https://site-anda.com/api/resource/User \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{
    "email": "kasir@perusahaan.co.id",
    "first_name": "Doni",
    "roles": [ { "role": "Kasir" } ]
  }'
```

**Langkah 4 — Batasi lingkup data per toko (User Permission):**

```bash
curl -X POST https://site-anda.com/api/resource/User%20Permission \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{
    "user": "kasir@perusahaan.co.id",
    "allow": "Warehouse",
    "for_value": "Toko Pusat - MJ",
    "is_default": 1
  }'
```

**Hasil:** user `kasir@...` bisa mengakses transaksi POS hanya di Warehouse *Toko Pusat - MJ*,
tanpa akses Sales Order.

### 8.6 Studi kasus B — Supervisor Toko (Role Profile)

Tujuan: jabatan dengan **kombinasi peran** (kasir + hak tambahan, mis. lihat laporan / set POS
Profile) yang rapi dibundel sebagai **Role Profile** agar mudah dipakai ulang.

**Langkah 1 — Buat Role Profile `Supervisor Toko` (menggabungkan peran kasir + hak tambahan):**

```bash
curl -X POST https://site-anda.com/api/resource/Role%20Profile \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{
    "role_profile": "Supervisor Toko",
    "roles": [
      { "role": "Kasir" },
      { "role": "Sales User" }
    ]
  }'
```

**Langkah 2 — Assign ke user via `role_profiles` (roles otomatis terisi dari profile):**

```bash
curl -X POST https://site-anda.com/api/resource/User \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{
    "email": "supervisor@perusahaan.co.id",
    "first_name": "Rina",
    "role_profiles": [
      { "role_profile": "Supervisor Toko" }
    ]
  }'
```

> Saat save, `roles` user diisi otomatis = `Kasir` + `Sales User` (`populate_role_profile_roles`).
> Satu user boleh punya **banyak** Role Profile — kirim beberapa baris `role_profiles`.

**Langkah 3 — (Opsional) User Permission lebih luas** (mis. semua Warehouse cabang):

```bash
curl -X POST https://site-anda.com/api/resource/User%20Permission \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{
    "user": "supervisor@perusahaan.co.id",
    "allow": "Cost Center",
    "for_value": "Pusat - MJ",
    "is_default": 1
  }'
```

### 8.7 Studi kasus C — Admin Toko (role luas + penyesuaian via override)

Tujuan: jabatan Admin Toko dengan akses lebih luas, sekaligus contoh **override** untuk
mengurangi hak role bawaan bila perlu (strategi "keduanya": role custom untuk jabatan + override
untuk menyesuaikan role bawaan).

**Langkah 1 — Buat role `Admin Toko` + beri permission (read/write/create) pada doctype operasional:**

```bash
curl -X POST https://site-anda.com/api/resource/Role \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{ "role_name": "Admin Toko", "desk_access": 1 }'
```

Beri permission sesuai kebutuhan (pola sama dengan §8.5 Langkah 2), mis. `Sales Invoice`
read/write/create, `Sales Order` read, `Warehouse` read, `Item` read/write.

**Langkah 2 — Assign role (bisa juga gabung profile):**

```bash
curl -X POST https://site-anda.com/api/resource/User \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{
    "email": "admin.toko@perusahaan.co.id",
    "first_name": "Agus",
    "roles": [ { "role": "Admin Toko" } ]
  }'
```

**Langkah 3 — Contoh override role bawaan:** bila tim memakai role bawaan `Sales User` untuk
jabatan lain dan ingin **menonaktifkan `create` Sales Order** pada role itu (berlaku untuk semua
pemegang `Sales User`), gunakan §8.4:

```bash
curl -X POST "https://site-anda.com/api/method/frappe.core.page.permission_manager.permission_manager.update" \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Sales Order",
    "role": "Sales User",
    "permlevel": 0,
    "ptype": "create",
    "value": 0
  }'
```

### 8.8 Ringkasan urutan panggilan per jabatan

| Urutan | Kasir (§8.5) | Supervisor (§8.6) | Admin (§8.7) |
|---|---|---|---|
| 1 | POST Role `Kasir` | POST Role `Kasir` (+ `Sales User` bawaan) | POST Role `Admin Toko` |
| 2 | `permission_manager.update` (Sales Invoice/Item/Customer) | POST Role Profile `Supervisor Toko` | `permission_manager.update` (doctype operasional) |
| 3 | POST User + `roles:[Kasir]` | POST User + `role_profiles:[Supervisor Toko]` | POST User + `roles:[Admin Toko]` |
| 4 | POST User Permission (Warehouse per toko) | POST User Permission (Cost Center) | — (atau User Permission sesuai kebutuhan) |
| 5 | — | — | (opsional) `permission_manager.update` utk override role bawaan |

> **Rekomendasi umum:**
> - Gunakan **role custom** untuk jabatan spesifik (Kasir, Supervisor, Admin) — lebih terkontrol
>   daripada mengubah role bawaan yang berefek global.
> - Gunakan **Role Profile** bila jabatan = kombinasi peran yang dipakai berulang.
> - Gunakan **User Permission** untuk membatasi lingkup data (per toko/cabang/company).
> - **User eksternal** (contact person supplier/customer) → `Website User` + role portal
>   `Supplier`/`Customer` + tautkan `Contact.user` (lihat §8.9).
> - Semua operasi di atas bersifat idempoten bila dijalankan dengan pre-check (GET sebelum POST)
>   dan mengikuti urutan; jalankan dari satu "service provisioning" ber-token System Manager.

---

### 8.9 Studi kasus D — Akses untuk contact person supplier/customer (Website User)

Kasus: memberi akun login kepada **contact person dari pihak eksternal** (mis. Markus, contact
person PT Maju Jaya). Ini user **eksternal** → `Website User`. ERPNext menyediakan role bawaan
`Supplier` / `Customer` yang ber-`desk_access=0` (`erpnext/setup/install.py` → `update_roles()`),
sehingga `user_type` otomatis menjadi `Website User`.

**Langkah 1 — Buat User dengan role `Supplier`:**

```bash
curl -X POST https://site-anda.com/api/resource/User \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{
    "email": "markus@majujaya.co.id",
    "first_name": "Markus",
    "roles": [ { "role": "Supplier" } ]
  }'
```

> Karena role `Supplier` ber-`desk_access=0`, `user_type` user ini otomatis `Website User`.

**Langkah 2 — Tautkan akun ke Contact-nya (field `user`, Link → User):**

```bash
curl -X PUT "https://site-anda.com/api/resource/Contact/Markus" \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{ "user": "markus@majujaya.co.id" }'
```

**Langkah 3 — (Opsional) Batasi lingkup data ke supplier-nya (User Permission):**

```bash
curl -X POST https://site-anda.com/api/resource/User%20Permission \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{
    "user": "markus@majujaya.co.id",
    "allow": "Supplier",
    "for_value": "PT Maju Jaya",
    "is_default": 1
  }'
```

**Menemukan Customer/Supplier milik user — jalur Contact:**

Untuk mengetahui user (ber-role `Customer`/`Supplier`) terhubung ke Customer/Supplier mana,
gunakan rantai **User → Contact (`Contact.user`) → `links[]` (Dynamic Link) → Customer/Supplier**.
Role sendiri tidak menyimpan info ini.

**Langkah 1 — Cari Contact milik user (filter field `user`):**

```bash
curl -G "https://site-anda.com/api/resource/Contact" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","full_name","user","links"]' \
  --data-urlencode 'filters=[["user","=","markus@majujaya.co.id"]]' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "Markus",
      "full_name": "Markus",
      "user": "markus@majujaya.co.id",
      "links": [
        { "link_doctype": "Supplier", "link_name": "PT Maju Jaya", "link_title": "PT Maju Jaya" }
      ]
    }
  ]
}
```

**Langkah 2 — Ekstrak Customer/Supplier dari `links[]`:** ambil `links[].link_name` yang
`link_doctype`-nya `Supplier` (atau `Customer`). Bila `data` kosong / tidak ada link tsb → user
belum terhubung ke Customer/Supplier mana pun via jalur ini.

**Langkah 3 — (Opsional) Ambil detail master:**

```bash
curl -X GET "https://site-anda.com/api/resource/Supplier/PT%20Maju%20Jaya" \
  -H 'Authorization: Bearer <access_token>'
```

> **Prasyarat jalur Contact:** `Contact.user` harus terisi — otomatis hanya bila email User =
> email **primer** Contact, atau di-set manual (`PUT Contact { "user": ... }`). Tanpa itu, rantai
> ini tidak menemukan apa pun.

> **Catatan:** `user_type` tidak menentukan hak ubah data — akses data ditentukan **roles** (+ User
> Permission). `Website User` dengan role ber-`write` tetap bisa mengubah data lewat API frontend.
> Pihak eksternal baru perlu `System User` bila ia juga karyawan internal (buat sebagai user
> internal terpisah).
