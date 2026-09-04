# PRD — REST API Doctype Contact (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Contact** di ERPNext (Frappe),
> diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** CRM (ERPNext)
- **Doctype:** `Contact`
- **Versi API:** `/api/resource/...` (API v1)
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

---

Dokumentasi REST API untuk doctype **Contact**, mengikuti pola yang sama dengan
[Supplier](./prd_supplier.md) dan [Customer](./prd_customer.md). Autentikasi (OAuth 2.0) dan
penanganan error umum berlaku sama (lihat [prd_supplier.md §3](./prd_supplier.md) /
[`prd_oauth.md`](../prd_oauth.md) dan [prd_supplier.md §6](./prd_supplier.md)).

> **Peran Contact:** Contact adalah tempat **kanonik** untuk email & telepon milik pihak
> (Supplier/Customer). Field `email_id`/`mobile_no` di Supplier/Customer hanyalah cerminan
> read-only dari Contact primer. Contact **tidak punya perilaku auto-create** — dia justru
> yang dibuat otomatis oleh [Supplier §4.1](./prd_supplier.md) / [Customer §2.1](./prd_customer.md) (Varian A).

### Ruang lingkup — Contact

| # | Endpoint | Metode | Keterangan |
|---|---|---|---|
| 1 | `/api/resource/Contact` | POST | Buat Contact baru (CREATE) |
| 2 | `/api/resource/Contact/{name}` | GET | Ambil detail 1 Contact (READ) |
| 3 | `/api/resource/Contact` | GET | Daftar Contact (READ list) |
| 4 | `/api/resource/Contact/{name}` | PUT | Ubah Contact (UPDATE) |
| 5 | `/api/resource/Contact/{name}` | PUT | Non-aktifkan Contact (`status:"Passive"`) — pengganti DELETE |
| 6 | `/api/resource/Contact` | GET | Daftar Contact per party (filter Dynamic Link, §3) |
| 7 | `/api/method/frappe.core.doctype.data_export.exporter.export_data` | GET | Export Contact ke Excel/CSV (§2.6) |
| 8 | `/api/method/frappe.client.get_count` | GET | Total record Contact sesuai filter — untuk pagination / lazy loading (§2.2) |

---

## 1. Ringkasan field & data wajib — Contact

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB (min. 1)** | `first_name` *atau* `middle_name` *atau* `last_name` *atau* `company_name` | Data | `name` dibangun dari field ini (`get_full_name()`). Jika semua kosong → error **"Name is mandatory"**. |
| 🔴 **WAJIB (jika baris diisi)** | `email_ids[].email_id` | Data | `reqd: 1` |
| 🔴 **WAJIB (jika baris diisi)** | `phone_nos[].phone` | Data | `reqd: 1` |
| 🔴 **WAJIB (jika baris diisi)** | `links[].link_doctype` & `links[].link_name` | — | keduanya `reqd: 1` |
| 🔴 **WAJIB (CREATE) / 🟢 Opsional (UPDATE)** | `status` | Select | CREATE: wajib kirim `"Open"` (mencegah kosong). UPDATE: tidak wajib, tapi jika dikirim harus salah satu dari `Open`/`Passive`/`Replied` dan tidak boleh kosong. Lihat §2.1 & §2.4. |
| 🟢 Opsional | `salutation`, `gender`, `designation`, `department`, `is_primary_contact`, `unsubscribed`, `image`, `address` (Link → Address), `user` | — | sesuai kebutuhan |
| ⚪ **Otomatis — jangan dikirim** | `name`, `full_name`, `email_id`, `phone`, `mobile_no`, `link_title` | — | dihitung controller saat save |

### 1.1 Catatan penting — Contact

1. **`name` = nama lengkap** (atau `company_name` bila tanpa nama orang). Karena bisa berisi
   spasi, gunakan URL-encode (`%20`) pada GET/PUT/DELETE.
2. **Email & telepon diisi via child table** (`email_ids`, `phone_nos`) — field `email_id`,
   `phone`, `mobile_no` di level parent bersifat read-only (dihitung otomatis). Jangan dikirim.
3. **`links`** menghubungkan Contact ke pihak (Supplier/Customer, dsb.) — gunakan untuk mengaitkan
   contact ke sebuah Supplier/Customer (lihat §3).
4. **Role:** `Sales User`, `Purchase User`, `Accounts User`, dst. (baca & tulis luas).
5. **Field `address`** pada Contact hanyalah referensi *convenience* ke Address — alamat utama
   tetap milik party (lihat §4).
6. **Aturan `status` — CREATE & UPDATE (keputusan):**
   - **CREATE** (`POST /api/resource/Contact`): frontend **wajib** mengirim `status: "Open"` pada
     payload — mencegah contact baru berstatus kosong. Lihat §2.1.
   - **UPDATE** (`PUT /api/resource/Contact/{name}`): `status` **opsional**.
     - Tidak dikirim → nilai `status` lama dibiarkan (tidak berubah).
     - Dikirim → wajib salah satu dari `Open` / `Passive` / `Replied` dan **tidak boleh kosong**
       (`""`). Nilai kosong atau di luar daftar → frontend menampilkan error dan **menolak
       mengirim** request. Lihat §2.4.
   - **Perilaku backend:** nilai `status` di luar daftar opsi ditolak ERPNext dengan HTTP 417
     `ValidationError` (*"Value ... is not in the options of Status"*). Nilai kosong **tidak**
     ditolak backend (field tidak `reqd`), jadi validasi kosong wajib dilakukan di frontend.

---

## 2. CRUD — Doctype Contact

### 2.1 CREATE — `POST /api/resource/Contact`

**Payload minimum yang valid:**

```json
{
  "first_name": "Budi",
  "last_name": "Santoso",
  "status": "Open"
}
```

> Field `status` **wajib** dikirim = `"Open"` (lihat §1.1 poin 6) — mencegah data kosong.

**Contoh request (dengan child table):**

```bash
curl -X POST https://site-anda.com/api/resource/Contact \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "first_name": "Budi",
    "last_name": "Santoso",
    "status": "Open",
    "company_name": "PT Maju Jaya",
    "is_primary_contact": 1,
    "email_ids": [
      { "email_id": "budi@majujaya.co.id", "is_primary": 1 }
    ],
    "phone_nos": [
      { "phone": "+6281234567890", "is_primary_mobile_no": 1 }
    ],
    "links": [
      { "link_doctype": "Supplier", "link_name": "SUP-00001" }
    ]
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "data": {
    "name": "Budi Santoso",
    "owner": "Administrator",
    "creation": "2026-08-13 11:00:00.000000",
    "modified": "2026-08-13 11:00:00.000000",
    "modified_by": "Administrator",
    "docstatus": 0,
    "idx": 0,
    "first_name": "Budi",
    "last_name": "Santoso",
    "company_name": "PT Maju Jaya",
    "is_primary_contact": 1,
    "email_id": "budi@majujaya.co.id",
    "phone": "+6281234567890",
    "mobile_no": "+6281234567890",
    "email_ids": [
      {
        "name": "abc123",
        "email_id": "budi@majujaya.co.id",
        "is_primary": 1,
        "parent": "Budi Santoso",
        "parentfield": "email_ids",
        "parenttype": "Contact"
      }
    ],
    "phone_nos": [
      {
        "name": "def456",
        "phone": "+6281234567890",
        "is_primary_mobile_no": 1,
        "parent": "Budi Santoso",
        "parentfield": "phone_nos",
        "parenttype": "Contact"
      }
    ],
    "links": [
      {
        "name": "ghi789",
        "link_doctype": "Supplier",
        "link_name": "SUP-00001",
        "parent": "Budi Santoso",
        "parentfield": "links",
        "parenttype": "Contact"
      }
    ]
  }
}
```

> `name` = `Budi Santoso` → simpan nilai ini (URL-encode `Budi%20Santoso` untuk request berikutnya).

### 2.2 READ (satu record) — alur lengkap

**Langkah 1 — Ambil detail Contact:**

```bash
curl -X GET "https://site-anda.com/api/resource/Contact/Budi%20Santoso" \
  -H 'Authorization: Bearer <access_token>'
```

Respons `data` berisi seluruh field Contact, termasuk child table **`links`** (daftar party yang
terhubung: `link_doctype` + `link_name`) dan field **`address`** (Link ke Address — hanya terisi
jika alamat sudah di-set, lihat §4).

> **Catatan:** `data` **tidak** menyertakan nama display party (mis. `supplier_name`) maupun
> detail Address — hanya `link_name` (kunci, mis. `SUP-00001`) dan `address` (kunci Address).
> Ambil detail-nya di langkah 2 & 3.

**Contoh respons (HTTP 200) — cuplikan `links` & `address`:**

```json
{
  "data": {
    "name": "Budi Santoso",
    "full_name": "Budi Santoso",
    "email_id": "budi@majujaya.co.id",
    "mobile_no": "+6281234567890",
    "address": "PT Maju Jaya-Billing",
    "links": [
      { "link_doctype": "Supplier", "link_name": "SUP-00001" },
      { "link_doctype": "Customer", "link_name": "CUST-00001" }
    ]
  }
}
```

> `links[]` bersifat generik — bisa berisi doctype apa pun (Supplier, Customer, dan ke depan
> User, Lead, dsb.). UI sebaiknya melakukan *loop* atas `links[]`, bukan meng-hardcode
> `link_doctype`.

**Langkah 2 — Ambil info party (Supplier/Customer) dari `links[]`:**

Untuk tiap `links[i]`, GET doctype-nya dengan `link_name` untuk mendapat nama display
(`supplier_name` / `customer_name`):

```bash
curl -X GET "https://site-anda.com/api/resource/Supplier/SUP-00001" \
  -H 'Authorization: Bearer <access_token>'

curl -X GET "https://site-anda.com/api/resource/Customer/CUST-00001" \
  -H 'Authorization: Bearer <access_token>'
```

Contoh respons `data.supplier_name` = `"PT Maju Jaya"` dan `data.customer_name` = `"PT Maju Jaya"`
(struktur penuh lihat [prd_supplier.md §4.3](./prd_supplier.md) / [prd_customer.md §2.3](./prd_customer.md)).

**Langkah 3 — Ambil Address milik Contact (dari field `address`):**

Jika `data.address` terisi (mis. `PT Maju Jaya-Billing`), ambil detailnya:

```bash
curl -X GET "https://site-anda.com/api/resource/Address/PT%20Maju%20Jaya-Billing" \
  -H 'Authorization: Bearer <access_token>'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": {
    "name": "PT Maju Jaya-Billing",
    "address_title": "PT Maju Jaya",
    "address_type": "Billing",
    "address_line1": "Jl. Sudirman No. 123",
    "city": "Jakarta",
    "country": "Indonesia",
    "is_primary_address": 1
  }
}
```

> Jika `address` kosong (belum di-set), lewati langkah ini. Address "milik Contact" ini hanyalah
> referensi *convenience* — alamat utama transaksi tetap milik party (lihat §4).

Opsional, batasi field dengan query param `fields`:

```bash
curl -X GET "https://site-anda.com/api/resource/Contact/Budi%20Santoso?\
fields=[\"name\",\"full_name\",\"email_id\",\"mobile_no\",\"company_name\",\"links\",\"address\"]" \
  -H 'Authorization: Bearer <access_token>'
```

**Total record count — `GET /api/method/frappe.client.get_count`**

Untuk kebutuhan pagination / lazy loading (mis. menampilkan "900 dari 1000 Contact"), ambil
**total record yang cocok dengan filter** lewat method whitelisted `get_count`. Respons list
(`GET /api/resource/Contact`) **tidak** menyertakan total — hitung terpisah dengan endpoint ini.

```bash
curl -G "https://site-anda.com/api/method/frappe.client.get_count" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Contact' \
  --data-urlencode 'filters=[["is_primary_contact","=",1]]'
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

### 2.3 READ (daftar) — `GET /api/resource/Contact`

Parameter pilihan: `fields`, `filters`, `order_by`, `limit_start`, `limit_page_length`.

> **Lazy loading / pagination:** `limit_page_length` = jumlah record per halaman,
> `limit_start` = index awal (offset). Contoh ambil **per 50 data**:

```bash
# Halaman 1 — record 1–50
curl -G "https://site-anda.com/api/resource/Contact" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","full_name","email_id","mobile_no","company_name"]' \
  --data-urlencode 'filters=[["is_primary_contact","=",1]]' \
  --data-urlencode 'limit_start=0' \
  --data-urlencode 'limit_page_length=50'
```

> Halaman berikutnya naikkan `limit_start` kelipatan 50 (`50`, `100`, dst.) dengan
> `limit_page_length=50` tetap. Jumlah halaman = `total / 50`, dengan `total` diperoleh
> dari `get_count` (§2.2). `limit_page_length=0` = ambil **semua** record (tanpa LIMIT).

```bash
curl -G "https://site-anda.com/api/resource/Contact" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","full_name","email_id","mobile_no","company_name"]' \
  --data-urlencode 'filters=[["is_primary_contact","=",1]]' \
  --data-urlencode 'limit_page_length=20'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "Budi Santoso",
      "full_name": "Budi Santoso",
      "email_id": "budi@majujaya.co.id",
      "mobile_no": "+6281234567890",
      "company_name": "PT Maju Jaya"
    }
  ]
}
```

### 2.4 UPDATE — `PUT /api/resource/Contact/{name}`

Kirim **hanya field yang diubah**. `email_ids`, `phone_nos`, `links` bersifat **replace-all**
(bukan append).

**Aturan `status` saat UPDATE:**
- `status` **tidak wajib** dikirim. Tidak dikirim → nilai `status` lama tetap (tidak berubah).
- Jika `status` dikirim, nilainya **wajib salah satu dari** `Open` / `Passive` / `Replied` dan
  **tidak boleh kosong** (`""`).
- Nilai kosong atau di luar daftar opsi → frontend **memunculkan error** dan menolak mengirim
  request (mis. *"Status tidak valid."*). Backend juga menolak nilai di luar opsi dengan HTTP 417
  `ValidationError`.

Contoh di bawah tidak menyertakan `status` → status lama dibiarkan tidak berubah (valid).

```bash
curl -X PUT "https://site-anda.com/api/resource/Contact/Budi%20Santoso" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "designation": "Direktur",
    "email_ids": [
      { "email_id": "budi@baru.co.id", "is_primary": 1 }
    ]
  }'
```

**Contoh respons (HTTP 200):** objek `data` berisi dokumen terbaru (`email_id` dihitung ulang
 dari `email_ids`).

### 2.5 Non-aktifkan (disarankan) — `PUT /api/resource/Contact/{name}`

Contact **tidak punya field `disabled`** (berbeda dari Supplier/Customer). Non-aktifkan dengan
mengatur field `status` menjadi `"Passive"`:

```bash
curl -X PUT "https://site-anda.com/api/resource/Contact/Budi%20Santoso" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "status": "Passive" }'
```

**Contoh respons (HTTP 200):** objek `data` terbaru dengan `"status": "Passive"`.

> `"Passive"` adalah nilai valid per aturan §2.4 dan satu-satunya representasi "non-aktif"
> untuk Contact. Mengaktifkan kembali: `PUT` dengan `{ "status": "Open" }`.

> **Catatan penting:** `status` Contact bersifat **informasional** (opsi: `Passive` / `Open` /
> `Replied`). Men-set `Passive` **tidak menghapus data**, tapi contact **tetap bisa dipilih**
> sebagai contact person karena ERPNext tidak memfilter-nya. Jika suatu saat butuh "benar-benar
> non-aktif" (tidak bisa dipilih), perlu *custom field* `is_active` + penyesuaian backend.

> ⚠️ **Jangan gunakan `DELETE /api/resource/Contact/{name}`** — data contact (email/telepon/links)
> akan hilang permanen. Karena aturan "jangan hapus", endpoint DELETE tidak dipakai.

### 2.6 Export ke Excel / CSV — `GET /api/method/frappe.core.doctype.data_export.exporter.export_data`

Pola sama dengan [Supplier §4.6](./prd_supplier.md). Endpoint whitelisted, tanpa kustomisasi.

**Export Excel (.xlsx):**

```bash
curl -G "https://site-anda.com/api/method/frappe.core.doctype.data_export.exporter.export_data" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Contact' \
  --data-urlencode 'file_type=Excel' \
  --data-urlencode 'with_data=1' \
  --data-urlencode 'select_columns={"Contact":["name","full_name","first_name","last_name","company_name","email_id","mobile_no","status","is_primary_contact","designation"]}' \
  --data-urlencode 'all_doctypes=0' \
  -o contact.xlsx
```

**Export CSV:**

```bash
curl -G "https://site-anda.com/api/method/frappe.core.doctype.data_export.exporter.export_data" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Contact' \
  --data-urlencode 'file_type=CSV' \
  --data-urlencode 'with_data=1' \
  -o contact.csv
```

**Contoh respons:** file biner (attachment) — `filename="Contact.xlsx"` / `Contact.csv`.

**Parameter & catatan:** sama dengan [prd_supplier.md §4.6](./prd_supplier.md) (tabel parameter). Khusus Contact:
- **Child table:** `email_ids`, `phone_nos`, `links` — dengan `all_doctypes=1` (default) kolom
  child ikut diexport (spreadsheet "lebar"). Gunakan `all_doctypes=0` + `select_columns` agar rapi
  (seperti contoh di atas).
- **Izin export:** role harus punya hak **Export** pada doctype Contact (default: `Sales User`,
  `Purchase User`, `Accounts User`, dst.).
- **Alternatif template/import:** `download_template` ([§4.6](./prd_supplier.md)) — sama.
- **Alternatif frontend:** `GET /api/resource/Contact?fields=...&limit_page_length=0` → JSON →
  generate di klien.
- **Postman:** request `3.12` di folder `3. Contact` (lihat §5).

### 2.7 Import masal (bulk) — CSV / XLSX

Alur sama dengan [Supplier §4.7](./prd_supplier.md): upload file → buat `Data Import` → `form_start_import`.

**Payload Data Import (Langkah 2):**

```bash
curl -X POST https://site-anda.com/api/resource/Data%20Import \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "reference_doctype": "Contact",
    "import_type": "Insert New Records",
    "import_file": "/private/files/contact_import.csv",
    "mute_emails": 1,
    "status": "Pending"
  }'
```

> Detail upload (Langkah 1) & start/pantau import (Langkah 3): lihat **[prd_supplier.md §4.7](./prd_supplier.md)**.

**Catatan khusus Contact:**
- **Child table:** `email_ids`, `phone_nos`, `links` — isi lewat kolom template bernama
  `email_ids.email_id`, `phone_nos.phone`, `links.link_doctype`, `links.link_name`, dst.
- **`status`:** aturan nilai tetap berlaku — kolom `status` diisi `Open` / `Passive` / `Replied`
  (jangan kosong pada Insert).
- **Izin:** role harus punya hak **Import** pada doctype Contact.
- **Postman:** request `3.13` di folder `3. Contact` (lihat §5).

---

## 3. GET pendukung UI — Contact (filter party)

Untuk dropdown "pilih contact" milik sebuah Supplier/Customer, filter child table `links`
(Dynamic Link) pada Contact:

```bash
curl -G "https://site-anda.com/api/resource/Contact" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","full_name","email_id","mobile_no"]' \
  --data-urlencode 'filters=[[["Dynamic Link","link_doctype","=","Supplier"],["Dynamic Link","link_name","=","SUP-00001"]]]' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "Budi Santoso",
      "full_name": "Budi Santoso",
      "email_id": "budi@majujaya.co.id",
      "mobile_no": "+6281234567890"
    }
  ]
}
```

> Ganti `link_doctype`/`link_name` untuk party lain (`Customer`, `CUST-00001`, dst.).
>
> Contoh pemakaian dalam alur READ lengkap ada di **[prd_supplier.md §4.2](./prd_supplier.md)** (langkah 2 & 3).

---

## 4. Contoh ekstra — tautkan Address ke Contact (opsional)

Address utama tetap milik party (Supplier/Customer). Field `address` pada Contact hanya
referensi *convenience* ke Address. Langkahnya:

**Langkah 1 — Buat Address** (ter-link ke party, mis. Supplier — lihat [Varian B §4.1](./prd_supplier.md)). Simpan `name` (mis. `PT Maju Jaya-Billing`).

**Langkah 2 — Set `address` pada Contact:**

```bash
curl -X PUT "https://site-anda.com/api/resource/Contact/Budi%20Santoso" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "address": "PT Maju Jaya-Billing" }'
```

**Contoh respons (HTTP 200):** objek `data` berisi dokumen terbaru dengan `address` terisi.

> Catatan: menghubungkan Address langsung ke doctype `Contact` (lewat `links`) **tidak disarankan**
> — alamat itu tidak akan muncul di tab "Address" party dan tidak terpakai transaksi.

---

## 5. Catatan tambahan — Contact

- **Autentikasi & token:** sama — lihat [prd_supplier.md §3](./prd_supplier.md) dan [`prd_oauth.md`](../prd_oauth.md).
- **Error umum:** sama — lihat [prd_supplier.md §6](./prd_supplier.md).
- **Role:** `Sales User`, `Purchase User`, `Accounts User`, dst. (baca & tulis luas).
- **Koleksi Postman:** contoh Contact ada di folder **`3. Contact`** pada
  `docs/postman/postman_erpnext_api.json`.
