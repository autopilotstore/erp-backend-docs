# PRD — REST API Doctype Lead (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Lead** di ERPNext (Frappe),
> diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** CRM (ERPNext)
- **Doctype:** `Lead`
- **Versi API:** `/api/resource/...` (API v1)
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

---

Dokumentasi REST API untuk doctype **Lead**, mengikuti pola yang sama dengan
[Supplier](./prd_supplier.md), [Customer](./prd_customer.md), [Contact](./prd_contact.md), dan
[Address](./prd_address.md). Autentikasi (OAuth 2.0) dan penanganan error umum berlaku sama
(lihat [prd_supplier.md §3](./prd_supplier.md) / [`prd_oauth.md`](../prd_oauth.md) dan
[prd_supplier.md §6](./prd_supplier.md)).

> **Peran Lead:** Lead adalah **calon customer** (prospek) di modul CRM — tahap sebelum
> Opportunity/Quotation. Di stock ERPNext, Lead **tidak** menyimpan relasi ke `Sales Person`
> (atribusi penjualan diakomodir di dokumen hilir: Customer/Quotation via child table
> `sales_team`). Penanggung jawab lead direpresentasikan oleh field **`lead_owner`** (Link → User).

### Ruang lingkup — Lead

| # | Endpoint | Metode | Keterangan |
|---|---|---|---|
| 1 | `/api/resource/Lead` | POST | Buat Lead baru (CREATE) |
| 2 | `/api/resource/Lead/{name}` | GET | Ambil detail 1 Lead (READ) |
| 3 | `/api/resource/Lead` | GET | Daftar Lead (READ list) |
| 4 | `/api/resource/Lead/{name}` | PUT | Ubah Lead (UPDATE) |
| 5 | `/api/resource/Lead/{name}` | PUT | Non-aktifkan Lead (`disabled=1`) — pengganti DELETE |
| 6 | `/api/resource/User` | GET | Daftar User (dropdown `lead_owner`, §3.1) |
| 7 | `/api/method/frappe.core.doctype.data_export.exporter.export_data` | GET | Export Lead ke Excel/CSV (§2.6) |
| 8 | `/api/method/erpnext.crm.doctype.lead.mapper.make_customer` | POST | Konversi Lead → Customer (§2.8) |
| 9 | `/api/method/frappe.client.get_count` | GET | Total record Lead sesuai filter — untuk pagination / lazy loading (§2.2) |

---

## 1. Ringkasan field & data wajib — Lead

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB (min. 1)** | `first_name` *atau* `company_name` | Data | Nama orang (`first_name`, opsional `middle_name`/`last_name`) **atau** nama organisasi (`company_name`). `mandatory_depends_on` saling meniadakan; `email_id` juga diterima server sebagai sumber nama (fallback). Jika semua kosong → error **"A Lead requires either a person's name or an organization's name"**. |
| 🔴 **WAJIB (CREATE) / 🟢 Opsional (UPDATE)** | `status` | Select | `reqd: 1`, default `Lead`. CREATE: wajib kirim `"Open"`. UPDATE: opsional — jika dikirim harus salah satu opsi & tidak boleh kosong. Opsi: `Lead`, `Open`, `Replied`, `Opportunity`, `Quotation`, `Lost Quotation`, `Interested`, `Converted`, `Do Not Contact`. |
| 🟢 Opsional (default otomatis) | `lead_owner` | Link → User | Penanggung jawab lead. Default `__user` (user login). Boleh dikirim eksplisit saat CREATE/UPDATE (mis. assign ke sales rep). Contoh CRUD: §2.1 / §2.4. |
| 🟢 Opsional | `email_id`, `mobile_no`, `phone` | — | sesuai kebutuhan. Ketiganya ikut terbawa ke Customer saat konversi (lewat Contact ter-link — lihat §2.8). |
| ⚪ **Otomatis — jangan dikirim** | `name`, `naming_series` | — | `name` dibangkitkan oleh *naming series* `CRM-LEAD-.YYYY.-` → mis. `CRM-LEAD-00001`. |
| ⚪ **Read-only — jangan dikirim** | `lead_name`, `title` | — | `lead_name` (Full Name) dihitung dari `first/middle/last_name` atau `company_name`/`email_id`; `title` = `company_name or lead_name`. |

### 1.1 Catatan penting — Lead

1. **`name` vs `lead_name` vs `first_name`** — `name` (kunci dokumen) = `CRM-LEAD-xxxxx`.
   `lead_name` (Full Name) **read-only** dan dihitung otomatis: dari `salutation + first/middle/
   last_name` bila ada, jika tidak dari `company_name`, jika tidak dari bagian lokal `email_id`.
   Jangan dikirim.
2. **Minimal satu sumber nama** — server menolak bila `lead_name`/`company_name`/`email_id` semua
   kosong. Untuk UI: wajib isi `first_name` (nama orang) **atau** `company_name` (organisasi).
3. **`email_id` unik & valid** — `check_email_id_is_unique()` menolak email yang sudah dipakai Lead
   lain (`DuplicateEntryError`); `validate_email_id()` memastikan format valid dan `lead_owner`
   **tidak boleh sama** dengan `email_id` (*"Lead Owner cannot be same as the Lead Email Address"*).
   → UI sebaiknya **pre-check email** sebelum CREATE (lihat §2.1 Langkah 0).
4. **`lead_owner` = User, bukan Sales Person** — nilai harus user yang terdaftar (default user
   login). Untuk atribusi *Sales Person*, gunakan child table `sales_team` di dokumen hilir
   (Customer/Quotation/Sales Order) — Lead tidak punya field sales person bawaan.
5. **Aturan `status` (keputusan, pola [Contact §1.1](./prd_contact.md)):**
   - **CREATE:** wajib kirim `status: "Open"`.
   - **UPDATE:** opsional. Tidak dikirim → status lama tetap. Dikirim → wajib salah satu opsi dan
     tidak boleh kosong (`""`). Nilai di luar opsi ditolak backend (HTTP 417 `ValidationError`);
     nilai kosong **tidak** ditolak backend (field punya default), jadi validasi kosong wajib di
     frontend.
6. **Non-aktif vs hapus** — Lead punya field `disabled` (Check). Gunakan `disabled=1` untuk
   non-aktif. ⚠️ Hindari DELETE: saat DELETE, Contact & Address yang ter-link ikut terhapus
   (`delete_contact_and_address`) dan `Issue` yang merujuk Lead ikut dilepas.
7. **Auto-create Contact** — bila *CRM Settings → Auto Creation of Contact* aktif, membuat Lead
   otomatis membuat **Contact** terkait (`before_insert`). Data minimal (nama/company) sudah cukup;
   email/telepon ikut bila terisi.
8. **Role yang dibutuhkan** — baca & tulis: `Sales User`; buat/hapus/export/import: `Sales Manager`;
   `System Manager`.

---

## 2. CRUD — Doctype Lead

### 2.1 CREATE — `POST /api/resource/Lead`

**Langkah 0 — Pre-check email (wajib sebelum POST)**

Karena `email_id` harus unik di antara Lead, UI disarankan memeriksa sebelum POST agar bisa memberi
pesan ramah (backend tetap menolak dengan `DuplicateEntryError`):

```bash
curl -G "https://site-anda.com/api/resource/Lead" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","lead_name","email_id"]' \
  --data-urlencode 'filters=[["email_id","=","budi@example.com"]]' \
  --data-urlencode 'limit_page_length=1'
```

**Hasil & aturan:**
- `data` kosong (`[]`) → lanjut ke POST.
- `data` terisi → blokir POST, tampilkan pesan, mis. *"Email sudah dipakai Lead lain. Gunakan email lain."*

**Payload minimum (data wajib):**

```json
{
  "first_name": "Budi",
  "last_name": "Santoso",
  "status": "Open"
}
```

**Contoh request (dengan `lead_owner` — opsional):**

```bash
curl -X POST https://site-anda.com/api/resource/Lead \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "first_name": "Budi",
    "last_name": "Santoso",
    "company_name": "PT Budi Maju",
    "email_id": "budi@example.com",
    "mobile_no": "+6281234567890",
    "status": "Open",
    "lead_owner": "kiki@perusahaan.co.id"
  }'
```

> **`lead_owner`:** opsional (default user yang login = `__user`). Dikirim eksplisit di sini, mis.
> meng-assign lead ke user sales "Kiki". Nilai harus **email user yang terdaftar** dan **tidak
> boleh sama** dengan `email_id`.

**Contoh respons sukses (HTTP 200):**

```json
{
  "data": {
    "name": "CRM-LEAD-00001",
    "owner": "kiki@perusahaan.co.id",
    "creation": "2026-08-19 09:00:00.000000",
    "modified": "2026-08-19 09:00:00.000000",
    "modified_by": "kiki@perusahaan.co.id",
    "docstatus": 0,
    "idx": 0,
    "naming_series": "CRM-LEAD-.YYYY.-",
    "first_name": "Budi",
    "last_name": "Santoso",
    "company_name": "PT Budi Maju",
    "lead_name": "Budi Santoso",
    "title": "PT Budi Maju",
    "email_id": "budi@example.com",
    "mobile_no": "+6281234567890",
    "status": "Open",
    "lead_owner": "kiki@perusahaan.co.id",
    "disabled": 0
  }
}
```

> `name` = `CRM-LEAD-00001` → simpan; dipakai untuk GET/PUT berikutnya. `lead_name` dihitung
> otomatis; `title` = `company_name or lead_name`.

**Varian B — CREATE Lead + daftarkan Address & Contact (cara andal lewat API)**

Karena doctype `Lead` **tidak punya field alamat/contact** (`address_html`/`contact_html` hanya
tampilan read-only; tidak ada child table contact di Lead), Address & Contact dibuat sebagai record
tersendiri lalu **ditautkan ke Lead** lewat child table `links` (Dynamic Link). Pola ini mendukung
**multi address & multi contact** dalam satu Lead.

**Langkah 1 — Buat Lead** (payload minimum / contoh di atas). Simpan `name` hasilnya, mis.
`CRM-LEAD-00001`.

**Langkah 2 — Buat Address yang ter-link ke Lead:**

```bash
curl -X POST https://site-anda.com/api/resource/Address \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "address_title": "PT Budi Maju",
    "address_type": "Billing",
    "address_line1": "Jl. Sudirman No. 123",
    "city": "Jakarta",
    "country": "Indonesia",
    "is_primary_address": 1,
    "links": [
      { "link_doctype": "Lead", "link_name": "CRM-LEAD-00001" }
    ]
  }'
```

Field wajib Address: `address_type`, `address_line1`, `city`, `country`. `is_primary_address = 1`
menandai address primary (default saat konversi). Simpan `name` — autoname
`{address_title}-{address_type}` (mis. `PT Budi Maju-Billing`). Untuk **multi address**, ulangi
Langkah 2 dengan `address_type` lain (mis. `Shipping`) — cukup satu yang ber-flag primary.

**Langkah 3 — Buat Contact yang ter-link ke Lead:**

```bash
curl -X POST https://site-anda.com/api/resource/Contact \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "first_name": "Budi",
    "last_name": "Santoso",
    "status": "Open",
    "email_ids": [
      { "email_id": "budi@example.com", "is_primary": 1 }
    ],
    "phone_nos": [
      { "phone": "+6281234567890", "is_primary_mobile_no": 1 }
    ],
    "is_primary_contact": 1,
    "links": [
      { "link_doctype": "Lead", "link_name": "CRM-LEAD-00001" }
    ]
  }'
```

`is_primary_contact = 1` menandai contact utama. Untuk **multi contact**, ulangi dengan nama/email
lain — cukup satu yang ber-flag primary.

> **Mengapa tidak langsung di Lead?** Lead tidak menyimpan address/contact sebagai field-nya
> sendiri. `links` (Dynamic Link) menghubungkan record Address/Contact ke Lead — mekanisme yang
> sama dipakai [Supplier §4.1](./prd_supplier.md) / [Customer §2.1](./prd_customer.md) (Varian B). Saat Lead dikonversi ke Customer, semua
> Address/Contact ter-link ikut terbawa (lihat **§2.8**).

> **Postman:** request `5.1`–`5.3` di folder `5. Lead` (lihat §4).

### 2.2 READ (satu record) — alur lengkap

**Langkah 1 — Ambil detail Lead:**

```bash
curl -X GET "https://site-anda.com/api/resource/Lead/CRM-LEAD-00001" \
  -H 'Authorization: Bearer <access_token>'
```

Respons `data` berisi seluruh field Lead (struktur seperti respons CREATE), termasuk `lead_owner`,
`lead_name`, `title`, dan `email_id`/`mobile_no` bila terisi.

> **Catatan:** `data` **tidak** menyertakan daftar Contact/Address yang ter-link (daftar itu hanya
> diisi lewat `onload` pada form Desk). Ambil terpisah lewat filter Dynamic Link pada Contact /
> Address bila diperlukan (pola sama dengan [prd_supplier.md §4.2](./prd_supplier.md) langkah 2 & 3, `link_doctype = "Lead"`).

**Total record count — `GET /api/method/frappe.client.get_count`**

Untuk kebutuhan pagination / lazy loading (mis. menampilkan "900 dari 1000 Lead"), ambil
**total record yang cocok dengan filter** lewat method whitelisted `get_count`. Respons list
(`GET /api/resource/Lead`) **tidak** menyertakan total — hitung terpisah dengan endpoint ini.

```bash
curl -G "https://site-anda.com/api/method/frappe.client.get_count" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Lead' \
  --data-urlencode 'filters=[["status","=","Open"]]'
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

### 2.3 READ (daftar) — `GET /api/resource/Lead`

Parameter pilihan: `fields`, `filters`, `order_by`, `limit_start`, `limit_page_length`.

> **Lazy loading / pagination:** `limit_page_length` = jumlah record per halaman,
> `limit_start` = index awal (offset). Contoh ambil **per 50 data**:

```bash
# Halaman 1 — record 1–50
curl -G "https://site-anda.com/api/resource/Lead" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","lead_name","company_name","status","lead_owner","email_id","mobile_no","disabled"]' \
  --data-urlencode 'filters=[["status","=","Open"]]' \
  --data-urlencode 'limit_start=0' \
  --data-urlencode 'limit_page_length=50'
```

> Halaman berikutnya naikkan `limit_start` kelipatan 50 (`50`, `100`, dst.) dengan
> `limit_page_length=50` tetap. Jumlah halaman = `total / 50`, dengan `total` diperoleh
> dari `get_count` (§2.2). `limit_page_length=0` = ambil **semua** record (tanpa LIMIT),
> seperti contoh di bawah ini.

```bash
curl -G "https://site-anda.com/api/resource/Lead" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","lead_name","company_name","status","lead_owner","email_id","mobile_no","disabled"]' \
  --data-urlencode 'filters=[["status","=","Open"]]' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "CRM-LEAD-00001",
      "lead_name": "Budi Santoso",
      "company_name": "PT Budi Maju",
      "status": "Open",
      "lead_owner": "kiki@perusahaan.co.id",
      "email_id": "budi@example.com",
      "mobile_no": "+6281234567890",
      "disabled": 0
    }
  ]
}
```

### 2.4 UPDATE — `PUT /api/resource/Lead/{name}`

Kirim **hanya field yang diubah**.

**Aturan `status` saat UPDATE:** opsional. Tidak dikirim → status lama tetap. Dikirim → wajib salah
satu opsi (§1) dan tidak boleh kosong (`""`).

```bash
curl -X PUT "https://site-anda.com/api/resource/Lead/CRM-LEAD-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "status": "Replied",
    "lead_owner": "kiki@perusahaan.co.id"
  }'
```

**Contoh respons (HTTP 200):** objek `data` berisi dokumen terbaru (`status` = `Replied`,
`lead_owner` diperbarui).

> **`lead_owner` saat UPDATE:** boleh dikirim untuk **serah-terima** lead ke user lain (mis. dari
> user lama ke Kiki). Nilai harus user terdaftar dan tidak boleh sama dengan `email_id`. Mengganti
> `first_name` juga otomatis memperbarui `lead_name`/`title` (dihitung ulang).

**Menambahkan Address / Contact ke Lead yang sudah ada**

Lead **tidak punya field address/contact** — jadi untuk menambah alamat atau contact person ke Lead
yang sudah ada, buat record Address/Contact baru dengan `links` → Lead (pola Varian B §2.1),
bukan lewat PUT Lead. PUT Lead hanya untuk field Lead (nama, `status`, `lead_owner`, dst.).

```bash
# (a) UPDATE field Lead (opsional — kirim hanya yang diubah)
curl -X PUT "https://site-anda.com/api/resource/Lead/CRM-LEAD-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "status": "Replied" }'

# (b) Tambah Address baru ke Lead (multi address)
curl -X POST https://site-anda.com/api/resource/Address \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "address_title": "PT Budi Maju",
    "address_type": "Shipping",
    "address_line1": "Jl. Gatot Subroto No. 45",
    "city": "Jakarta",
    "country": "Indonesia",
    "links": [
      { "link_doctype": "Lead", "link_name": "CRM-LEAD-00001" }
    ]
  }'

# (c) Tambah Contact baru ke Lead (multi contact)
curl -X POST https://site-anda.com/api/resource/Contact \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "first_name": "Markus",
    "last_name": "Santoso",
    "status": "Open",
    "email_ids": [
      { "email_id": "markus@example.com", "is_primary": 1 }
    ],
    "links": [
      { "link_doctype": "Lead", "link_name": "CRM-LEAD-00001" }
    ]
  }'
```

> Menghapus address/contact = menghapus baris `links`-nya (PUT Address/Contact — `links` bersifat
> replace-all) atau non-aktifkan; jangan DELETE (lihat [prd_address.md §2.5](./prd_address.md) /
> [prd_contact.md §2.5](./prd_contact.md)).

> **Postman:** request `5.7` (UPDATE Lead) & `5.2`–`5.3` (tambah Address/Contact) di folder
> `5. Lead` (lihat §4).

### 2.5 Non-aktifkan (disarankan) — `PUT /api/resource/Lead/{name}`

**Aturan:** data Lead **tidak boleh dihapus**, hanya dinon-aktifkan. Non-aktifkan dengan field
`disabled`:

```bash
curl -X PUT "https://site-anda.com/api/resource/Lead/CRM-LEAD-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "disabled": 1 }'
```

**Contoh respons (HTTP 200):** objek `data` terbaru dengan `"disabled": 1`. Mengaktifkan kembali:
`PUT` `{ "disabled": 0 }`.

> **Efek non-aktif:** Lead `disabled=1` tidak muncul di daftar aktif. **Contact/Address ter-link
> TIDAK dihapus** (berbeda dari DELETE).

> ⚠️ **Jangan gunakan `DELETE /api/resource/Lead/{name}`** — saat DELETE, Contact & Address
> ter-link ikut terhapus dan `Issue` yang merujuk Lead dilepas. Karena aturan "jangan hapus",
> endpoint DELETE tidak dipakai.

### 2.6 Export ke Excel / CSV — `GET /api/method/frappe.core.doctype.data_export.exporter.export_data`

Pola sama dengan [Supplier §4.6](./prd_supplier.md). Endpoint whitelisted, tanpa kustomisasi.

**Export Excel (.xlsx):**

```bash
curl -G "https://site-anda.com/api/method/frappe.core.doctype.data_export.exporter.export_data" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Lead' \
  --data-urlencode 'file_type=Excel' \
  --data-urlencode 'with_data=1' \
  --data-urlencode 'select_columns={"Lead":["name","lead_name","company_name","first_name","last_name","email_id","mobile_no","status","lead_owner","disabled"]}' \
  --data-urlencode 'all_doctypes=0' \
  -o lead.xlsx
```

**Export CSV:** ganti `file_type=CSV` (pola sama). **Contoh respons:** file biner (attachment) —
`filename="Lead.xlsx"` / `Lead.csv`.

**Parameter & catatan:** sama dengan [prd_supplier.md §4.6](./prd_supplier.md) (tabel parameter). Khusus Lead:
- **Child table:** `notes` (CRM Note) — ikut via `all_doctypes=1` (default).
- **Izin export:** role harus punya hak **Export** pada doctype Lead (default: `Sales Manager`).
- **Alternatif template/import:** `download_template` ([§4.6](./prd_supplier.md)) — sama.
- **Postman:** request `5.12` di folder `5. Lead` (lihat §4).

### 2.7 Import masal (bulk) — CSV / XLSX

Alur sama dengan [Supplier §4.7](./prd_supplier.md): upload file → buat `Data Import` → `form_start_import`.

**Payload Data Import (Langkah 2):**

```bash
curl -X POST https://site-anda.com/api/resource/Data%20Import \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "reference_doctype": "Lead",
    "import_type": "Insert New Records",
    "import_file": "/private/files/lead_import.csv",
    "mute_emails": 1,
    "status": "Pending"
  }'
```

> Detail upload (Langkah 1) & start/pantau import (Langkah 3): lihat **[prd_supplier.md §4.7](./prd_supplier.md)**.

**Catatan khusus Lead:**
- **`allow_import`:** aktif (`allow_import = 1`) — import masal didukung.
- **Sumber nama:** kolom `first_name`/`company_name` tetap perlu terisi (min. 1) pada Insert.
- **`status`:** aturan nilai tetap berlaku (`Open`/`Lead`/dst.); jangan kosong pada Insert.
- **Izin:** role harus punya hak **Import** (default: `Sales Manager`).
- **Postman:** request `5.13` di folder `5. Lead` (lihat §4).

### 2.8 Konversi Lead → Customer

ERPNext menyediakan mapper native `make_customer` untuk mengubah Lead menjadi Customer. Alur yang
wajib dijalankan frontend: **panggil mapper → periksa/lengkapi data → `POST /api/resource/Customer`**
(mapper **tidak** menyimpan Customer; ia hanya menghasilkan draft terisi).

**Langkah 0 — Pre-check: Lead belum pernah dikonversi (wajib)**

Backend **tidak** mencegah konversi ulang: tidak ada *unique constraint* pada `Customer.lead_name`,
dan mapper `make_customer` tidak mengecek apakah Lead sudah punya Customer. Jika diabaikan, akan
tecipta **Customer duplikat** untuk Lead yang sama. Karena itu UI wajib mengecek sebelum konversi:

```bash
curl -G "https://site-anda.com/api/resource/Customer" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","customer_name"]' \
  --data-urlencode 'filters=[["lead_name","=","CRM-LEAD-00001"]]' \
  --data-urlencode 'limit_page_length=1'
```

**Hasil & aturan:**
- `data` kosong (`[]`) → Lead belum dikonversi → lanjut ke Langkah 1.
- `data` terisi → **blokir konversi**, tampilkan pesan, mis. *"Lead ini sudah dikonversi ke
  Customer {customer_name}. Tidak bisa dikonversi ulang."*

> **Penanda Lead yang sudah dikonversi:** `Lead.status == "Converted"` (di-set otomatis oleh
> `update_lead_status()`) **dan** adanya Customer dengan `lead_name` = nama Lead (relasi balik —
> paling definitif, sekaligus menunjukkan Customer tujuan). Cek `status` bisa berubah manual, jadi
> pre-check via `lead_name` lebih kuat.

> **Bila pre-check diabaikan:** mapper tetap sukses (draft tanpa error) dan `POST Customer` tetap
> membuat Customer kedua; Address/Contact milik Lead ikut ter-link ke Customer baru, dan komunikasi
> di-copy ulang bila *CRM Settings → Carry Forward Communication and Comments* aktif.

**Langkah 1 — Ambil data Customer hasil mapping (draft):**

```bash
curl -X POST "https://site-anda.com/api/method/erpnext.crm.doctype.lead.mapper.make_customer" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "source_name": "CRM-LEAD-00001" }'
```

Respons berisi objek Customer yang terisi otomatis:

| Field Customer | Diisi otomatis dari Lead |
|---|---|
| `customer_name` | `company_name`; jika kosong → `lead_name` |
| `customer_type` | `Company` (ada `company_name`) / `Individual` (tanpa) |
| `customer_group` | default *Customer Group* sistem |
| `customer_primary_address` | Address default milik Lead (ter-link) — lihat penjelasan di bawah |
| `customer_primary_contact` | Contact default milik Lead (ter-link) — lihat penjelasan di bawah |

> `customer_primary_address` / `customer_primary_contact` hanya terisi bila Lead **memang punya
> Address/Contact yang ter-link**. Jika tidak ada, field dibiarkan kosong (bukan error) dan bisa
> diisi belakangan pada Customer.

**Penjelasan `customer_primary_address` & `customer_primary_contact` (penting):**

Kedua field ini bukan sekadar "salin nama", melainkan hasil pembacaan **Address/Contact yang
ter-link ke Lead** lewat child table `links` (`link_doctype = "Lead"`, `link_name = <nama Lead>`).
Logika pengambilannya:

- **`customer_primary_address`** = Address yang **ter-link ke Lead** dengan `is_primary_address = 1`
  (dan tidak `disabled`). Jika tidak ada yang bertanda primary → fallback ke **Address ter-link
  pertama** yang tidak `disabled`. Jika tidak ada Address ter-link sama sekali → kosong.
- **`customer_primary_contact`** = Contact yang **ter-link ke Lead** dengan `is_primary_contact = 1`.
  Jika tidak ada yang bertanda primary → fallback ke **Contact ter-link pertama**. Jika tidak ada
  Contact ter-link → kosong.

**Multi Address & Contact pada 1 Lead:**

Lead **bukan** menyimpan address/contact sebagai field-nya sendiri (tidak ada `address_line1`,
`city`, child table contact di doctype Lead; `address_html`/`contact_html` hanya tampilan read-only).
Sebaliknya, Address & Contact dibuat sebagai record terpisah lalu **ditautkan ke Lead** via child
table `links` — mekanisme Dynamic Link yang **many-to-many** dan **tanpa batas jumlah**:

- **Banyak Address** dalam 1 Lead → masing-masing Address dibuat dengan
  `links: [{ "link_doctype": "Lead", "link_name": "CRM-LEAD-00001" }]`.
- **Banyak Contact** dalam 1 Lead → masing-masing Contact dibuat dengan `links` yang sama.
- Satu Address/Contact juga bisa ter-link ke banyak party sekaligus (mis. Lead + Customer).
- **Primary:** Address memakai `is_primary_address`, Contact memakai `is_primary_contact` —
  penanda yang menentukan default saat konversi (hanya satu yang bisa jadi primary per party;
  controller `validate_preferred_address()` me-reset flag lain).

**Carry-over otomatis saat konversi (semua ikut terbawa):**

1. **Primary → default Customer:** `customer_primary_address` / `customer_primary_contact` diisi
   dari Address/Contact default (primary) milik Lead.
2. **SEMUA address & contact ter-link ikut dibawa:** saat Customer dibuat dari Lead (`lead_name`
   terisi), `link_address_and_contact()` di customer.py mengambil **seluruh** Dynamic Link milik
   Lead (Address + Contact) lalu menambahkan baris `links` → Customer pada masing-masing. Jadi
   semua alamat & contact person milik Lead otomatis muncul di tab Customer — tanpa membuat ulang.
3. **Tidak dipindah & tidak duplikat:** Address/Contact tetap ter-link ke Lead dan **tambahan**
   ter-link ke Customer; auto-create contact Customer di-skip karena `lead_name` sudah terisi
   (`create_primary_contact()` tidak dijalankan).

Contoh: Lead Budi punya 2 address (Billing + Shipping) dan 2 contact (Budi + Markus). Setelah
konversi → `customer_primary_address` = Billing, `customer_primary_contact` = Budi, tab Address
Customer = Billing + Shipping, tab Contact Customer = Budi + Markus.

**Langkah 2 — Lengkapi data yang tidak terbawa, lalu simpan:**

Field yang **tidak** ikut terbawa otomatis: `territory`, `default_price_list`, `payment_terms`,
`credit_limits`, child table `accounts`, dan **`sales_team`** (tempat mencatat sales person).

```bash
curl -X POST https://site-anda.com/api/resource/Customer \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "customer_name": "PT Budi Maju",
    "customer_type": "Company",
    "customer_group": "Commercial",
    "territory": "Indonesia",
    "customer_primary_address": "<nama Address ter-link>",
    "customer_primary_contact": "<nama Contact ter-link>",
    "sales_team": [
      { "sales_person": "Kiki", "allocated_percentage": 100 }
    ]
  }'
```

> **Efek samping otomatis:** status Lead berubah menjadi **`Converted`** (`update_lead_status()`).
> Jika Lead dibuat tanpa `company_name` (individu), `customer_name` = `lead_name` — isi
> `company_name` pada Lead dulu bila seharusnya berupa perusahaan. Role konversi: `Sales User`
> (baca/tulis Lead & Customer).

> **Cegah konversi ulang:** karena backend tidak memblokirnya, jalankan **Langkah 0 — Pre-check**
> di atas sebelum konversi agar tidak terjadi Customer duplikat.

> **Postman:** request `5.10` (mapper `make_customer`) & `5.11` (POST Customer) di folder
> `5. Lead` (lihat §4).

## 3. GET pendukung UI — Lead

### 3.1 GET User (dropdown `lead_owner`) — `GET /api/resource/User`

`lead_owner` adalah **Link → User** (nilai = email user). Ambil daftar user aktif untuk dropdown:

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
    { "name": "kiki@perusahaan.co.id", "full_name": "Kiki", "email": "kiki@perusahaan.co.id" },
    { "name": "admin@perusahaan.co.id", "full_name": "Administrator", "email": "admin@perusahaan.co.id" }
  ]
}
```

> Kirim `name` (email user) pada field `lead_owner`. *(Dropdown lain — Territory, Company, UTM
> Source, Industry Type — mengikuti pola yang sama; dapat ditambahkan belakangan bila dipakai.)*

> **Postman:** request `5.9` di folder `5. Lead` (lihat §4).

## 4. Catatan tambahan — Lead

- **Autentikasi & token:** sama — lihat [prd_supplier.md §3](./prd_supplier.md) dan [`prd_oauth.md`](../prd_oauth.md).
- **Error umum:** sama — lihat [prd_supplier.md §6](./prd_supplier.md) (tambah: `DuplicateEntryError` untuk email duplikat).
- **Role:** baca & tulis `Sales User`; buat/hapus/export/import `Sales Manager`; `System Manager`.
- **Koleksi Postman:** contoh Lead ada di folder **`5. Lead`** pada
  `docs/postman/postman_erpnext_api.json`.
