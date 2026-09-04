# PRD — REST API Doctype Customer (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Customer** di ERPNext (Frappe),
> diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Selling (ERPNext)
- **Doctype:** `Customer`
- **Versi API:** `/api/resource/...` (API v1)
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

---

Dokumentasi REST API untuk doctype **Customer**, mengikuti pola yang sama dengan
[Supplier](./prd_supplier.md). Autentikasi (OAuth 2.0) dan penanganan error umum berlaku sama
(lihat [prd_supplier.md §3](./prd_supplier.md) / [`prd_oauth.md`](../prd_oauth.md) dan
[prd_supplier.md §6](./prd_supplier.md)).

### Ruang lingkup — Customer

| # | Endpoint | Metode | Keterangan |
|---|---|---|---|
| 1 | `/api/resource/Customer` | POST | Buat Customer baru (CREATE) |
| 2 | `/api/resource/Customer/{name}` | GET | Ambil detail 1 Customer (READ) |
| 3 | `/api/resource/Customer` | GET | Daftar Customer (READ list) |
| 4 | `/api/resource/Customer/{name}` | PUT | Ubah Customer (UPDATE) |
| 5 | `/api/resource/Customer/{name}` | PUT | Non-aktifkan Customer (`disabled=1`) — pengganti DELETE |
| 6 | `/api/resource/Customer Group` | GET | Daftar Customer Group (dropdown UI) |
| 7 | `/api/resource/Territory` | GET | Daftar Territory (dropdown UI) |
| 8 | `/api/method/frappe.core.doctype.data_export.exporter.export_data` | GET | Export Customer ke Excel/CSV (§2.6) |
| 9 | `/api/method/frappe.client.get_count` | GET | Total record Customer sesuai filter — untuk pagination / lazy loading (§2.2) |

---

## 1. Ringkasan field & data wajib — Customer

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `customer_name` | Data | Nama customer (display name). `reqd: 1`. |
| 🔴 **WAJIB** | `customer_type` | Select | `reqd: 1`, default `Company`. Opsi: `Company`, `Individual`, `Partnership`. |
| 🟠 **DISARANKAN** | `customer_group` | Link → Customer Group | **Tidak** `reqd`, tapi dipakai di UI & praktik standar. Hanya node daun (`is_group = 0`). |
| 🟠 **DISARANKAN** | `territory` | Link → Territory | **Tidak** `reqd`; untuk dropdown UI (lihat §3.2). |
| ⚪ **Otomatis — jangan dikirim** | `name`, `naming_series` | — | `name` dibangkitkan otomatis oleh *naming series* `CUST-.YYYY.-` → mis. `CUST-00001`. |
| ⚪ **Read-only — jangan dikirim** | `email_id`, `mobile_no`, `primary_address`, `first_name`, `last_name`, `address_html`, `contact_html` | — | Diisi otomatis dari kontak/alamat primer. |

### 1.1 Catatan penting — Customer

1. **`name` vs `customer_name`** — `name` (kunci dokumen) otomatis menjadi `CUST-xxxxx`.
   Operasi GET/PUT/DELETE memakai `name`. *(Jika pengaturan global `Cust Master Name` di-set
   ke "Customer Name", maka `name = customer_name`.)*
2. **Auto-create Contact & Address** — sama dengan Supplier: **Contact** dibuat otomatis bila
   `email_id` / `mobile_no` terisi (**Varian A**, §2.1); **Address** tidak bisa dipicu lewat API
   (bukan field doctype Customer) — gunakan **Varian B** (§2.1). Saat DELETE, Contact & Address
   ter-link ikut terhapus. Sebelum CREATE, wajib **pre-check** nama — cek Contact & Supplier
   (lihat **Langkah 0** §2.1). Aturan yang sama berlaku saat UPDATE mengganti nama (lihat **§2.4**).
3. **Role yang dibutuhkan** — baca: `Sales User` / `Sales Manager`, `Accounts User` / `Accounts Manager`,
   `Stock User` / `Stock Manager`; buat & tulis: `Sales User`, `Sales Master Manager`; hapus:
   `Sales Master Manager`. *(Berbeda dari Supplier: `Sales User` sudah bisa create/write.)*
4. **`customer_group`** di form dibatasi ke node daun via link filter `[["Customer Group","is_group","=",0]]`.
   Field `territory` di doctype Customer **tidak** punya link filter itu — UI perlu memfilter sendiri
   (`is_group = 0`) bila hanya ingin node daun.

---

## 2. CRUD — Doctype Customer

### 2.1 CREATE — `POST /api/resource/Customer`

**Langkah 0 — Pre-check (wajib sebelum POST)**

Karena `customer_name` dipakai sebagai dasar nama auto-contact (Varian A) dan harus unik terhadap
master lain, UI wajib memastikan **dua hal** sebelum mengirim POST:

**a) Contact dengan nama yang sama belum ada:**

```bash
curl -G "https://site-anda.com/api/resource/Contact" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","status","company_name"]' \
  --data-urlencode 'filters=[["name","=","PT Maju Jaya"]]' \
  --data-urlencode 'limit_page_length=1'
```

**b) Supplier dengan nama yang sama belum ada (nama tidak boleh dipakai ganda):**

```bash
curl -G "https://site-anda.com/api/resource/Supplier" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","supplier_name"]' \
  --data-urlencode 'filters=[["supplier_name","=","PT Maju Jaya"]]' \
  --data-urlencode 'limit_page_length=1'
```

**Hasil & aturan:**
- Kedua `data` kosong (`[]`) → **lanjut** ke POST Customer.
- Salah satu `data` terisi → **blokir POST** dan tampilkan pesan, mis.
  - Contact ada → *"Contact dengan nama {customer_name} sudah ada. Gunakan contact tersebut / pilih nama lain."*
  - Supplier ada → *"Nama {customer_name} sudah dipakai sebagai Supplier. Gunakan nama lain."*
  *(Alternatif cek Contact: `GET /api/resource/Contact/{customer_name}` — HTTP 404 = belum ada.)*

> **Field yang dicek:** Contact → `name` (nama lengkap/company, sama dengan `customer_name` saat
> auto-contact). Supplier → `supplier_name` (nama tampilan), **bukan** `name` (ID otomatis
> `SUP-` yang belum diketahui sebelum POST).

> **Catatan status "aktif":** field `status` di doctype Contact hanya berisi
> `Passive` / `Open` / `Replied` — **tidak ada opsi "Active"**. Status informasional; rekomendasi:
> blokir berdasarkan keberadaan nama saja. Jika tetap ingin filter "aktif", petakan sendiri
> (mis. `status == "Open"`) atau tambah *custom field* `is_active` di Contact (butuh kustomisasi
> backend).

> **Catatan backend:** meski tanpa pre-check, ERPNext melempar `DuplicateEntryError` saat
> auto-create Contact menemukan nama yang sama (POST Customer gagal). Pre-check memberi pesan
> ramah & lebih cepat, sekaligus mencegah nama ganda di Supplier.

**Payload minimum (data wajib + disarankan):**

```json
{
  "customer_name": "PT Maju Jaya",
  "customer_group": "Commercial",
  "territory": "Indonesia",
  "customer_type": "Company"
}
```

**Contoh request:**

```bash
curl -X POST https://site-anda.com/api/resource/Customer \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "customer_name": "PT Maju Jaya",
    "customer_group": "Commercial",
    "territory": "Indonesia",
    "customer_type": "Company"
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "data": {
    "name": "CUST-00001",
    "owner": "Administrator",
    "creation": "2026-08-13 10:00:00.000000",
    "modified": "2026-08-13 10:00:00.000000",
    "modified_by": "Administrator",
    "docstatus": 0,
    "idx": 0,
    "naming_series": "CUST-.YYYY.-",
    "customer_name": "PT Maju Jaya",
    "customer_group": "Commercial",
    "territory": "Indonesia",
    "customer_type": "Company",
    "disabled": 0,
    "is_frozen": 0,
    "accounts": []
  }
}
```

> `name` = `CUST-00001` → simpan nilai ini; dipakai untuk GET/PUT/DELETE berikutnya.

**Varian A — CREATE + auto-generate Contact (opsional)** *(pola sama seperti Supplier)*

Jika body juga berisi `email_id` dan/atau `mobile_no`, ERPNext otomatis membuat **Contact**
primer yang ter-link ke Customer, bernama sesuai `customer_name`:

| `customer_type` | Contact yang dibuat otomatis |
|---|---|
| `Company` / `Partnership` | `company_name = customer_name` (tanpa first/last name) |
| `Individual` | `first/middle/last_name` dipecah dari `customer_name` |

> Tidak berlaku bila Customer dibuat dari Lead (`lead_name` terisi, read-only).

> **Pre-check:** Langkah 0 di atas wajib dijalankan sebelum POST — auto-contact Varian A akan
> bernama persis `customer_name`, dan aturan cek Contact + Supplier tetap berlaku di sini.

```bash
curl -X POST https://site-anda.com/api/resource/Customer \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "customer_name": "PT Maju Jaya",
    "customer_group": "Commercial",
    "territory": "Indonesia",
    "customer_type": "Company",
    "email_id": "info@majujaya.co.id",
    "mobile_no": "+6281234567890"
  }'
```

Verifikasi Contact yang dibuat: `GET /api/resource/Contact/PT%20Maju%20Jaya`.

> **Catatan `status` pada auto-create Contact (penting):** Contact yang dibuat otomatis oleh
> Customer ini **tidak bisa diatur status-nya dari frontend** — diisi oleh backend, dan secara
> default ERPNext mengisinya `"Passive"`. Jika syaratnya semua contact baru (termasuk yang dibuat
> otomatis) harus berstatus `"Open"`, perlu perubahan di sisi backend/admin: ubah default field
> `status` doctype Contact menjadi `"Open"` (Customize Form → Contact → Status → Default), atau
> pasang *server script*/hook `doc_events` yang mengisi `status="Open"` saat create. Tanpa
> perubahan itu, contact auto-create tetap `"Passive"`.

**Varian B — CREATE Customer + daftarkan Address (cara andal lewat API)** *(pola sama seperti Supplier)*

**Langkah 1 — Buat Customer** (di atas). Simpan `name`, mis. `CUST-00001`.

**Langkah 2 — Buat Address yang ter-link ke Customer:**

```bash
curl -X POST https://site-anda.com/api/resource/Address \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "address_title": "PT Maju Jaya",
    "address_type": "Billing",
    "address_line1": "Jl. Sudirman No. 123",
    "city": "Jakarta",
    "country": "Indonesia",
    "is_primary_address": 1,
    "links": [
      { "link_doctype": "Customer", "link_name": "CUST-00001" }
    ]
  }'
```

Field wajib Address: `address_type`, `address_line1`, `city`, `country`. Simpan `name` — aturan
autoname: **`name = "{address_title}-{address_type}"`**. Karena `PT Maju Jaya-Billing` sudah
dipakai contoh Supplier, di sini menjadi `PT Maju Jaya-Billing-1` (duplikat → akhiran `-1`).

**Langkah 3 — Jadikan Address itu primary di Customer:**

```bash
curl -X PUT "https://site-anda.com/api/resource/Customer/CUST-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "customer_primary_address": "PT Maju Jaya-Billing-1" }'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": {
    "name": "CUST-00001",
    "customer_name": "PT Maju Jaya",
    "customer_group": "Commercial",
    "territory": "Indonesia",
    "customer_type": "Company",
    "customer_primary_address": "PT Maju Jaya-Billing-1"
  }
}
```

**Varian C — Menambahkan contact person yang sudah ada & set primary** *(pola sama seperti [Supplier §4.1](./prd_supplier.md))*

**Kasus:** Customer `CUST-00001` (PT Maju Jaya) memakai **Markus** dan **Tommy** — dua Contact
 yang **sudah ada** di `tabContact` — sebagai contact person; **Markus** dijadikan primary.

**Catatan konsep:**
- Menjadikan seseorang contact person = Contact tsb ter-link ke Customer (child table `links`).
- Menetapkan primary = field **`customer_primary_contact`** di **Customer** (bukan di Contact).
- Karena contact sudah ada, `customer_primary_contact` bisa **langsung diisi saat CREATE**
  (auto-create Contact dari Varian A akan dilewati karena field ini sudah terisi).

**Langkah 1 — CREATE Customer, langsung set primary contact (Markus):**

```bash
curl -X POST https://site-anda.com/api/resource/Customer \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "customer_name": "PT Maju Jaya",
    "customer_group": "Commercial",
    "territory": "Indonesia",
    "customer_type": "Company",
    "customer_primary_contact": "Markus"
  }'
```

> `customer_primary_contact` hanya valid bila Contact **"Markus" sudah ada** (Link field divalidasi
> saat save — kalau belum ada akan error link).

**Langkah 2 — (UPDATE) mengganti primary di kemudian hari:**

```bash
curl -X PUT "https://site-anda.com/api/resource/Customer/CUST-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "customer_primary_contact": "Tommy" }'
```

**Menautkan contact yang sudah ada (jika belum ter-link):** Markus & Tommy baru dianggap
"contact person" bila `links`-nya mengarah ke Customer. Bila belum, update Contact-nya (PUT)
dengan **semua** `links` (lama + baru) karena `links` bersifat **replace-all**:

```bash
curl -X PUT "https://site-anda.com/api/resource/Contact/Markus" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "links": [
      { "link_doctype": "Customer", "link_name": "CUST-00001" }
    ]
  }'
```

**Hasil:** tab "Address" dan tab "Contact" Customer menampilkan Markus & Tommy; `email_id`/`mobile_no`
Customer ter-fetch dari Markus; transaksi (SO/PI) memakai Markus sebagai kontak default.

### 2.2 READ (satu record) — alur lengkap

**Langkah 1 — Ambil detail Customer:**

```bash
curl -X GET "https://site-anda.com/api/resource/Customer/CUST-00001" \
  -H 'Authorization: Bearer <access_token>'
```

Respons `data` berisi seluruh field Customer (struktur seperti respons CREATE), termasuk
`customer_primary_contact` / `customer_primary_address` (hanya nama link) dan
`email_id`/`mobile_no` (dari contact primer).

> **Catatan:** `data` **tidak** menyertakan daftar lengkap address/contact person yang ter-link
> (daftar itu hanya diisi lewat `onload` pada form Desk, bukan REST API). Ambil terpisah seperti
> langkah 2 & 3 di bawah.

**Langkah 2 — Ambil daftar contact person milik Customer (filter Dynamic Link):**

```bash
curl -G "https://site-anda.com/api/resource/Contact" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","full_name","email_id","mobile_no","is_primary_contact"]' \
  --data-urlencode 'filters=[[["Dynamic Link","link_doctype","=","Customer"],["Dynamic Link","link_name","=","CUST-00001"]]]' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "Markus",
      "full_name": "Markus",
      "email_id": "markus@majujaya.co.id",
      "mobile_no": "+6281234567890",
      "is_primary_contact": 1
    }
  ]
}
```

**Langkah 3 — Ambil daftar address milik Customer (filter Dynamic Link):**

```bash
curl -G "https://site-anda.com/api/resource/Address" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","address_title","address_type","address_line1","city","country","is_primary_address"]' \
  --data-urlencode 'filters=[[["Dynamic Link","link_doctype","=","Customer"],["Dynamic Link","link_name","=","CUST-00001"]]]' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "PT Maju Jaya-Billing-1",
      "address_title": "PT Maju Jaya",
      "address_type": "Billing",
      "address_line1": "Jl. Sudirman No. 123",
      "city": "Jakarta",
      "country": "Indonesia",
      "is_primary_address": 1
    }
  ]
}
```

**Total record count — `GET /api/method/frappe.client.get_count`**

Untuk kebutuhan pagination / lazy loading (mis. menampilkan "900 dari 1000 Customer"), ambil
**total record yang cocok dengan filter** lewat method whitelisted `get_count`. Respons list
(`GET /api/resource/Customer`) **tidak** menyertakan total — hitung terpisah dengan endpoint ini.

```bash
curl -G "https://site-anda.com/api/method/frappe.client.get_count" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Customer' \
  --data-urlencode 'filters=[["disabled","=",0]]'
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

### 2.3 READ (daftar) — `GET /api/resource/Customer`

Parameter pilihan: `fields`, `filters`, `order_by`, `limit_start`, `limit_page_length`.

> **Lazy loading / pagination:** `limit_page_length` = jumlah record per halaman,
> `limit_start` = index awal (offset). Contoh ambil **per 50 data**:

```bash
# Halaman 1 — record 1–50
curl -G "https://site-anda.com/api/resource/Customer" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","customer_name","customer_group","territory","customer_type","disabled"]' \
  --data-urlencode 'filters=[["customer_group","=","Commercial"]]' \
  --data-urlencode 'limit_start=0' \
  --data-urlencode 'limit_page_length=50'
```

> Halaman berikutnya naikkan `limit_start` kelipatan 50 (`50`, `100`, dst.) dengan
> `limit_page_length=50` tetap. Jumlah halaman = `total / 50`, dengan `total` diperoleh
> dari `get_count` (§2.2). `limit_page_length=0` = ambil **semua** record (tanpa LIMIT),
> seperti contoh di bawah ini.

```bash
curl -G "https://site-anda.com/api/resource/Customer" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","customer_name","customer_group","territory","customer_type","disabled"]' \
  --data-urlencode 'filters=[["customer_group","=","Commercial"]]' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "CUST-00001",
      "customer_name": "PT Maju Jaya",
      "customer_group": "Commercial",
      "territory": "Indonesia",
      "customer_type": "Company",
      "disabled": 0
    }
  ]
}
```

### 2.4 UPDATE — `PUT /api/resource/Customer/{name}`

Kirim **hanya field yang diubah**.

**Langkah 0 — Pre-check nama (hanya saat `customer_name` diganti)**

Aturan nama ganda (lihat §2.1 Langkah 0) tetap berlaku saat UPDATE, dengan **self-exclusion**:

- Jalankan cek **hanya jika `customer_name` berubah** (bukan setiap PUT).
- **Abaikan milik sendiri:** Contact primer milik Customer ini dan Customer itu sendiri tidak
  dianggap konflik (supaya PUT tanpa perubahan nama tidak diblokir).

```bash
# (a) Contact dengan nama baru (selain milik Customer ini) belum ada
curl -G "https://site-anda.com/api/resource/Contact" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","company_name"]' \
  --data-urlencode 'filters=[["name","=","PT Maju Jaya Tbk"]]' \
  --data-urlencode 'limit_page_length=1'

# (b) Supplier dengan nama baru belum ada
curl -G "https://site-anda.com/api/resource/Supplier" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","supplier_name"]' \
  --data-urlencode 'filters=[["supplier_name","=","PT Maju Jaya Tbk"]]' \
  --data-urlencode 'limit_page_length=1'
```

**Aturan:** jika salah satu `data` terisi (selain milik Customer ini) → **blokir PUT** dan tampilkan
pesan, mis. *"Nama {customer_name} sudah dipakai. Gunakan nama lain."*

> **Catatan backend:** rename `customer_name` **tidak** memicu auto-create Contact (karena
> `customer_primary_contact` sudah terisi) sehingga backend **tidak** memblokir konflik nama — cek
> ini wajib di sisi UI. Pengecualian: jika Customer dibuat tanpa `email_id`/`mobile_no` (primary
> kosong) lalu di UPDATE ditambahkan, auto-contact akan dibuat dengan nama baru — cek (a) melindungi
> dari `DuplicateEntryError`. Rename juga **tidak mengubah nama** auto-contact lama (nama contact
> tetap nama lama).

```bash
curl -X PUT "https://site-anda.com/api/resource/Customer/CUST-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "customer_name": "PT Maju Jaya Tbk",
    "customer_group": "Retail"
  }'
```

**Contoh respons (HTTP 200):** objek `data` berisi dokumen terbaru (field berubah).

### 2.5 Non-aktifkan (disarankan) — `PUT /api/resource/Customer/{name}`

**Aturan:** data Customer **tidak boleh dihapus**, hanya dinon-aktifkan. Non-aktifkan dengan
mengirim field `disabled`:

```bash
curl -X PUT "https://site-anda.com/api/resource/Customer/CUST-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "disabled": 1 }'
```

**Contoh respons (HTTP 200):** objek `data` terbaru dengan `"disabled": 1`.

Untuk mengaktifkan kembali: `PUT` dengan `{ "disabled": 0 }`.

> **Efek non-aktif:** Customer tidak muncul sebagai pilihan pada transaksi baru (SO / Sales
> Invoice). **Contact & Address yang ter-link TIDAK dihapus** — tetap utuh.

> ⚠️ **Jangan gunakan `DELETE /api/resource/Customer/{name}`** — saat DELETE, Contact & Address
> ter-link ikut terhapus dan gagal bila Customer sudah dipakai transaksi (*"linked with"*).
> Karena aturan master data "jangan hapus", endpoint DELETE tidak dipakai di aplikasi.

### 2.6 Export ke Excel / CSV — `GET /api/method/frappe.core.doctype.data_export.exporter.export_data`

Pola sama dengan [Supplier §4.6](./prd_supplier.md). Endpoint whitelisted, tanpa kustomisasi.

**Export Excel (.xlsx):**

```bash
curl -G "https://site-anda.com/api/method/frappe.core.doctype.data_export.exporter.export_data" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Customer' \
  --data-urlencode 'file_type=Excel' \
  --data-urlencode 'with_data=1' \
  --data-urlencode 'select_columns={"Customer":["name","customer_name","customer_group","territory","customer_type","disabled"]}' \
  --data-urlencode 'filters=[["customer_group","=","Commercial"]]' \
  -o customer.xlsx
```

**Export CSV:**

```bash
curl -G "https://site-anda.com/api/method/frappe.core.doctype.data_export.exporter.export_data" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Customer' \
  --data-urlencode 'file_type=CSV' \
  --data-urlencode 'with_data=1' \
  -o customer.csv
```

**Contoh respons:** file biner (attachment) — `filename="Customer.xlsx"` / `Customer.csv`.

**Parameter & catatan:** sama dengan [prd_supplier.md §4.6](./prd_supplier.md) (tabel parameter). Khusus Customer:
- **Izin export:** role harus punya hak **Export** pada doctype Customer (default: `Sales User`,
  `Accounts User`, `Stock User`, dst.).
- **Alternatif template/import:** `download_template` ([§4.6](./prd_supplier.md)) — sama.
- **Alternatif frontend:** `GET /api/resource/Customer?fields=...&limit_page_length=0` → JSON →
  generate di klien.
- **Postman:** request `2.16` di folder `2. Customer` (lihat §4).

### 2.7 Import masal (bulk) — CSV / XLSX

Alur sama dengan [Supplier §4.7](./prd_supplier.md): upload file → buat `Data Import` → `form_start_import`.

**Payload Data Import (Langkah 2):**

```bash
curl -X POST https://site-anda.com/api/resource/Data%20Import \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "reference_doctype": "Customer",
    "import_type": "Insert New Records",
    "import_file": "/private/files/customer_import.csv",
    "mute_emails": 1,
    "status": "Pending"
  }'
```

> Detail upload (Langkah 1) & start/pantau import (Langkah 3): lihat **[prd_supplier.md §4.7](./prd_supplier.md)**.

**Catatan khusus Customer:**
- **Izin:** role harus punya hak **Import** pada doctype Customer (default: `Sales User`,
  `Sales Master Manager`, dst.).
- **Child table:** `accounts` (akun per-company) ikut didukung bila kolomnya ada di template.
- **Catatan nama:** pre-check nama (Langkah 0 §2.1) TIDAK berlaku untuk import masal — baris
  duplikat di-update/di-skip sesuai `import_type`. Upsert memakai kolom `name`.
- **Postman:** request `2.17` di folder `2. Customer` (lihat §4).

---

## 3. GET pendukung UI — Customer

### 3.1 GET Customer Group — `GET /api/resource/Customer Group`

```bash
curl -G "https://site-anda.com/api/resource/Customer%20Group" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","customer_group_name","parent_customer_group","is_group"]' \
  --data-urlencode 'filters=[["is_group","=",0]]' \
  --data-urlencode 'order_by=customer_group_name asc' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "Commercial",
      "customer_group_name": "Commercial",
      "parent_customer_group": "All Customer Groups",
      "is_group": 0
    },
    {
      "name": "Retail",
      "customer_group_name": "Retail",
      "parent_customer_group": "All Customer Groups",
      "is_group": 0
    }
  ]
}
```

### 3.2 GET Territory — `GET /api/resource/Territory`

```bash
curl -G "https://site-anda.com/api/resource/Territory" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","territory_name","parent_territory","is_group"]' \
  --data-urlencode 'filters=[["is_group","=",0]]' \
  --data-urlencode 'order_by=territory_name asc' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "Indonesia",
      "territory_name": "Indonesia",
      "parent_territory": "All Territories",
      "is_group": 0
    },
    {
      "name": "Singapore",
      "territory_name": "Singapore",
      "parent_territory": "All Territories",
      "is_group": 0
    }
  ]
}
```

> Untuk dropdown Territory di form Customer: ambil yang `is_group = 0` (node daun). Field
> `territory` di doctype Customer tidak punya link filter otomatis, jadi UI yang memfilter.

### 3.3 CREATE Customer Group — `POST /api/resource/Customer Group`

`Customer Group` adalah **master konfigurasi & tree**. `name` = `customer_group_name`
(`autoname: field:customer_group_name`).

**Payload minimum:**

```json
{
  "customer_group_name": "Commercial",
  "parent_customer_group": "All Customer Groups",
  "is_group": 0
}
```

**Contoh request:**

```bash
curl -X POST https://site-anda.com/api/resource/Customer%20Group \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "customer_group_name": "Commercial",
    "parent_customer_group": "All Customer Groups",
    "is_group": 0
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "data": {
    "name": "Commercial",
    "customer_group_name": "Commercial",
    "parent_customer_group": "All Customer Groups",
    "is_group": 0
  }
}
```

> `name` = `Commercial` → simpan nilai ini. `parent_customer_group` opsional (bila diisi harus
> node `is_group = 1`). `is_group = 1` = node parent; `is_group = 0` = node daun (dipakai
> transaksi). Field opsional lain: `default_price_list`, `payment_terms`, `credit_limits`.
> Role buat: `Sales Master Manager`.

### 3.4 UPDATE Customer Group — `PUT /api/resource/Customer Group/{name}`

Kirim **hanya field yang diubah** (mis. pindah parent / `is_group` / `payment_terms` /
`default_price_list`):

```bash
curl -X PUT "https://site-anda.com/api/resource/Customer%20Group/Commercial" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "parent_customer_group": "All Customer Groups",
    "payment_terms": ""
  }'
```

**Contoh respons (HTTP 200):** objek `data` berisi dokumen terbaru.

> **Catatan nama:** `name` mengikuti `customer_group_name` (autoname by fieldname — hanya berlaku
> saat insert). Mengganti `customer_group_name` pada dokumen yang sudah ada **tidak** otomatis
> mengganti `name` — hindari rename via REST.

### 3.5 DELETE Customer Group — `DELETE /api/resource/Customer Group/{name}`

⚠️ **Customer Group TIDAK punya field `disabled`** — jadi "non-aktif" native tidak tersedia
(berbeda dari Customer/Supplier/Address). Opsi:

- **Non-aktif (disarankan):** pindahkan Customer yang memakai group ini ke group lain
  (`PUT /api/resource/Customer/{name}` → ganti `customer_group`), lalu berhenti memakai group lama.
- **DELETE** (hanya bila benar-benar yakin):

```bash
curl -X DELETE "https://site-anda.com/api/resource/Customer%20Group/Commercial" \
  -H 'Authorization: Bearer <access_token>'
```

**Contoh respons (HTTP 202):**

```json
{
  "data": "ok"
}
```

> **Batasan DELETE (tree):**
> - Hanya bisa menghapus node **daun** (`is_group = 0`) — node parent yang masih punya child
>   **tidak bisa** dihapus (error *"linked with"* / child exists).
> - Group yang **masih dipakai Customer** (`customer_group` merujuk ke sini) **tidak bisa** dihapus
>   — error *"linked with"*.
> - Role hapus: `Sales Master Manager`.

---

## 4. Catatan tambahan — Customer

- **Autentikasi & token:** sama seperti [Supplier §3](./prd_supplier.md) — lihat juga [`prd_oauth.md`](../prd_oauth.md).
- **Error umum:** sama — lihat [prd_supplier.md §6](./prd_supplier.md).
- **Role:** baca `Sales User` / `Sales Manager`, `Accounts User` / `Accounts Manager`, `Stock User` / `Stock Manager`;
  buat & tulis `Sales User` + `Sales Master Manager`; hapus `Sales Master Manager`.
- **Koleksi Postman:** contoh Customer ada di folder **`2. Customer`** pada
  `docs/postman/postman_erpnext_api.json`.
