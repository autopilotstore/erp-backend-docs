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
| 10 | `/api/resource/Customer/{name}` | GET + PUT | Baca / set **credit limit** (limitasi piutang) lewat child table `credit_limits` (§2.8) |
| 11 | `/api/resource/Payment Terms Template` + `/api/resource/Payment Term` | GET + POST | Termin / cicilan pembayaran: dropdown template, detail termin, dan pembuatan master (§2.9) |

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

### 2.8 Credit limit (limitasi piutang) — child table `Customer Credit Limit`

Limitasi piutang per Customer disimpan di **child table** `Customer Credit Limit`, yang diakses
lewat field **`credit_limits`** pada dokumen Customer (label form: *"Credit & Overdue Limits"*).

| Field | Tipe | Keterangan |
|---|---|---|
| `company` | Link → Company | **Kunci baris** — satu baris per company, tidak boleh duplikat. |
| `credit_limit` | Currency | Batas piutang. `0`/kosong = tidak ada limit **di baris ini** (bukan berarti "unlimited"; lihat §2.8.4). |
| `bypass_credit_limit_check` | Check | Label UI: **"Bypass credit limit check at sales order"**. Lihat §2.8.3. |
| `overdue_billing_threshold` | Currency | Label UI: **"Overdue Limit"** — `hidden: 1`, hanya berlaku bila *Accounts Settings → Restrict Customer Over Billing* aktif. Lihat §2.8.4. |

> **Identitas baris:** tiap baris punya `name` sendiri (hash yang dibangkitkan server — **bukan**
> `company`). Kirimkan `name` ini saat PUT agar baris yang sama di-update, bukan dihapus lalu
> dibuat ulang.

#### 2.8.1 Cara menulis — wajib lewat parent Customer (replace-all)

`Customer Credit Limit` adalah **child table** (`istable: 1`) dan **tidak punya DocPerm sendiri**.
Jangan panggil `/api/resource/Customer Credit Limit` (POST/PUT ke child doctype ditolak — permission
child selalu dievaluasi lewat parent). Semua operasi lewat parent: `POST`/`PUT /api/resource/Customer`.

`PUT /api/resource/Customer/{name}` menjalankan `doc.update(data)` + `doc.save()`, sehingga field
`credit_limits` bersifat **replace-all**: array yang dikirim **menggantikan seluruh isi tabel** dan
baris yang tidak disertakan akan **terhapus**. Urutan yang benar: **GET dulu → ubah satu baris →
PUT array lengkap.**

**Langkah 1 — GET data limit yang ada (termasuk `name` tiap baris):**

```bash
curl -G "https://site-anda.com/api/resource/Customer/CUST-00001" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","customer_name","credit_limits"]'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": {
    "name": "CUST-00001",
    "customer_name": "PT Maju Jaya",
    "credit_limits": [
      {
        "name": "a1b2c3d4e5",
        "doctype": "Customer Credit Limit",
        "parent": "CUST-00001",
        "parentfield": "credit_limits",
        "parenttype": "Customer",
        "company": "PT Maju Jaya",
        "credit_limit": 50000000,
        "overdue_billing_threshold": 0,
        "bypass_credit_limit_check": 0
      }
    ]
  }
}
```

**Langkah 2 — set / ubah limit (PUT array penuh):**

```bash
curl -X PUT "https://site-anda.com/api/resource/Customer/CUST-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "credit_limits": [
      {
        "name": "a1b2c3d4e5",
        "company": "PT Maju Jaya",
        "credit_limit": 75000000,
        "bypass_credit_limit_check": 0
      }
    ]
  }'
```

**Contoh respons (HTTP 200):** objek `data` terbaru, `credit_limits[0].credit_limit = 75000000`.

> **Catatan UI:** selalu kirim **semua** baris hasil Langkah 1 (bukan hanya baris yang diubah).
> Menghilangkan satu baris = menghapus limit company tersebut.

**Langkah 3 (opsional) — langsung set limit saat CREATE Customer:** tambahkan `credit_limits` pada
payload `POST /api/resource/Customer` (§2.1):

```json
{
  "customer_name": "PT Maju Jaya",
  "customer_group": "Commercial",
  "territory": "Indonesia",
  "customer_type": "Company",
  "credit_limits": [
    {
      "company": "PT Maju Jaya",
      "credit_limit": 75000000,
      "bypass_credit_limit_check": 0
    }
  ]
}
```

**Validasi server yang harus diantisipasi UI** (`Customer.validate_credit_limit_on_change`, hanya
jalan pada dokumen yang sudah ada — jadi muncul saat PUT, bukan POST):

| Kondisi | Perilaku |
|---|---|
| Dua baris dengan `company` sama | **Gagal.** *"Credit limit is already defined for the Company {0}"* |
| Limit baru **<** outstanding saat ini | **Gagal.** *"New credit limit is less than current outstanding amount for the customer. Credit limit has to be atleast {0}"* (nilai `{0}` = outstanding berjalan) |
| Limit baru **≥** outstanding | Sukses, `credit_limits` diganti |

> Karena validasi kedua dihitung dari outstanding **berjalan**, PUT bisa gagal walau payload sudah
> benar secara format. Tampilkan pesan error dari server apa adanya ke user.

#### 2.8.2 Menghapus / "mengosongkan" limit

Tidak ada endpoint DELETE untuk satu baris limit. Dua cara yang tersedia, **efeknya berbeda**:

| Aksi | Payload | Efek |
|---|---|---|
| Hapus baris | PUT dengan `credit_limits` **tanpa** baris company tsb. | Baris hilang dari tabel → limit efektif **jatuh ke fallback** (§2.8.4), bukan otomatis tanpa limit. |
| Kosongkan nilai | PUT dengan baris tetap ada, `"credit_limit": 0` | Sama: limit baris itu dianggap tidak diset → **jatuh ke fallback**. |

> Karena keduanya berujung ke fallback (Customer Group → default Company), **"hapus limit" di UI
> harus dikonfirmasi** dan sebaiknya menampilkan limit efektif hasil fallback tersebut.

#### 2.8.3 `bypass_credit_limit_check` — "Bypass credit limit check at sales order"

Bersifat **per baris company**, bukan global. Fungsinya **memindahkan titik blokir dari Sales Order
ke Delivery Note / Sales Invoice**, bukan mematikan limit. Perilaku di kode backend:

| Tahap | `bypass = 0` | `bypass = 1` |
|---|---|---|
| **Sales Order** | Dicek penuh (`check_credit_limit`); SO ikut dihitung sebagai outstanding. | `SalesOrder.check_credit_limit()` langsung `return` → SO **tidak pernah diblokir**. |
| **Outstanding** | GLE + Sales Order + Delivery Note non-SO. | SO **tidak dihitung** (`ignore_outstanding_sales_order=True`) — hanya GLE + DN. |
| **Delivery Note** | Divalidasi untuk item yang belum terhubung SO/DN. | Hanya divalidasi untuk item yang belum punya Sales Invoice, plus `extra_amount = base_grand_total` DN. |
| **Sales Invoice** | Divalidasi hanya bila ada item tanpa SO/DN. | **Selalu** divalidasi. |

**Implikasi untuk UI:** dengan `bypass = 1`, user bisa membuat Sales Order melebihi limit, tetapi
blokir tetap datang saat membuat Delivery Note / Sales Invoice. Tampilkan status ini di form Customer
(mis. badge *"Limit diperiksa di Invoice"*) agar user tidak mengira limit sudah dibebaskan.

> **Catatan istilah (sering tertukar):** `bypass_credit_limit_check_at_sales_order` adalah **nama
> field lama di level doctype Customer** (v11). Patch v12
> `move_credit_limit_to_customer_credit_limit.py` memindahkannya ke child table sebagai
> `bypass_credit_limit_check` **per company**. Di UI maupun REST API sekarang **tidak ada** lagi
> field `bypass_credit_limit_check_at_sales_order` di Customer — nama itu hanya tersisa sebagai nama
> variabel di dalam kode SO/DN/SI.

#### 2.8.4 Limit efektif (fallback), Overdue Limit & pengaturan Accounts Settings

**Limit efektif** tidak hanya dibaca dari Customer. `erpnext.selling.doctype.customer.customer.get_credit_limit()`
meresolusi berurutan:

1. `Customer Credit Limit` dengan `parenttype = "Customer"` (baris Customer tsb),
2. jika kosong/`0` → `Customer Credit Limit` dengan `parenttype = "Customer Group"` untuk
   `customer_group` Customer itu (**Customer Group juga punya tabel `credit_limits`**),
3. jika masih kosong → default `credit_limit` pada doctype **Company**.

`check_credit_limit()` `return` lebih awal bila hasil resolusi = `0` (benar-benar tanpa pengecekan).
Jadi `credit_limit: 0` di Customer = "pakai limit Group/Company", bukan "tanpa batas".

**Overdue Limit (`overdue_billing_threshold`)** — hanya aktif bila *Accounts Settings* diisi:

| Accounts Settings | Label | Fungsi |
|---|---|---|
| `enable_overdue_billing_threshold` | *Restrict Customer Over Billing* | Menyalakan pengecekan Overdue Limit. |
| `role_allowed_to_bypass_overdue_billing` | *Role Allowed to Bypass Over Billing Restriction* | Role yang boleh tetap submit Sales Invoice meski Overdue Limit terlampaui. |
| `credit_controller` | *Role allowed to bypass credit limit* | Role yang boleh menembus credit limit tanpa diblokir. |
| `over_billing_allowance` (+ `role_allowed_to_over_bill`) | *Over Billing Allowance (%)* | Toleransi over-billing terhadap nilai order — terpisah dari credit limit. |

Saat submit Sales Invoice: `check_overdue_billing_threshold()` menghitung overdue dari Payment Ledger;
if `overdue_amount > threshold` dan user tidak punya role bypass → error
*"Overdue Limit crossed for customer {0}…"*.

**Saat credit limit terlanggar (SO/DN/SI):** `check_credit_limit()` menampilkan dialog
*"Credit Limit has been crossed for customer …"*. Bila user **tidak** punya role `credit_controller`,
blokir bersifat keras (`raise_exception=1`) dan user hanya ditawari mengirim email ke user ber-role
`Sales Master Manager` (fallback daftar penerima). Jadi menaikkan limit tetap wewenang user dengan
role tersebut.

> **Ringkas untuk UI:** tampilkan `credit_limit`, `bypass_credit_limit_check`, dan
> `overdue_billing_threshold` dari baris company yang relevan; sediakan aksi "ubah limit" =
> GET → PUT array penuh, dan tangani pesan error validasi server (§2.8.1) apa adanya.

### 2.9 Termin pembayaran & cicilan (jatuh tempo)

Cicilan pelanggan **bukan** field di Customer. Mekanismenya **tiga lapis**:

| Lapis | Doctype | Fungsi |
|---|---|---|
| 1. Komponen | `Payment Term` (master, autoname `field:payment_term_name`) | Definisi **satu** termin: porsi %, jatuh tempo, diskon. |
| 2. Paket | `Payment Terms Template` (autoname `field:template_name`) + child `terms` → `Payment Terms Template Detail` | Kumpulan termin; **total `invoice_portion` wajib 100%**. |
| 3. Eksekusi | Child table **`Payment Schedule`** di Sales Order / Sales Invoice | Baris cicilan hasil generate: `due_date` + `payment_amount` per termin. |

Customer hanya menyimpan **default**: field `payment_terms` (Link → `Payment Terms Template`).

`Payment Term` dan `Payment Terms Template` **bukan child table** → keduanya bisa ditulis langsung
lewat `/api/resource/...` (berbeda dari `Customer Credit Limit`, §2.8).

**Field `Payment Term`:**

| Field | Tipe | Keterangan |
|---|---|---|
| `payment_term_name` | Data | **Kunci/autoname** (unik), `allow_rename: 1`. |
| `invoice_portion` | Float | Porsi terhadap nilai invoice (%). |
| `due_date_based_on` | Select | `Day(s) after invoice date`, `Day(s) after the end of the invoice month`, `Month(s) after the end of the invoice month`. |
| `credit_days` / `credit_months` | Int | Dipakai sesuai `due_date_based_on` (default `0`). |
| `mode_of_payment` | Link → Mode of Payment | Opsional. |
| `discount_type`, `discount`, `discount_validity_based_on`, `discount_validity` | — | Diskon bila dibayar lebih awal (opsional). |
| `description` | Small Text | Muncul di baris `payment_schedule`. |

**Field `Payment Terms Template`:** `template_name` (autoname), `terms` (child, `reqd`),
`allocate_payment_based_on_payment_terms` (Check, default `0`).

**Field child `terms` (`Payment Terms Template Detail`):** `payment_term` (Link → Payment Term),
`invoice_portion` (**reqd**), `due_date_based_on` (**reqd**), `credit_days`, `credit_months`,
`mode_of_payment`, `description`, `discount_*`.

> **Validasi server saat simpan template** (`PaymentTermsTemplate.validate`):
> - Total `invoice_portion` semua baris **harus = 100** → *"Combined invoice portion must equal 100%"*.
> - Baris dengan kombinasi (`payment_term`, `credit_days`, `credit_months`, `due_date_based_on`) yang
>   sama dianggap duplikat → *"The Payment Term at row {0} is possibly a duplicate."*
> - Bila `allocate_payment_based_on_payment_terms = 1`, `payment_term` **wajib** di tiap baris →
>   *"Row {0}: Payment Term is mandatory"*.
> - `payment_term` hanya Link biasa: kalau diisi, master `Payment Term`-nya harus sudah ada (validasi
>   link). Bila ingin template berdiri sendiri tanpa master, **kosongkan** `payment_term`.

#### 2.9.1 Buat master termin — `POST /api/resource/Payment Term`

```bash
curl -X POST https://site-anda.com/api/resource/Payment%20Term \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "payment_term_name": "DP 30%",
    "invoice_portion": 30,
    "due_date_based_on": "Day(s) after invoice date",
    "credit_days": 0,
    "description": "Uang muka 30%"
  }'
```

**Contoh respons (HTTP 200):** `{ "data": { "name": "DP 30%", ... } }` — `name` = `payment_term_name`.

#### 2.9.2 Buat template 3 termin — `POST /api/resource/Payment Terms Template`

Skenario: **DP 30%** (jatuh tempo saat invoice), **Termin 2 40%** (+30 hari), **Termin 3 30%** (+60 hari).

```bash
curl -X POST https://site-anda.com/api/resource/Payment%20Terms%20Template \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "template_name": "3 Termin - 30/40/30",
    "allocate_payment_based_on_payment_terms": 0,
    "terms": [
      { "payment_term": "DP 30%",   "invoice_portion": 30, "due_date_based_on": "Day(s) after invoice date", "credit_days": 0,  "description": "DP 30%" },
      { "payment_term": "Termin 2", "invoice_portion": 40, "due_date_based_on": "Day(s) after invoice date", "credit_days": 30, "description": "Termin 2 (30 hari)" },
      { "payment_term": "Termin 3", "invoice_portion": 30, "due_date_based_on": "Day(s) after invoice date", "credit_days": 60, "description": "Termin 3 (60 hari)" }
    ]
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": {
    "name": "3 Termin - 30/40/30",
    "template_name": "3 Termin - 30/40/30",
    "allocate_payment_based_on_payment_terms": 0,
    "terms": [
      { "payment_term": "DP 30%", "invoice_portion": 30, "due_date_based_on": "Day(s) after invoice date", "credit_days": 0 },
      { "payment_term": "Termin 2", "invoice_portion": 40, "due_date_based_on": "Day(s) after invoice date", "credit_days": 30 },
      { "payment_term": "Termin 3", "invoice_portion": 30, "due_date_based_on": "Day(s) after invoice date", "credit_days": 60 }
    ]
  }
}
```

**Baca template untuk dropdown / pratinjau termin:**

```bash
# dropdown: cukup name + template_name
curl -G "https://site-anda.com/api/resource/Payment%20Terms%20Template" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","template_name"]' \
  --data-urlencode 'limit_page_length=0'

# detail termin (child `terms`)
curl -G "https://site-anda.com/api/resource/Payment%20Terms%20Template/3%20Termin%20-%2030%2F40%2F30" \
  -H 'Authorization: Bearer <access_token>'
```

#### 2.9.3 Set / ganti default termin di Customer

```bash
curl -X PUT "https://site-anda.com/api/resource/Customer/CUST-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "payment_terms": "3 Termin - 30/40/30" }'
```

**Rantai fallback** — `erpnext.accounts.party.get_payment_terms_template()` meresolusi berurutan:

1. `Customer.payment_terms`,
2. jika kosong → `Customer Group.payment_terms` (label: *Default Payment Terms Template*),
3. jika masih kosong → `Company.payment_terms`.

> **Catatan penting:** field ini hanya **default untuk transaksi baru**. Mengubah `payment_terms`
> **tidak** mengubah Sales Order / Sales Invoice yang sudah dibuat. Untuk SI yang dibuat dari SO,
> penarikan termin diatur *Accounts Settings → `automatically_fetch_payment_terms`*.

#### 2.9.4 Pratinjau jatuh tempo — `GET /api/method/erpnext.accounts.party.get_due_date`

Method whitelisted untuk menghitung jatuh tempo dari template — berguna untuk pratinjau di form Customer
(tanpa membuat dokumen).

```bash
curl -G "https://site-anda.com/api/method/erpnext.accounts.party.get_due_date" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'posting_date=2026-09-12' \
  --data-urlencode 'party_type=Customer' \
  --data-urlencode 'party=CUST-00001'
```

**Contoh respons (HTTP 200):**

```json
{ "message": "2026-11-11" }
```

Parameter: `posting_date` (wajib), `party_type`, `party` (wajib), `company`, `bill_date`,
`template_name` (opsional — bila dikosongkan, otomatis memakai default Customer → Group → Company).
Nilai balikan = jatuh tempo **terakhir** (maksimum dari seluruh termin template).

#### 2.9.5 Mengirim cicilan custom di Sales Order / Sales Invoice

`Sales Order` dan `Sales Invoice` punya field `payment_terms_template` (Link) dan `payment_schedule`
(Table → `Payment Schedule`). Field `payment_schedule` **boleh dikirim langsung** — penting karena
`set_payment_schedule()` hanya meng-generate baris dari template **jika `payment_schedule` masih kosong**.

**Field `Payment Schedule`:** `payment_term`, `description`, `due_date` (**reqd**), `invoice_portion`
(Percent), `payment_amount` (**reqd**), `mode_of_payment`, `outstanding` (read-only),
`paid_amount`, `base_payment_amount`, `discount_date`.

```bash
curl -X POST https://site-anda.com/api/resource/Sales%20Invoice \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "customer": "CUST-00001",
    "company": "PT Maju Jaya",
    "posting_date": "2026-09-12",
    "payment_terms_template": "3 Termin - 30/40/30",
    "payment_schedule": [
      { "payment_term": "DP 30%",   "description": "DP 30%",              "due_date": "2026-09-12", "invoice_portion": 30 },
      { "payment_term": "Termin 2", "description": "Termin 2 (30 hari)",  "due_date": "2026-10-12", "invoice_portion": 40 },
      { "payment_term": "Termin 3", "description": "Termin 3 (60 hari)",  "due_date": "2026-11-11", "invoice_portion": 30 }
    ]
  }'
```

> Kalau `invoice_portion` diisi, backend yang menghitung `payment_amount = grand_total * portion / 100`
> (begitu juga `base_payment_amount`). Kalau `invoice_portion` kosong, nilai `payment_amount` dikirim
> apa adanya.

> **`due_date` dokumen** di-set otomatis = **maksimum** `due_date` di `payment_schedule`
> (`set_due_date()`). Jangan mengandalkan `due_date` yang dikirim manual.

**Validasi server yang harus diantisipasi UI:**

| Kondisi | Perilaku |
|---|---|
| Σ `payment_amount` ≠ grand / rounded total (> 0.1) | **Gagal.** *"Total Payment Amount in Payment Schedule must be equal to Grand / Rounded Total"* |
| Dua baris dengan `due_date` sama | **Gagal.** *"Rows with duplicate due dates in other rows were found: …"* |
| Sales Order: `due_date` < `transaction_date` | **Gagal.** *"Row {0}: Due Date in the Payment Terms table cannot be before Posting Date"* |
| Sales Invoice: `due_date` < `posting_date` | **Gagal.** *"Due Date cannot be before Posting Date"* |
| SI punya `payment_terms_template` dan `due_date` (hasil maksimum schedule) > jatuh tempo template | **Gagal** *"Due Date cannot be after {0}"* — kecuali user punya role `credit_controller` (hanya peringatan *"Due Date exceeds allowed credit days by N day(s)"*) |

> Sales Invoice dengan `is_pos = 1` atau `is_return = 1` → `payment_terms_template` dan
> `payment_schedule` **dikosongkan** otomatis oleh backend.

#### 2.9.6 Menagih per termin — Payment Request

Agar pelanggan bisa **membayar per termin**, buat `Payment Request` yang menunjuk baris
`payment_schedule` terpilih (satu PR = satu cicilan).

```bash
curl -X POST https://site-anda.com/api/method/erpnext.accounts.doctype.payment_request.payment_request.make_payment_request \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "dt": "Sales Invoice",
    "dn": "SINV-00001",
    "party_type": "Customer",
    "party": "CUST-00001",
    "schedules": "[{\"name\":\"a1b2c3d4e5\",\"payment_term\":\"Termin 2\",\"description\":\"Termin 2 (30 hari)\",\"due_date\":\"2026-10-12\",\"payment_amount\":40000000}]",
    "submit_doc": 1,
    "mute_email": 1
  }'
```

**Cara paling praktis:** jalankan `GET /api/resource/Sales Invoice/SINV-00001` → ambil array
`payment_schedule` → pilih baris yang ingin ditagih → kirim array itu (apa adanya) sebagai parameter
`schedules` (JSON string). Backend memetakan tiap baris menjadi child `payment_reference`
(`payment_term`, `description`, `due_date`, `amount`, `payment_schedule` = `name` baris) dan
menjumlahkan `payment_amount` menjadi `grand_total` Payment Request.

| Parameter | Keterangan |
|---|---|
| `dt`, `dn` | Doctype & nama dokumen sumber. Diizinkan: `Sales Order`, `Sales Invoice`, `Purchase Order`, `Purchase Invoice`, `POS Invoice`, `Fees`. |
| `schedules` | JSON string baris `payment_schedule` terpilih (kunci yang dibaca: `name`, `payment_term`, `description`, `due_date`, `payment_amount`, `currency`). |
| `party_type`, `party` | Untuk pengambilan bank account pihak. |
| `submit_doc` | `1` = langsung submit (bila *Accounts Settings → `create_pr_in_draft_status`* aktif, dokumen dibuat draft dulu). |
| `mute_email` | `1` = jangan kirim email payment link. |
| `return_doc` | `1` = kembalikan dokumen (bukan dict). |

> **Batasan backend:** (a) PR berbasis schedule **ditolak** bila sudah ada Payment Entry pada dokumen itu
> (*"Payment Schedule based Payment Requests cannot be created because a Payment Entry already exists…"*);
> (b) baris schedule yang **sudah pernah** dijadikan PR akan ditolak
> (*"The following payment schedule(s) already exist: …"*); (c) bila sudah ada PR draft, baris baru
> di-append ke PR draft tersebut, bukan membuat PR baru.

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
