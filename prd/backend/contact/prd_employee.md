# PRD — REST API Doctype Employee (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Employee** di ERPNext (Frappe),
> diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** HR (ERPNext)
- **Doctype:** `Employee`
- **Versi API:** `/api/resource/...` (API v1)
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

---

Dokumentasi REST API untuk doctype **Employee**, mengikuti pola yang sama dengan
[Supplier](./prd_supplier.md), [Customer](./prd_customer.md), [Contact](./prd_contact.md),
[Address](./prd_address.md), dan [Lead](./prd_lead.md). Autentikasi (OAuth 2.0) dan penanganan
error umum berlaku sama (lihat [prd_supplier.md §3](./prd_supplier.md) /
[`prd_oauth.md`](../prd_oauth.md) dan [prd_supplier.md §6](./prd_supplier.md)).

> **Peran Employee:** Employee adalah **master data karyawan** (data kepegawaian + data pribadi).
> Berbeda dari Customer/Supplier, alamat/email/telepon karyawan disimpan **inline di satu dokumen
> Employee** (keputusan desain) — **tidak** memakai record Address/Contact ter-link. Relasi ke
> akun login direpresentasikan oleh field **`user_id`** (Link → User).

### Ruang lingkup — Employee

| # | Endpoint | Metode | Keterangan |
|---|---|---|---|
| 1 | `/api/resource/Employee` | POST | Buat Employee baru (CREATE) |
| 2 | `/api/resource/Employee/{name}` | GET | Ambil detail 1 Employee (READ) |
| 3 | `/api/resource/Employee` | GET | Daftar Employee (READ list) |
| 4 | `/api/resource/Employee/{name}` | PUT | Ubah Employee (UPDATE) |
| 5 | `/api/resource/Employee/{name}` | PUT | Non-aktifkan (`status=Inactive`/`Left`) — pengganti DELETE |
| 6 | `/api/method/erpnext.setup.doctype.employee.employee.create_user` | POST | Buat User dari Employee (§2.6) |
| 7 | `/api/resource/User` | GET | Daftar User (dropdown `user_id`, §3.2) |
| 8 | `/api/resource/Company` | GET | Daftar Company (dropdown `company`, §3.1) |
| 9 | `/api/method/frappe.core.doctype.data_export.exporter.export_data` | GET | Export Employee ke Excel/CSV (§2.7) |
| 10 | `/api/resource/Data Import` | POST | Import masal Employee (§2.8) |
| 11 | `/api/resource/Sales Person` | POST | Buat Sales Person dari Employee (§2.9) |
| 12 | `/api/resource/Sales Person` | GET | Cek/daftar Sales Person — dropdown & cek status (§2.9, §3.4) |
| 13 | `/api/method/frappe.rename_doc` | POST | Rename Sales Person — sinkron nama (opsional, §2.10) |
| 14 | `/api/method/erpnext.setup.doctype.employee.employee.deactivate_sales_person` | POST | Non-aktifkan Sales Person saat `status="Left"` (helper §2.10) |
| 15 | `/api/method/frappe.client.get_count` | GET | Total record Employee sesuai filter — untuk pagination / lazy loading (§2.2) |

---

## 1. Ringkasan field & data wajib — Employee

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `first_name` | Data | Nama depan. `employee_name` (Full Name) dihitung dari `first/middle/last_name`. |
| 🔴 **WAJIB** | `company` | Link → Company | **Perusahaan (badan hukum) tempat karyawan bekerja.** Harus berupa Company yang sudah ada (buat Company dulu jika belum — Setup Wizard membuatkan otomatis). |
| 🔴 **WAJIB** | `status` | Select | `reqd: 1`, default `Active`. CREATE: wajib kirim `"Active"`. Opsi: `Active`, `Inactive`, `Suspended`, `Left`. |
| 🔴 **WAJIB** | `gender` | Link → Gender | Nilai dari doctype `Gender` (mis. `Male`, `Female`). |
| 🔴 **WAJIB** | `date_of_birth` | Date | Tanggal lahir. |
| 🔴 **WAJIB** | `date_of_joining` | Date | Tanggal mulai bekerja. |
| 🔴 **WAJIB (bersyarat)** | `relieving_date` | Date | **Hanya wajib** bila `status == "Left"` (`mandatory_depends_on`). |
| 🟢 Opsional | `company_email`, `personal_email` | Data (Email) | Email perusahaan & pribadi. |
| 🟢 Opsional | `prefered_contact_email` | Select | `Company Email` / `Personal Email` / `User ID` — menentukan sumber email utama. |
| 🟢 Opsional | `cell_number` | Data (Phone) | Nomor HP. |
| 🟢 Opsional | `emergency_phone_number`, `person_to_be_contacted`, `relation` | — | Kontak darurat. |
| 🟢 Opsional | `current_address`, `permanent_address` | Small Text | Alamat (teks bebas, inline di Employee). |
| 🟢 Opsional | `current_accommodation_type`, `permanent_accommodation_type` | Select | `Rented` / `Owned`. |
| 🟢 Opsional | `department`, `designation`, `branch`, `reports_to`, `holiday_list`, `salary_currency`, `salutation` | Link | Penghubung ke master lain (lihat §1.1 poin 8). |
| 🟢 Opsional | `user_id` | Link → User | Akun login karyawan (lihat §1.1 poin 4–6). |
| 🟢 Opsional | `create_user_automatically`, `create_user_permission` | Check | Auto-create User / buat User Permission (lihat §2.1 Varian B & §2.6). |
| 🟢 Opsional | `employee_number`, `attendance_device_id`, `passport_number`, `bank_ac_no`, `ctc`, `salary_mode`, dll. | — | Data tambahan sesuai kebutuhan. |
| ⚪ **Otomatis — jangan dikirim** | `name`, `naming_series` | — | `name` dibangkitkan *naming series* `HR-EMP-` → mis. `HR-EMP-00001`. |
| ⚪ **Read-only — jangan dikirim** | `employee_name`, `employee`, `prefered_email` | — | `employee_name` (Full Name) dihitung dari nama; `employee` = `name`; `prefered_email` dihitung dari `prefered_contact_email`. |

### 1.1 Catatan penting — Employee

1. **`name` vs `employee_name` vs `first_name`** — `name` (kunci dokumen) = `HR-EMP-xxxxx`.
   `employee_name` (Full Name) **read-only**, dihitung dari `first + middle + last_name`. Jangan
   dikirim.
2. **`status` + `relieving_date` (keputusan, pola [Contact §1.1](./prd_contact.md)):**
   - **CREATE:** wajib kirim `status: "Active"`.
   - **UPDATE:** opsional. Tidak dikirim → status lama tetap. Dikirim → wajib salah satu opsi dan
     tidak boleh kosong (`""`). Nilai di luar opsi ditolak backend (HTTP 417 `ValidationError`);
     nilai kosong **tidak** ditolak backend (field punya default) → validasi kosong wajib di
     frontend.
   - **`status = "Left"`** → field `relieving_date` **wajib** diisi.
3. **Alamat/email/telepon inline (satu dokumen — keputusan desain):** semua data kontak disimpan
   sebagai field Employee itu sendiri (`company_email`, `personal_email`, `cell_number`,
   `current_address`, `permanent_address`, dll). **Tidak** memakai doctype Address/Contact
   ter-link → tidak ada data ganda, dan GET Employee langsung mengembalikan semuanya.
   `prefered_email` **read-only** = nilai field yang dipilih di `prefered_contact_email`
   (`Company Email`/`Personal Email`/`User ID`). Jika field pilihan kosong, backend mengingatkan
   *"Please enter {field}"* (`validate_preferred_email`).
4. **`user_id` = Link → User (penghubung ke akun login):** nilai harus email **user yang terdaftar
   dan aktif** (`validate_for_enabled_user_id`). Saat Employee disimpan dengan `user_id`, sistem
   otomatis (`update_user()`):
   - menambahkan role **Employee** ke user tsb (bila belum ada),
   - menyalin `employee_name`, `date_of_birth`, `gender`, `image` ke dokumen User.
5. **Satu `user_id` hanya untuk satu Employee aktif** — `validate_duplicate_user_id()` menolak
   user yang sudah dipakai Employee aktif lain (`DuplicateEntryError`). Jika `user_id` dilepas
   (UPDATE mengosongkannya), role Employee di user otomatis dihapus (`validate_employee_role`).
6. **`create_user_automatically` (Check):** jika dicentang & `user_id` belum terisi, User dibuat
   otomatis saat save memakai `prefered_email`/`company_email`/`personal_email` — salah satu email
   **wajib** terisi, jika tidak error **"Company or Personal Email is mandatory when 'Create User
   Automatically' is enabled"**. Hanya berlaku saat CREATE (`set_only_once`).
7. **Non-aktif vs hapus:** Employee **tidak punya field `disabled`** — non-aktif dilakukan lewat
   `status` (`Inactive` untuk berhenti sementara, `Left` untuk keluar). ⚠️ Hindari DELETE:
   saat DELETE, `on_trash` menghapus event terkait (`delete_events`) dan tidak ada penanda arsip.
8. **Field Link lain (penghubung ke master):** `company` (→ Company, wajib), `department`
   (→ Department), `designation` (→ Designation), `branch` (→ Branch), `reports_to` (→ Employee,
   self — membentuk hierarki/org chart karena Employee adalah **tree/NestedSet**), `holiday_list`
   (→ Holiday List), `salary_currency` (→ Currency), `salutation` (→ Salutation).
9. **Role yang dibutuhkan:** baca & tulis: `HR User`, `HR Manager`; `Employee` (baca dirinya
   sendiri); buat/hapus/export/import: `HR Manager`; `System Manager`.
10. **`company` harus ada lebih dulu** — karena `company` adalah Link (bukan teks bebas), POST
    gagal bila Company belum dibuat (error link validation). Pre-check wajib (lihat §2.1).

---

## 2. CRUD — Doctype Employee

### 2.1 CREATE — `POST /api/resource/Employee`

**Langkah 0 — Pre-check (wajib sebelum POST)**

a) **Company sudah ada** (company adalah Link, wajib record Company yang valid):

```bash
curl -G "https://site-anda.com/api/resource/Company" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","company_name"]' \
  --data-urlencode 'filters=[["name","=","PT Makan Enak"]]' \
  --data-urlencode 'limit_page_length=1'
```

`data` kosong → **blokir POST**, arahkan user membuat Company dulu (Setup Wizard membuatkannya
otomatis saat instalasi).

b) **`user_id` belum dipakai Employee aktif** (hanya bila `user_id` dikirim):

```bash
curl -G "https://site-anda.com/api/resource/Employee" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","employee_name","user_id","status"]' \
  --data-urlencode 'filters=[["user_id","=","budi@makanenak.co.id"],["status","=","Active"]]' \
  --data-urlencode 'limit_page_length=1'
```

`data` terisi → **blokir POST**, tampilkan pesan, mis. *"User sudah dipakai Employee lain."*

c) **Bila `create_user_automatically` dicentang:** pastikan salah satu email
(`prefered_email`/`company_email`/`personal_email`) terisi.

**Payload minimum (data wajib):**

```json
{
  "first_name": "Budi",
  "company": "PT Makan Enak",
  "status": "Active",
  "gender": "Male",
  "date_of_birth": "1990-05-08",
  "date_of_joining": "2026-08-01"
}
```

**Contoh request (lengkap — kontak inline + `user_id`):**

```bash
curl -X POST https://site-anda.com/api/resource/Employee \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "first_name": "Budi",
    "last_name": "Santoso",
    "company": "PT Makan Enak",
    "status": "Active",
    "gender": "Male",
    "date_of_birth": "1990-05-08",
    "date_of_joining": "2026-08-01",
    "department": "Kitchen",
    "designation": "Koki",
    "branch": "Toko B - Bogor",
    "company_email": "budi@makanenak.co.id",
    "personal_email": "budi@example.com",
    "prefered_contact_email": "Company Email",
    "cell_number": "+6281234567890",
    "current_address": "Jl. Melati No. 5, Bogor",
    "permanent_address": "Jl. Melati No. 5, Bogor",
    "user_id": "budi@makanenak.co.id"
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "data": {
    "name": "HR-EMP-00001",
    "owner": "Administrator",
    "creation": "2026-08-19 09:00:00.000000",
    "modified": "2026-08-19 09:00:00.000000",
    "modified_by": "Administrator",
    "docstatus": 0,
    "idx": 0,
    "naming_series": "HR-EMP-",
    "first_name": "Budi",
    "last_name": "Santoso",
    "employee_name": "Budi Santoso",
    "employee": "HR-EMP-00001",
    "company": "PT Makan Enak",
    "status": "Active",
    "gender": "Male",
    "date_of_birth": "1990-05-08",
    "date_of_joining": "2026-08-01",
    "department": "Kitchen",
    "designation": "Koki",
    "branch": "Toko B - Bogor",
    "company_email": "budi@makanenak.co.id",
    "personal_email": "budi@example.com",
    "prefered_contact_email": "Company Email",
    "prefered_email": "budi@makanenak.co.id",
    "cell_number": "+6281234567890",
    "current_address": "Jl. Melati No. 5, Bogor",
    "permanent_address": "Jl. Melati No. 5, Bogor",
    "user_id": "budi@makanenak.co.id",
    "create_user_automatically": 0,
    "create_user_permission": 0
  }
}
```

> `name` = `HR-EMP-00001` → simpan; dipakai untuk GET/PUT berikutnya. `employee_name`,
> `employee`, dan `prefered_email` dihitung otomatis (read-only).

**Varian B — CREATE + auto-create User (opsional):**

Jika body CREATE berisi `"create_user_automatically": 1` (dan salah satu email terisi), sistem
otomatis membuat **User** dari data Employee saat insert:

```bash
curl -X POST https://site-anda.com/api/resource/Employee \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "first_name": "Budi",
    "last_name": "Santoso",
    "company": "PT Makan Enak",
    "status": "Active",
    "gender": "Male",
    "date_of_birth": "1990-05-08",
    "date_of_joining": "2026-08-01",
    "company_email": "budi@makanenak.co.id",
    "prefered_contact_email": "Company Email",
    "create_user_automatically": 1
  }'
```

**Apa yang terjadi:** `user_id` otomatis terisi = email utama, User dibuat dengan role
**Employee**, nama lengkap/DOB/gender/image disalin ke User. Cek hasil di GET Employee
(`user_id` terisi).

> **Postman:** request `6.1` & `6.1b` di folder `6. Employee` (lihat §4).

### 2.2 READ (satu record) — alur lengkap

**Langkah 1 — Ambil detail Employee:**

```bash
curl -X GET "https://site-anda.com/api/resource/Employee/HR-EMP-00001" \
  -H 'Authorization: Bearer <access_token>'
```

Respons `data` berisi seluruh field Employee (struktur seperti respons CREATE), termasuk
`employee_name`, `user_id`, `prefered_email`, dan **semua field kontak inline** (`company_email`,
`personal_email`, `cell_number`, `current_address`, `permanent_address`, dll).

> **Catatan:** karena alamat/email/telepon tersimpan di dokumen Employee itu sendiri, **tidak perlu**
> query tambahan `Dynamic Link` (berbeda dari Supplier/Customer yang memakai Address/Contact
> ter-link).

**Total record count — `GET /api/method/frappe.client.get_count`**

Untuk kebutuhan pagination / lazy loading (mis. menampilkan "900 dari 1000 Employee"), ambil
**total record yang cocok dengan filter** lewat method whitelisted `get_count`. Respons list
(`GET /api/resource/Employee`) **tidak** menyertakan total — hitung terpisah dengan endpoint ini.

```bash
curl -G "https://site-anda.com/api/method/frappe.client.get_count" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Employee' \
  --data-urlencode 'filters=[["status","=","Active"]]'
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
> `limit`. Nilai ini dipakai untuk menghitung total halaman saat lazy loading di §2.3.

### 2.3 READ (daftar) — `GET /api/resource/Employee`

Parameter pilihan: `fields`, `filters`, `order_by`, `limit_start`, `limit_page_length`.

> **Lazy loading / pagination:** `limit_page_length` = jumlah record per halaman,
> `limit_start` = index awal (offset). Contoh ambil **per 50 data**:

```bash
# Halaman 1 — record 1–50
curl -G "https://site-anda.com/api/resource/Employee" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","employee_name","company","status","department","designation","branch","user_id","company_email","cell_number"]' \
  --data-urlencode 'filters=[["status","=","Active"]]' \
  --data-urlencode 'order_by=employee_name asc' \
  --data-urlencode 'limit_start=0' \
  --data-urlencode 'limit_page_length=50'
```

> Halaman berikutnya naikkan `limit_start` kelipatan 50 (`50`, `100`, dst.) dengan
> `limit_page_length=50` tetap. Jumlah halaman = `total / 50`, dengan `total` diperoleh
> dari `get_count` (§2.2). `limit_page_length=0` = ambil **semua** record (tanpa LIMIT),
> seperti contoh di bawah ini.

```bash
curl -G "https://site-anda.com/api/resource/Employee" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","employee_name","company","status","department","designation","branch","user_id","company_email","cell_number"]' \
  --data-urlencode 'filters=[["status","=","Active"]]' \
  --data-urlencode 'order_by=employee_name asc' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "HR-EMP-00001",
      "employee_name": "Budi Santoso",
      "company": "PT Makan Enak",
      "status": "Active",
      "department": "Kitchen",
      "designation": "Koki",
      "branch": "Toko B - Bogor",
      "user_id": "budi@makanenak.co.id",
      "company_email": "budi@makanenak.co.id",
      "cell_number": "+6281234567890"
    }
  ]
}
```

> Filter yang umum dipakai: `status`, `company`, `branch`, `department`, `designation` (semuanya
> field biasa → filter langsung, bukan Dynamic Link).

### 2.4 UPDATE — `PUT /api/resource/Employee/{name}`

Kirim **hanya field yang diubah**.

**Aturan `status` saat UPDATE:** opsional. Tidak dikirim → status lama tetap. Dikirim → wajib salah
satu opsi (§1) dan tidak boleh kosong (`""`). Jika `"Left"` → `relieving_date` wajib.

```bash
curl -X PUT "https://site-anda.com/api/resource/Employee/HR-EMP-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "prefered_contact_email": "Personal Email",
    "cell_number": "+6281987654321"
  }'
```

**Contoh respons (HTTP 200):** objek `data` terbaru — `prefered_email` otomatis dihitung ulang
menjadi nilai `personal_email`.

> **Mengganti `user_id`:** PUT `user_id` ke user lain = serah-terima akun; role Employee
> berpindah/ditambahkan ke user baru. PUT `user_id: ""` (kosong) = melepas akun → role Employee
> di user lama otomatis dihapus. Mengganti `first_name`/`last_name` juga memperbarui
> `employee_name` (dihitung ulang).

> **Postman:** request `6.4` di folder `6. Employee` (lihat §4).

### 2.5 Non-aktifkan (disarankan) — `PUT /api/resource/Employee/{name}`

**Aturan:** data Employee **tidak boleh dihapus**, hanya dinon-aktifkan. Karena Employee **tidak
punya field `disabled`**, non-aktifkan dengan `status`:

**Berhenti sementara (tidak aktif):**

```bash
curl -X PUT "https://site-anda.com/api/resource/Employee/HR-EMP-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "status": "Inactive" }'
```

**Karyawan keluar (Left) — `relieving_date` wajib:**

```bash
curl -X PUT "https://site-anda.com/api/resource/Employee/HR-EMP-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "status": "Left", "relieving_date": "2026-08-31", "reason_for_leaving": "Mengundurkan diri" }'
```

**Contoh respons (HTTP 200):** objek `data` terbaru dengan `"status": "Inactive"` (atau `"Left"`).

> **Efek non-aktif:** Employee `Inactive`/`Left` tidak muncul pada daftar aktif / tidak dipakai
> transaksi HR baru. Mengaktifkan kembali: PUT `{ "status": "Active" }`.

> ⚠️ **Jangan gunakan `DELETE /api/resource/Employee/{name}`** — saat DELETE, `on_trash` menghapus
> event terkait Employee. Karena aturan "jangan hapus", endpoint DELETE tidak dipakai.

### 2.6 Buat User dari Employee — `POST /api/method/erpnext.setup.doctype.employee.employee.create_user`

Alternatif **on-demand** untuk membuat akun User dari Employee yang sudah ada (tanpa harus
`create_user_automatically` saat CREATE). Method ini *whitelisted* (POST).

```bash
curl -X POST "https://site-anda.com/api/method/erpnext.setup.doctype.employee.employee.create_user" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "employee": "HR-EMP-00001",
    "email": "budi@makanenak.co.id",
    "create_user_permission": 1
  }'
```

**Hasil & aturan:**
- Membuat **User** dengan role **Employee**; nama (first/middle/last), `gender`, `birth_date`,
  `phone`, `bio` disalin dari Employee.
- Field `user_id` Employee otomatis terisi = email tsb.
- Bila `create_user_permission = 1` → dibuat **User Permission** untuk `Employee` (dokumen ini)
  dan `Company` (perusahaan Employee) — user hanya melihat data miliknya.
- **Error bila:** email kosong (*"Email is required to create a user"*) atau Employee **sudah**
  punya `user_id` (*"Employee {name} already has a linked user"*).
- Perlu hak **write** pada Employee (`HR User` / `HR Manager`).

> **Postman:** request `6.6` di folder `6. Employee` (lihat §4).

### 2.7 Export ke Excel / CSV — `GET /api/method/frappe.core.doctype.data_export.exporter.export_data`

Pola sama dengan [Supplier §4.6](./prd_supplier.md). Endpoint whitelisted, tanpa kustomisasi.

**Export Excel (.xlsx):**

```bash
curl -G "https://site-anda.com/api/method/frappe.core.doctype.data_export.exporter.export_data" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Employee' \
  --data-urlencode 'file_type=Excel' \
  --data-urlencode 'with_data=1' \
  --data-urlencode 'select_columns={"Employee":["name","employee_name","first_name","last_name","company","status","department","designation","branch","user_id","company_email","personal_email","cell_number","current_address","permanent_address","date_of_joining","date_of_birth","gender"]}' \
  --data-urlencode 'all_doctypes=0' \
  -o employee.xlsx
```

**Export CSV:** ganti `file_type=CSV` (pola sama). **Contoh respons:** file biner (attachment).

**Parameter & catatan:** sama dengan [prd_supplier.md §4.6](./prd_supplier.md) (tabel parameter).
Khusus Employee:
- **Child table:** `education`, `external_work_history`, `internal_work_history` — ikut via
  `all_doctypes=1` (default).
- **Izin export:** role harus punya hak **Export** pada doctype Employee (default: `HR Manager`).
- **Postman:** request `6.10` di folder `6. Employee` (lihat §4).

### 2.8 Import masal (bulk) — CSV / XLSX

Alur sama dengan [Supplier §4.7](./prd_supplier.md): upload file → buat `Data Import` →
`form_start_import`.

**Payload Data Import (Langkah 2):**

```bash
curl -X POST https://site-anda.com/api/resource/Data%20Import \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "reference_doctype": "Employee",
    "import_type": "Insert New Records",
    "import_file": "/private/files/employee_import.csv",
    "mute_emails": 1,
    "status": "Pending"
  }'
```

> Detail upload (Langkah 1) & start/pantau import (Langkah 3): lihat **[prd_supplier.md §4.7](./prd_supplier.md)**.

**Catatan khusus Employee:**
- **`allow_import`:** aktif (`allow_import = 1`) — import masal didukung.
- **Kolom wajib pada Insert:** `first_name`, `company`, `gender`, `date_of_birth`,
  `date_of_joining`, `status` (jangan kosong).
- **`user_id`:** jika di-import, pastikan belum dipakai Employee aktif lain (pre-check / koreksi
  log error).
- **Izin:** role harus punya hak **Import** (default: `HR Manager`).
- **Postman:** request `6.11` di folder `6. Employee` (lihat §4).

### 2.9 Jadikan Employee sebagai Sales Person

**Konsep:** `Sales Person` adalah **dokumen terpisah** (master tree) yang di-*link* ke Employee via
field `employee` (Link → Sales Person: `employee` → Employee). Doctype Employee **tidak** punya
field `sales_person` — jadi "karyawan ini sales person?" ditentukan dari sisi Sales Person. Alur
frontend = **2 panggilan API berurutan**: Employee dulu, lalu Sales Person.

**UI yang disarankan:** toggle **"Jadikan Sales Person"** di halaman Employee. Saat aktif →
tampilkan `commission_rate` (opsional) & pilihan `parent_sales_person` (dropdown grup, §3.4).
Setelah Employee tersimpan, frontend memanggil langkah 2.

**Langkah 1 — Buat Employee** (pola §2.1). Simpan `name`, mis. `HR-EMP-00001`.

**Langkah 2 — Buat Sales Person ter-link:**

```bash
curl -X POST https://site-anda.com/api/resource/Sales%20Person \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "sales_person_name": "Budi Santoso",
    "parent_sales_person": "All Sales Persons",
    "is_group": 0,
    "enabled": 1,
    "employee": "HR-EMP-00001",
    "commission_rate": 2.5
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "data": {
    "name": "Budi Santoso",
    "owner": "Administrator",
    "creation": "2026-08-19 10:00:00.000000",
    "modified": "2026-08-19 10:00:00.000000",
    "modified_by": "Administrator",
    "docstatus": 0,
    "idx": 0,
    "sales_person_name": "Budi Santoso",
    "parent_sales_person": "All Sales Persons",
    "is_group": 0,
    "enabled": 1,
    "employee": "HR-EMP-00001",
    "commission_rate": 2.5,
    "department": "Kitchen"
  }
}
```

> `name` = `Budi Santoso` (= `sales_person_name`, `autoname: field:sales_person_name`). `department`
> read-only, di-fetch otomatis dari `employee.department`.

**Aturan & catatan:**
- **Field wajib Sales Person:** `sales_person_name` (reqd + unique), `is_group` (reqd, default 0).
  `parent_sales_person` wajib menunjuk node **grup** (root `All Sales Persons` sudah ada otomatis).
- **`sales_person_name` unik** → dua karyawan bernama sama = `DuplicateEntryError`. Gunakan
  penamaan unik bila perlu (mis. `"Budi Santoso (B)"`).
- **Role:** membuat Employee butuh `HR User`/`HR Manager`; membuat Sales Person butuh
  **`Sales Master Manager`** — user frontend harus punya kedua role (atau `System Manager`).
- **Cek status sales person:** `GET /api/resource/Sales Person?filters=[["employee","=","HR-EMP-00001"]]&limit_page_length=1` → `data` kosong = bukan sales person; terisi = sudah.
- **Non-aktif konsisten:** saat Employee `status=Inactive`/`Left`, non-aktifkan juga Sales Person-nya
  (`PUT` Sales Person → `enabled: 0`).
- **Efek:** karyawan bisa dipilih pada `sales_team` di Sales Order / POS Invoice untuk atribusi
  penjualan & komisi per toko (lihat §3.4 & §4).

> **Postman:** request `6.12` (Buat Sales Person) & `6.13` (Cek Sales Person milik Employee) di
> folder `6. Employee` (lihat §4).

### 2.10 Sinkronisasi UPDATE Employee → Sales Person

Saat Employee di-UPDATE dan ia terhubung ke Sales Person (`employee` terisi), frontend wajib
menyinkronkan **2 hal** (berurutan): **status** (disarankan) dan **nama** (opsional). Keduanya
menjaga `name` dan `sales_person_name` tetap **sama** sehingga tidak ambigu bagi pengguna aplikasi
(penjualan vs autocomplete).

**a) Sinkron status (`enabled`) — disarankan:**

| Status Employee | Sales Person (`enabled`) |
|---|---|
| `Active` | `1` |
| `Inactive` / `Left` | `0` |

```bash
# (1) UPDATE Employee — mis. non-aktifkan
curl -X PUT "https://site-anda.com/api/resource/Employee/HR-EMP-00001" \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "status": "Inactive" }'

# (2) PUT Sales Person — ikut non-aktif (2 PUT berurutan)
curl -X PUT "https://site-anda.com/api/resource/Sales%20Person/Budi%20Santoso" \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "enabled": 0 }'
```

> `name` Sales Person berisi spasi → URL-encode (`Budi%20Santoso`). Aktifkan kembali: `enabled: 1`.

**b) Sinkron nama (`sales_person_name` = `employee_name`) — opsional:**

Perubahan nama Employee (mis. Siti → Budi) **tidak** otomatis mengubah Sales Person. Jika ingin
nama ikut, **jangan hanya PUT `sales_person_name`** — itu membuat `name` ≠ `sales_person_name`
→ ambigu (penjualan menampilkan `name` lama, autocomplete mencari `sales_person_name` baru).
Cara yang benar: **rename dokumen** via `rename_doc` — Frappe **otomatis menyinkronkan
`sales_person_name`** saat rename dokumen ber-autoname `field:` (`update_autoname_field`):

```bash
# (1) UPDATE Employee (ganti nama) — mis. Siti → Budi
curl -X PUT "https://site-anda.com/api/resource/Employee/HR-EMP-00002" \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "first_name": "Budi" }'

# (2) Rename Sales Person — name & sales_person_name ikut jadi "Budi Santoso"
curl -X POST "https://site-anda.com/api/method/frappe.rename_doc" \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Sales Person", "old": "Siti Santoso", "new": "Budi Santoso" }'
```

**Hasil rename:** `name` = `Budi Santoso` **dan** `sales_person_name` = `Budi Santoso` (sinkron
otomatis) → di fasilitas penjualan **dan** autocomplete sama-sama tampil "Budi Santoso", tidak
ambigu. Semua referensi (`sales_team`) ikut diperbarui oleh `rename_doc`.

> **Aturan & catatan:**
> - **`name` == `sales_person_name` harus selalu dijaga.** Jika keduanya berbeda (mis. hanya PUT
>   `sales_person_name` tanpa rename), muncul ambiguitas. **Hindari** pola ini.
> - `rename_doc` gagal bila nama baru sudah dipakai Sales Person lain (`DuplicateEntryError`).
> - Rename mengubah label **histori** transaksi (referensi lama ikut menampilkan nama baru).
> - **Rekomendasi:** untuk bisnis FnB, cukup **sinkron status (a)** — identitas sales person
>   (nama) dipertahankan stabil, pelaporan penjualan tetap konsisten. Gunakan (b) hanya bila
>   benar-benar ingin nama baru muncul di mana-mana.
> - `department` Sales Person read-only (fetch dari employee) — ter-refresh saat Sales Person
>   di-save (PUT).
> - **Postman:** request `6.15` (sync enabled) & `6.16` (rename via rename_doc) di folder
>   `6. Employee` (lihat §4).

**c) Helper bawaan `deactivate_sales_person` (opsional, khusus `Left`):**

ERPNext menyediakan method *whitelisted* untuk menonaktifkan Sales Person ter-link saat karyawan
berstatus `Left` — tanpa perlu frontend mencari & PUT Sales Person secara manual:

```bash
curl -X POST "https://site-anda.com/api/method/erpnext.setup.doctype.employee.employee.deactivate_sales_person" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "status": "Left", "employee": "HR-EMP-00001" }'
```

**Perilaku & batasan:**
- Hanya bereaksi bila `status == "Left"` — nilai lain diabaikan (tidak ada efek).
- Jika Employee punya Sales Person ter-link, field `enabled` di-set `0` (non-aktif).
- **Tidak** menangani `Inactive`, dan **tidak** mengaktifkan kembali (`enabled=1`) saat `Active` —
  untuk kasus itu tetap pakai pola §2.10(a) (2 PUT), atau gabungkan: helper ini untuk `Left`,
  PUT manual untuk status lain.
- Butuh hak **write** pada Employee (`HR User` / `HR Manager`).

> **Postman:** request `6.17` di folder `6. Employee` (lihat §4).

---

## 3. GET pendukung UI — Employee

### 3.1 GET Company (dropdown `company`) — `GET /api/resource/Company`

`company` adalah **Link → Company** (nilai = `name` Company). Ambil daftar Company untuk dropdown:

```bash
curl -G "https://site-anda.com/api/resource/Company" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","company_name"]' \
  --data-urlencode 'order_by=company_name asc' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    { "name": "PT Makan Enak", "company_name": "PT Makan Enak" }
  ]
}
```

> Kirim `name` pada field `company`. Company dibuat sekali (Setup Wizard / manual) — Employee
> tinggal memilihnya.

### 3.2 GET User (dropdown `user_id`) — `GET /api/resource/User`

`user_id` adalah **Link → User** (nilai = email user). Ambil daftar user aktif untuk dropdown:

```bash
curl -G "https://site-anda.com/api/resource/User" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","full_name","email"]' \
  --data-urlencode 'filters=[["enabled","=",1]]' \
  --data-urlencode 'order_by=full_name asc' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    { "name": "budi@makanenak.co.id", "full_name": "Budi Santoso", "email": "budi@makanenak.co.id" },
    { "name": "admin@perusahaan.co.id", "full_name": "Administrator", "email": "admin@perusahaan.co.id" }
  ]
}
```

> Kirim `name` (email user) pada field `user_id`. *(Alternatif: lewat alur §2.6 / Varian B §2.1,
> user dibuat otomatis dari Employee — dropdown ini dipakai bila user-nya sudah ada lebih dulu.)*

### 3.3 GET Department / Designation / Branch (dropdown)

Mengikuti pola §3.1 — ambil `name` dari masing-masing doctype untuk dropdown:

```bash
curl -G "https://site-anda.com/api/resource/Department" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","department_name"]' \
  --data-urlencode 'filters=[["is_group","=",0]]' \
  --data-urlencode 'limit_page_length=0'

curl -G "https://site-anda.com/api/resource/Designation" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","designation_name"]' \
  --data-urlencode 'limit_page_length=0'

curl -G "https://site-anda.com/api/resource/Branch" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","branch"]' \
  --data-urlencode 'limit_page_length=0'
```

> Kirim `name` masing-masing pada `department` / `designation` / `branch`. Dropdown lain yang
> serupa (`Gender`, `Salutation`, `Holiday List`, `Currency`, `reports_to` dari daftar Employee
> aktif) mengikuti pola yang sama; dapat ditambahkan belakangan bila dipakai.

### 3.4 GET Sales Person (dropdown `parent_sales_person` & cek status) — `GET /api/resource/Sales Person`

`parent_sales_person` adalah **Link → Sales Person** (tree master). Ambil node grup untuk dropdown parent:

```bash
curl -G "https://site-anda.com/api/resource/Sales%20Person" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","sales_person_name","is_group"]' \
  --data-urlencode 'filters=[["is_group","=",1]]' \
  --data-urlencode 'order_by=sales_person_name asc' \
  --data-urlencode 'limit_page_length=0'
```

**Cek "apakah Employee sudah jadi sales person"** (mode edit form Employee):

```bash
curl -G "https://site-anda.com/api/resource/Sales%20Person" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","sales_person_name","enabled","commission_rate","parent_sales_person"]' \
  --data-urlencode 'filters=[["employee","=","HR-EMP-00001"]]' \
  --data-urlencode 'limit_page_length=1'
```

`data` kosong (`[]`) → belum jadi sales person; terisi → tampilkan toggle aktif beserta
`commission_rate` / `parent_sales_person`.

### 3.5 Filter employee yang terdaftar sebagai sales person

Employee **tidak** menyimpan penanda sales person (relasi satu arah `Sales Person.employee` →
Employee), jadi filter memakai **2 GET API** lalu di-*join* di frontend. Dua pola:

**Pola A — daftar employee yang menjadi sales person (filter khusus):**

**Langkah 1 — Ambil semua Sales Person yang ter-link ke Employee:**

```bash
curl -G "https://site-anda.com/api/resource/Sales%20Person" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","sales_person_name","employee","enabled"]' \
  --data-urlencode 'filters=[["employee","is","set"]]' \
  --data-urlencode 'limit_page_length=0'
```

`data` → kumpulkan daftar `employee` (nama Employee), mis. `["HR-EMP-00001","HR-EMP-00003"]`.

**Langkah 2 — Ambil Employee yang termasuk daftar itu:**

```bash
curl -G "https://site-anda.com/api/resource/Employee" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","employee_name","status","branch","department","designation"]' \
  --data-urlencode 'filters=[["name","in",["HR-EMP-00001","HR-EMP-00003"]]]' \
  --data-urlencode 'limit_page_length=0'
```

Hasil = daftar Employee yang terdaftar sebagai sales person (lengkap dengan `enabled` bila
digabung dari langkah 1).

**Pola B — list semua employee + kolom penanda "Sales Person?":**

**Langkah 1** — ambil semua Employee (pola §2.3). **Langkah 2** — ambil semua Sales Person
ber-`employee` (query langkah 1 Pola A), lalu join di frontend:

```js
const spByEmployee = {};
salesPersons.forEach(sp => { if (sp.employee) spByEmployee[sp.employee] = sp; });

employees.forEach(emp => {
  const sp = spByEmployee[emp.name];
  emp.isSalesPerson = !!sp;
  emp.salesPersonName = sp?.sales_person_name;
  emp.salesPersonEnabled = sp?.enabled;
});
```

> Gunakan **Pola A** untuk halaman/filter "hanya sales person"; **Pola B** untuk halaman daftar
> Employee umum dengan kolom penanda. Keduanya cukup 2 GET.

> **Postman:** request `6.14` (GET Sales Person by employee) & `6.3` (GET Employee list) di
> folder `6. Employee` (lihat §4).

---

## 4. Catatan tambahan — Employee

- **Autentikasi & token:** sama — lihat [prd_supplier.md §3](./prd_supplier.md) dan [`prd_oauth.md`](../prd_oauth.md).
- **Error umum:** sama — lihat [prd_supplier.md §6](./prd_supplier.md). Tambahan khusus Employee:
  - `MandatoryError` (417) untuk `first_name`/`company`/`status`/`gender`/`date_of_birth`/
    `date_of_joining` (dan `relieving_date` saat `Left`).
  - `DuplicateEntryError` untuk `user_id` yang sudah dipakai Employee aktif lain.
  - Link validation untuk `company`/`user_id`/`department`/`designation`/dst. bila nilai belum ada
    (mis. Company belum dibuat).
- **Role:** baca & tulis `HR User` / `HR Manager`; `Employee` (baca dirinya sendiri); export/import
  `HR Manager`; `System Manager`. **Untuk alur §2.9 (jadikan Sales Person),** user frontend perlu
  role tambahan **`Sales Master Manager`** (create/write Sales Person). Aturan sinkronisasi
  Employee ↔ Sales Person (status & nama) ada di **§2.10**; filter employee-sales person di **§3.5**.
- **Helper opsional (bonus, whitelisted):** `erpnext.setup.doctype.employee.employee.get_contact_details`
  (POST, arg `employee`) mengembalikan email dengan prioritas Preferred → Company → Personal →
  User ID — berguna bila UI butuh satu email utama per karyawan.
- **Helper non-aktif Sales Person:** `erpnext.setup.doctype.employee.employee.deactivate_sales_person`
  (POST, arg `status` & `employee`) — set `enabled=0` pada Sales Person ter-link saat
  `status="Left"` (lihat §2.10).
- **Koleksi Postman:** contoh Employee ada di folder **`6. Employee`** pada
  `docs/postman/postman_erpnext_api.json`.
