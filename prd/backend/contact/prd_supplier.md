# PRD — REST API Doctype Supplier (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Supplier** di ERPNext (Frappe),
> diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Buying (ERPNext)
- **Doctype:** `Supplier`
- **Versi API:** `/api/resource/...` (API v1)
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

---

## 1. Ruang lingkup

| # | Endpoint | Metode | Keterangan |
|---|---|---|---|
| 1 | `/api/resource/Supplier` | POST | Buat Supplier baru (CREATE) |
| 2 | `/api/resource/Supplier/{name}` | GET | Ambil detail 1 Supplier (READ) |
| 3 | `/api/resource/Supplier` | GET | Daftar Supplier (READ list) |
| 4 | `/api/resource/Supplier/{name}` | PUT | Ubah Supplier (UPDATE) |
| 5 | `/api/resource/Supplier/{name}` | PUT | Non-aktifkan Supplier (`disabled=1`) — pengganti DELETE |
| 6 | `/api/resource/Supplier Group` | GET | Daftar Supplier Group (dropdown UI) |
| 7 | `/api/resource/DocField` | GET | Opsi Supplier Type (dropdown UI) |
| 8 | `/api/method/frappe.core.doctype.data_export.exporter.export_data` | GET | Export Supplier ke Excel/CSV (§4.6) |
| 9 | `/api/method/frappe.client.get_count` | GET | Total record Supplier sesuai filter — untuk pagination / lazy loading (§4.2) |

> Sesuai keputusan, dokumen ini hanya membahas **data wajib terisi**.
> Field lain (kontak, alamat, akun per-company, pajak, dsb.) ditambahkan belakangan
> bila sudah ada keputusan penggunaan.

---

## 2. Ringkasan field & data wajib

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `supplier_name` | Data | Nama supplier (display name). `reqd: 1`. |
| 🔴 **WAJIB** | `supplier_type` | Select | `reqd: 1`, default `Company`. Opsi: `Company`, `Individual`, `Partnership`. |
| 🟠 **DISARANKAN** | `supplier_group` | Link → Supplier Group | **Tidak** `reqd` di doctype, tapi dipakai di UI & praktik standar. Hanya node daun (`is_group = 0`). |
| ⚪ **Otomatis — jangan dikirim** | `name`, `naming_series` | — | `name` dibangkitkan otomatis oleh *naming series* `SUP-.YYYY.-` → mis. `SUP-00001`. |
| ⚪ **Read-only — jangan dikirim** | `email_id`, `mobile_no`, `primary_address`, `address_html`, `contact_html` | — | Diisi otomatis dari kontak/alamat primer. |

### 2.1 Catatan penting

1. **`name` vs `supplier_name`** — `name` (kunci dokumen) otomatis menjadi `SUP-xxxxx` (naming series).
   `supplier_name` hanyalah judul tampilan. Operasi GET/PUT/DELETE memakai `name`.
   *(Jika pengaturan global `Supp Master Name` di-set ke "Supplier Name", maka `name = supplier_name`.)*
2. **Auto-create Contact & Address (hanya dalam kondisi tertentu)** — di `supplier.py`, `on_update()`
   memanggil `create_primary_contact()` / `create_primary_address()`, tetapi:
   - **Contact** hanya dibuat jika `supplier_primary_contact` masih kosong **dan** ada
     `mobile_no` / `email_id` terisi (lihat **Varian A** di §4.1).
   - **Address** hanya dibuat untuk dokumen **baru** (`is_new_doc`) **dan** ada `address_line1` terisi
     — dan `address_line1`/`city` **bukan field doctype Supplier**, sehingga lewat REST API jalur ini
     tidak bisa dipicu; di ERPNext ia berjalan lewat form *Quick Entry* di UI.
     Untuk mendaftarkan Address lewat API, gunakan **Varian B** di §4.1.
   - Karena payload minimum di dokumen ini (hanya `supplier_name`, `supplier_group`, `supplier_type`)
     tidak mengirim `email_id` / `mobile_no`, maka **tidak ada** Contact tambahan yang dibuat otomatis.
   - **Sebelum CREATE** & **saat UPDATE mengganti nama**, wajib **pre-check** nama — cek Contact &
     Customer (lihat **Langkah 0** §4.1 / **§4.4**).
   - **Saat DELETE**, semua `Contact` & `Address` yang ter-link ke Supplier ikut dihapus
     (`delete_contact_and_address`). Karena itu **jangan gunakan DELETE** — gunakan
     **non-aktifkan** (§4.5) agar data contact/address tidak hilang.
3. **Role yang dibutuhkan** — baca: `Purchase User` / `Accounts User`; tulis: `Purchase Manager`;
   buat/hapus: `Purchase Master Manager`.
4. **`supplier_group`** di form dibatasi ke node daun via link filter `[["Supplier Group","is_group","=",0]]`
   — gunakan hasil GET pada Bagian 5.

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Detail lengkap setup OAuth Client, alur mendapatkan/memperbarui/mencabut token ada di
**[`prd_oauth.md`](../prd_oauth.md)**.

---

## 4. CRUD — Doctype Supplier

### 4.1 CREATE — `POST /api/resource/Supplier`

**Langkah 0 — Pre-check (wajib sebelum POST)**

Karena `supplier_name` dipakai sebagai dasar nama auto-contact (Varian A) dan harus unik terhadap
master lain, UI wajib memastikan **dua hal** sebelum mengirim POST:

**a) Contact dengan nama yang sama belum ada:**

```bash
curl -G "https://site-anda.com/api/resource/Contact" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","status","company_name"]' \
  --data-urlencode 'filters=[["name","=","PT Maju Jaya"]]' \
  --data-urlencode 'limit_page_length=1'
```

**b) Customer dengan nama yang sama belum ada (nama tidak boleh dipakai ganda):**

```bash
curl -G "https://site-anda.com/api/resource/Customer" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","customer_name"]' \
  --data-urlencode 'filters=[["customer_name","=","PT Maju Jaya"]]' \
  --data-urlencode 'limit_page_length=1'
```

**Hasil & aturan:**
- Kedua `data` kosong (`[]`) → **lanjut** ke POST Supplier.
- Salah satu `data` terisi → **blokir POST** dan tampilkan pesan, mis.
  - Contact ada → *"Contact dengan nama {supplier_name} sudah ada. Gunakan contact tersebut / pilih nama lain."*
  - Customer ada → *"Nama {supplier_name} sudah dipakai sebagai Customer. Gunakan nama lain."*
  *(Alternatif cek Contact: `GET /api/resource/Contact/{supplier_name}` — HTTP 404 = belum ada.)*

> **Field yang dicek:** Contact → `name` (nama lengkap/company, sama dengan `supplier_name` saat
> auto-contact). Customer → `customer_name` (nama tampilan), **bukan** `name` (ID otomatis
> `CUST-` yang belum diketahui sebelum POST).

> **Catatan status "aktif":** field `status` di doctype Contact hanya berisi
> `Passive` / `Open` / `Replied` — **tidak ada opsi "Active"**. Status informasional; rekomendasi:
> blokir berdasarkan keberadaan nama saja. Jika tetap ingin filter "aktif", petakan sendiri
> (mis. `status == "Open"`) atau tambah *custom field* `is_active` di Contact (butuh kustomisasi
> backend).

> **Catatan backend:** meski tanpa pre-check, ERPNext melempar `DuplicateEntryError` saat
> auto-create Contact menemukan nama yang sama (POST Supplier gagal). Pre-check memberi pesan
> ramah & lebih cepat, sekaligus mencegah nama ganda di Customer.

**Payload minimum (data wajib):**

```json
{
  "supplier_name": "PT Maju Jaya",
  "supplier_group": "Local",
  "supplier_type": "Company"
}
```

**Contoh request:**

```bash
curl -X POST https://site-anda.com/api/resource/Supplier \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "supplier_name": "PT Maju Jaya",
    "supplier_group": "Local",
    "supplier_type": "Company"
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "data": {
    "name": "SUP-00001",
    "owner": "Administrator",
    "creation": "2026-08-13 09:15:00.000000",
    "modified": "2026-08-13 09:15:00.000000",
    "modified_by": "Administrator",
    "docstatus": 0,
    "idx": 0,
    "naming_series": "SUP-.YYYY.-",
    "supplier_name": "PT Maju Jaya",
    "supplier_group": "Local",
    "supplier_type": "Company",
    "disabled": 0,
    "is_frozen": 0,
    "on_hold": 0,
    "hold_type": "All",
    "accounts": []
  }
}
```

> `name` = `SUP-00001` → simpan nilai ini; dipakai untuk GET/PUT/DELETE berikutnya.

**Varian A — CREATE + auto-generate Contact (opsional)**

Jika body CREATE juga berisi `email_id` dan/atau `mobile_no`, ERPNext otomatis membuat
**Contact** primer yang ter-link ke Supplier, bernama sesuai `supplier_name`:

| `supplier_type` | Contact yang dibuat otomatis |
|---|---|
| `Company` / `Partnership` | `company_name = supplier_name` (tanpa first/last name) |
| `Individual` | `first/middle/last_name` dipecah dari `supplier_name` |

> Contact ini hanya *placeholder* bernama supplier — **bukan** "orang kontak" sungguhan.
> Untuk orang kontak yang spesifik (mis. "Budi Santoso"), buat Contact secara eksplisit.

> **Pre-check:** Langkah 0 di atas wajib dijalankan sebelum POST — auto-contact Varian A akan
> bernama persis `supplier_name`, dan aturan cek Contact + Customer tetap berlaku di sini.

**Contoh request (Varian A):**

```bash
curl -X POST https://site-anda.com/api/resource/Supplier \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "supplier_name": "PT Maju Jaya",
    "supplier_group": "Local",
    "supplier_type": "Company",
    "email_id": "info@majujaya.co.id",
    "mobile_no": "+6281234567890"
  }'
```

**Apa yang terjadi:** selain Supplier tersimpan, ERPNext otomatis membuat `Contact` dengan
`company_name = "PT Maju Jaya"`, `email_ids[].email_id`, `phone_nos[].phone`,
`is_primary_contact = 1`, dan `links` → Supplier. Setelah itu `supplier_primary_contact`,
`email_id`, `mobile_no` pada Supplier diisi dari contact tersebut.

**Cara memverifikasi Contact yang dibuat:**

```bash
curl -X GET "https://site-anda.com/api/resource/Contact/PT%20Maju%20Jaya" \
  -H 'Authorization: Bearer <access_token>'
```

Respons `data` berisi Contact dengan `company_name = "PT Maju Jaya"` serta child table
`email_ids` dan `phone_nos` terisi.

> Catatan: `email_id` & `mobile_no` di Supplier bersifat **read-only** (di-fetch dari contact
> primer). Pengiriman lewat CREATE hanya berfungsi sebagai **pemicu auto-create Contact**.

> **Primary otomatis:** karena frontend tidak mengisi daftar contact person, contact atas nama
> supplier itu sendiri (mis. "PT Maju Jaya") yang dibuat otomatis dan **langsung dijadikan
> `supplier_primary_contact`**. Frontend tidak perlu melakukan apa pun lagi.

> **Catatan `status` pada auto-create Contact (penting):** Contact yang dibuat otomatis oleh
> Supplier ini **tidak bisa diatur status-nya dari frontend** — diisi oleh backend, dan secara
> default ERPNext mengisinya `"Passive"`. Jika syaratnya semua contact baru (termasuk yang dibuat
> otomatis) harus berstatus `"Open"`, perlu perubahan di sisi backend/admin: ubah default field
> `status` doctype Contact menjadi `"Open"` (Customize Form → Contact → Status → Default), atau
> pasang *server script*/hook `doc_events` yang mengisi `status="Open"` saat create. Tanpa
> perubahan itu, contact auto-create tetap `"Passive"`.

**Varian B — CREATE Supplier + daftarkan Address (cara andal lewat API)**

Karena doctype `Supplier` **tidak punya field alamat** (`address_line1`/`city` bukan field
Supplier), auto-create Address dari payload Supplier **tidak bisa dipicu lewat REST API**
(auto-create Address hanya terjadi lewat form *Quick Entry* di UI). Cara yang andal untuk UI
adalah membuat Address secara eksplisit lalu menautkannya ke Supplier.

**Langkah 1 — Buat Supplier** (payload minimum di atas). Simpan `name` hasilnya, mis. `SUP-00001`.

**Langkah 2 — Buat Address yang ter-link ke Supplier:**

```bash
curl -X POST https://site-anda.com/api/resource/Address \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "address_title": "PT Maju Jaya",
    "address_type": "Billing",
    "address_line1": "Jl. Sudirman No. 123",
    "address_line2": "Gedung B, Lt. 5",
    "city": "Jakarta",
    "state": "DKI Jakarta",
    "pincode": "10220",
    "country": "Indonesia",
    "is_primary_address": 1,
    "links": [
      { "link_doctype": "Supplier", "link_name": "SUP-00001" }
    ]
  }'
```

Field **wajib** di doctype `Address`: `address_type`, `address_line1`, `city`, `country`
(semuanya `reqd: 1`). `address_title` baru wajib bila `links` kosong — karena kita mengirim
`links`, `address_title` opsional (tetap disarankan diisi). Simpan `name` Address — aturan
**autoname**: `name = "{address_title}-{address_type}"`, mis. `PT Maju Jaya-Billing` (jika
duplikat → `PT Maju Jaya-Billing-1`, dst.).

**Langkah 3 — Jadikan Address itu primary di Supplier:**

```bash
curl -X PUT "https://site-anda.com/api/resource/Supplier/SUP-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "supplier_primary_address": "PT Maju Jaya-Billing" }'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": {
    "name": "SUP-00001",
    "supplier_name": "PT Maju Jaya",
    "supplier_group": "Local",
    "supplier_type": "Company",
    "supplier_primary_address": "PT Maju Jaya-Billing"
  }
}
```

> `links` pada Address menghubungkannya ke Supplier, sehingga alamat muncul di tab
> "Address" dan tab "Contact" Supplier, bisa dipilih sebagai primary, dan dipakai pada transaksi
> (PO / Purchase Invoice). Saat Supplier dihapus, Address ter-link ikut terhapus.

**Varian C — Menambahkan contact person yang sudah ada & set primary**

**Kasus:** Supplier `SUP-00001` (PT Maju Jaya) memakai **Markus** dan **Tommy** — dua Contact
 yang **sudah ada** di `tabContact` — sebagai contact person; **Markus** dijadikan primary.

**Catatan konsep:**
- Menjadikan seseorang contact person = Contact tsb ter-link ke Supplier (child table `links`).
- Menetapkan primary = field **`supplier_primary_contact`** di **Supplier** (bukan di Contact).
- Karena contact sudah ada, `supplier_primary_contact` bisa **langsung diisi saat CREATE**
  (auto-create Contact dari Varian A akan dilewati karena field ini sudah terisi).

**Langkah 1 — CREATE Supplier, langsung set primary contact (Markus):**

```bash
curl -X POST https://site-anda.com/api/resource/Supplier \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "supplier_name": "PT Maju Jaya",
    "supplier_group": "Local",
    "supplier_type": "Company",
    "supplier_primary_contact": "Markus"
  }'
```

> `supplier_primary_contact` hanya valid bila Contact **"Markus" sudah ada** (Link field divalidasi
> saat save — kalau belum ada akan error link).

**Langkah 2 — (UPDATE) mengganti primary di kemudian hari:**

```bash
curl -X PUT "https://site-anda.com/api/resource/Supplier/SUP-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "supplier_primary_contact": "Tommy" }'
```

**Menautkan contact yang sudah ada (jika belum ter-link):** Markus & Tommy baru dianggap
"contact person" bila `links`-nya mengarah ke Supplier. Bila belum, update Contact-nya (PUT)
dengan **semua** `links` (lama + baru) karena `links` bersifat **replace-all**:

```bash
curl -X PUT "https://site-anda.com/api/resource/Contact/Markus" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "links": [
      { "link_doctype": "Supplier", "link_name": "SUP-00001" }
    ]
  }'
```

**Hasil:** tab "Address" dan tab "Contact" Supplier menampilkan Markus & Tommy; `email_id`/`mobile_no`
Supplier ter-fetch dari Markus; transaksi (PO/PI) memakai Markus sebagai kontak default.

### 4.2 READ (satu record) — alur lengkap

**Langkah 1 — Ambil detail Supplier:**

```bash
curl -X GET "https://site-anda.com/api/resource/Supplier/SUP-00001" \
  -H 'Authorization: Bearer <access_token>'
```

Respons `data` berisi seluruh field Supplier (struktur seperti respons CREATE), termasuk
`sales_partner` `supplier_primary_contact` / `supplier_primary_address` (hanya nama link)
dan `email_id`/`mobile_no` (dari contact primer).

> **Catatan:** `data` **tidak** menyertakan daftar lengkap address/contact person yang ter-link
> (daftar itu hanya diisi lewat `onload` pada form Desk, bukan REST API). Ambil terpisah seperti
> langkah 2 & 3 di bawah.

**Langkah 2 — Ambil daftar contact person milik Supplier (filter Dynamic Link):**

```bash
curl -G "https://site-anda.com/api/resource/Contact" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","full_name","email_id","mobile_no","is_primary_contact"]' \
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
      "mobile_no": "+6281234567890",
      "is_primary_contact": 1
    }
  ]
}
```

**Langkah 3 — Ambil daftar address milik Supplier (filter Dynamic Link):**

```bash
curl -G "https://site-anda.com/api/resource/Address" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","address_title","address_type","address_line1","city","country","is_primary_address"]' \
  --data-urlencode 'filters=[[["Dynamic Link","link_doctype","=","Supplier"],["Dynamic Link","link_name","=","SUP-00001"]]]' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "PT Maju Jaya-Billing",
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

> Versi lengkap untuk Customer didokumentasikan terpisah di
> **[prd_customer.md §2.2](./prd_customer.md)** (alur yang sama, `link_doctype = Customer`).

**Total record count — `GET /api/method/frappe.client.get_count`**

Untuk kebutuhan pagination / lazy loading (mis. menampilkan "900 dari 1000 Supplier"), ambil
**total record yang cocok dengan filter** lewat method whitelisted `get_count`. Respons list
(`GET /api/resource/Supplier`) **tidak** menyertakan total — hitung terpisah dengan endpoint ini.

```bash
curl -G "https://site-anda.com/api/method/frappe.client.get_count" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Supplier' \
  --data-urlencode 'filters=[["disabled","=",0]]'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": 900
}
```

> Parameter: `doctype` (wajib), `filters` (opsional, format sama seperti READ list).
> Respons berupa `message` (bukan `data`) = jumlah record yang cocok. Dari 1000 Supplier,
> filter `disabled=0` mengembalikan 900; tanpa filter → 1000. Count menghormati filter
> **permission user** (angka sesuai hak akses user) dan akurat selama tidak mengirim param
> `limit`. Nilai ini dipakai untuk menghitung total halaman saat lazy loading di §4.3.

### 4.3 READ (daftar) — `GET /api/resource/Supplier`

Parameter pilihan: `fields`, `filters`, `order_by`, `limit_start`, `limit_page_length`.

> **Lazy loading / pagination:** `limit_page_length` = jumlah record per halaman,
> `limit_start` = index awal (offset). Contoh ambil **per 50 data**:

```bash
# Halaman 1 — record 1–50
curl -G "https://site-anda.com/api/resource/Supplier" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","supplier_name","supplier_group","supplier_type","disabled"]' \
  --data-urlencode 'filters=[["supplier_group","=","Local"]]' \
  --data-urlencode 'limit_start=0' \
  --data-urlencode 'limit_page_length=50'
```

> Halaman berikutnya naikkan `limit_start` kelipatan 50 (`50`, `100`, dst.) dengan
> `limit_page_length=50` tetap. Jumlah halaman = `total / 50`, dengan `total` diperoleh
> dari `get_count` (§4.2). `limit_page_length=0` = ambil **semua** record (tanpa LIMIT),
> seperti contoh di bawah ini.

```bash
curl -G "https://site-anda.com/api/resource/Supplier" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","supplier_name","supplier_group","supplier_type","disabled"]' \
  --data-urlencode 'filters=[["supplier_group","=","Local"]]' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "SUP-00001",
      "supplier_name": "PT Maju Jaya",
      "supplier_group": "Local",
      "supplier_type": "Company",
      "disabled": 0
    }
  ]
}
```

### 4.4 UPDATE — `PUT /api/resource/Supplier/{name}`

Kirim **hanya field yang diubah**.

**Langkah 0 — Pre-check nama (hanya saat `supplier_name` diganti)**

Aturan nama ganda (lihat §4.1 Langkah 0) tetap berlaku saat UPDATE, dengan **self-exclusion**:

- Jalankan cek **hanya jika `supplier_name` berubah** (bukan setiap PUT).
- **Abaikan milik sendiri:** Contact primer milik Supplier ini dan Supplier itu sendiri tidak
  dianggap konflik (supaya PUT tanpa perubahan nama tidak diblokir).

```bash
# (a) Contact dengan nama baru (selain milik Supplier ini) belum ada
curl -G "https://site-anda.com/api/resource/Contact" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","company_name"]' \
  --data-urlencode 'filters=[["name","=","PT Maju Jaya Tbk"]]' \
  --data-urlencode 'limit_page_length=1'

# (b) Customer dengan nama baru belum ada
curl -G "https://site-anda.com/api/resource/Customer" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","customer_name"]' \
  --data-urlencode 'filters=[["customer_name","=","PT Maju Jaya Tbk"]]' \
  --data-urlencode 'limit_page_length=1'
```

**Aturan:** jika salah satu `data` terisi (selain milik Supplier ini) → **blokir PUT** dan tampilkan
pesan, mis. *"Nama {supplier_name} sudah dipakai. Gunakan nama lain."*

> **Catatan backend:** rename `supplier_name` **tidak** memicu auto-create Contact (karena
> `supplier_primary_contact` sudah terisi) sehingga backend **tidak** memblokir konflik nama — cek
> ini wajib di sisi UI. Pengecualian: jika Supplier dibuat tanpa `email_id`/`mobile_no` (primary
> kosong) lalu di UPDATE ditambahkan, auto-contact akan dibuat dengan nama baru — cek (a) melindungi
> dari `DuplicateEntryError`. Rename juga **tidak mengubah nama** auto-contact lama (nama contact
> tetap nama lama).

```bash
curl -X PUT "https://site-anda.com/api/resource/Supplier/SUP-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "supplier_name": "PT Maju Jaya Tbk",
    "supplier_group": "Distributor"
  }'
```

**Contoh respons (HTTP 200):** objek `data` berisi dokumen terbaru (field berubah).

### 4.5 Non-aktifkan (disarankan) — `PUT /api/resource/Supplier/{name}`

**Aturan:** data Supplier **tidak boleh dihapus**, hanya dinon-aktifkan. Non-aktifkan dengan
mengirim field `disabled`:

```bash
curl -X PUT "https://site-anda.com/api/resource/Supplier/SUP-00001" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "disabled": 1 }'
```

**Contoh respons (HTTP 200):** objek `data` terbaru dengan `"disabled": 1`.

Untuk mengaktifkan kembali: `PUT` dengan `{ "disabled": 0 }`.

> **Efek non-aktif:** Supplier tidak muncul sebagai pilihan pada transaksi baru (PO / Purchase
> Invoice). **Contact & Address yang ter-link TIDAK dihapus** — email/telepon, `links`, dan
> `supplier_primary_contact`/`supplier_primary_address` tetap utuh.

> ⚠️ **Jangan gunakan `DELETE /api/resource/Supplier/{name}`** — ERPNext tetap mendukungnya, tapi
> saat DELETE semua Contact & Address ter-link ikut terhapus (`delete_contact_and_address`) dan
> gagal bila Supplier sudah dipakai transaksi (*"linked with"*). Karena aturan master data
> "jangan hapus", endpoint DELETE tidak dipakai di aplikasi.

### 4.6 Export ke Excel / CSV — `GET /api/method/frappe.core.doctype.data_export.exporter.export_data`

ERPNext mendukung export Supplier ke file **Excel (.xlsx)** dan **CSV** lewat REST API tanpa
kustomisasi — method `export_data` sudah *whitelisted* dan langsung mengembalikan file (tidak perlu Desk).

**Export Excel (.xlsx):**

```bash
curl -G "https://site-anda.com/api/method/frappe.core.doctype.data_export.exporter.export_data" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Supplier' \
  --data-urlencode 'file_type=Excel' \
  --data-urlencode 'with_data=1' \
  --data-urlencode 'select_columns={"Supplier":["name","supplier_name","supplier_group","supplier_type","disabled"]}' \
  --data-urlencode 'filters=[["supplier_group","=","Local"]]' \
  -o supplier.xlsx
```

**Export CSV:**

```bash
curl -G "https://site-anda.com/api/method/frappe.core.doctype.data_export.exporter.export_data" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Supplier' \
  --data-urlencode 'file_type=CSV' \
  --data-urlencode 'with_data=1' \
  -o supplier.csv
```

**Contoh respons:** file biner (attachment download) — header `Content-Disposition: attachment;
filename="Supplier.xlsx"` (atau `.csv`). Frontend cukup `saveAs` dari respons.

**Parameter (query string):**

| Parameter | Wajib | Keterangan |
|---|---|---|
| `doctype` | ✅ | Nama doctype (`Supplier`). |
| `file_type` | 🟠 | `Excel` atau `CSV` (default `CSV`). |
| `with_data` | 🟠 | `1` = ikut data (default `0` = hanya template kolom). |
| `select_columns` | 🟠 | JSON `{"Supplier":["field1","field2",...]}`. Kosong → semua kolom yang bisa diexport. |
| `filters` | 🟠 | Filter seperti READ list, mis. `[["supplier_group","=","Local"]]`. |
| `all_doctypes` | 🟠 | Default `1` (ikut child table bila ada). |

**Catatan penting:**
- **Izin export:** method mengecek `can_export(doctype)` — role harus punya hak **Export** pada
  doctype Supplier (default: `Purchase User` / `Purchase Manager`).
- **Alternatif template/import:** `frappe.core.doctype.data_import.data_import.download_template`
  (whitelisted) — mendukung `export_records=all` / `5_records` / `blank_template` dan
  `file_type=Excel|CSV`.
- **Alternatif frontend:** `GET /api/resource/Supplier?fields=...&limit_page_length=0` → JSON →
  generate `.xlsx`/`.csv` di klien (SheetJS/exceljs). Tetap valid, tanpa endpoint khusus.
- **Postman:** request `1.14` di folder `1. Supplier` (lihat §7).

### 4.7 Import masal (bulk) — CSV / XLSX

ERPNext mendukung **import masal** Supplier (insert / update / upsert) dari file **CSV / XLSX**
lewat REST API — memakai modul **Data Import** (tidak perlu Desk). Alurnya 3 langkah:

**Langkah 1 — Upload file (CSV/XLSX) via REST:**

```bash
curl -X POST "https://site-anda.com/api/method/upload_file" \
  -H 'Authorization: Bearer <access_token>' \
  -F 'file=@supplier_import.csv' \
  -F 'is_private=1'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": {
    "name": "abc123",
    "file_name": "supplier_import.csv",
    "file_url": "/private/files/supplier_import.csv"
  }
}
```

> Simpan `message.file_url` — dipakai di Langkah 2. Parameter opsional lain: `doctype`,
> `docname`, `folder`.

**Langkah 2 — Buat dokumen `Data Import`:**

```bash
curl -X POST https://site-anda.com/api/resource/Data%20Import \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "reference_doctype": "Supplier",
    "import_type": "Insert New Records",
    "import_file": "/private/files/supplier_import.csv",
    "mute_emails": 1,
    "status": "Pending"
  }'
```

> `import_type` (Literal): `Insert New Records` / `Update Existing Records` /
> `Insert or Update Records` (upsert). `import_file` = `file_url` dari Langkah 1.

**Langkah 3 — Jalankan import:**

```bash
curl -X POST "https://site-anda.com/api/method/frappe.core.doctype.data_import.data_import.form_start_import" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "data_import": "SUP-IMPORT-00001" }'
```

> Ganti `SUP-IMPORT-00001` dengan `name` Data Import dari Langkah 2.

**Pantau hasil (opsional):**

```bash
curl -X POST "https://site-anda.com/api/method/frappe.core.doctype.data_import.data_import.get_import_status" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "data_import_name": "SUP-IMPORT-00001" }'

curl -X POST "https://site-anda.com/api/method/frappe.core.doctype.data_import.data_import.get_import_logs" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "data_import": "SUP-IMPORT-00001" }'
```

**Catatan penting:**
- **Izin:** role harus punya hak **Import** pada doctype Supplier, dan doctype harus mengizinkan
  import (`allow_import = 1` — default aktif untuk Supplier/Customer/Contact/Address). Tanpa itu,
  Data Import ditolak (`PermissionError` / *"Data Import is not allowed"*).
- **Template:** gunakan output **Export** (langkah `download_template` / §4.6) sebagai format
  kolom template import. Kolom `name` boleh dikosongkan untuk Insert (nama dibangkitkan), atau
  diisi untuk Update/Upsert.
- **Relasi & child table:** Supplier bisa di-import bersama child (mis. `accounts`); Contact
  dengan child `email_ids` / `phone_nos` / `links`; Address dengan child `links` (kolom bernama
  `links.link_doctype`, `links.link_name`, dst.).
- **Status import:** `Pending` → `Success` / `Partial Success` / `Error` / `Timed Out`. Log per
  baris ada di `Data Import Log` (via `get_import_logs`).
- **Postman:** request `1.15` di folder `1. Supplier` (lihat §7).

---

## 5. GET pendukung UI

### 5.1 GET Supplier Group — `GET /api/resource/Supplier Group`

```bash
curl -G "https://site-anda.com/api/resource/Supplier%20Group" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","supplier_group_name","parent_supplier_group","is_group"]' \
  --data-urlencode 'filters=[["is_group","=",0]]' \
  --data-urlencode 'order_by=supplier_group_name asc' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "name": "Distributor",
      "supplier_group_name": "Distributor",
      "parent_supplier_group": "All Supplier Groups",
      "is_group": 0
    },
    {
      "name": "Local",
      "supplier_group_name": "Local",
      "parent_supplier_group": "All Supplier Groups",
      "is_group": 0
    },
    {
      "name": "Services",
      "supplier_group_name": "Services",
      "parent_supplier_group": "All Supplier Groups",
      "is_group": 0
    }
  ]
}
```

> Untuk dropdown Supplier Group di form Supplier: **hanya** ambil yang `is_group = 0` (node daun).

### 5.2 GET Supplier Type — opsi via metadata `DocField`

`supplier_type` **bukan doctype** (di ERPNext modern, `Supplier Type` sudah diganti menjadi
`Supplier Group`). Field `supplier_type` di Supplier hanyalah **Select** dengan 3 opsi tetap.
Cara mengambil opsi-opsi tersebut lewat API:

```bash
curl -G "https://site-anda.com/api/resource/DocField" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["fieldname","fieldtype","label","options"]' \
  --data-urlencode 'filters=[["parent","=","Supplier"],["fieldname","=","supplier_type"]]'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "fieldname": "supplier_type",
      "fieldtype": "Select",
      "label": "Supplier Type",
      "options": "Company\nIndividual\nPartnership"
    }
  ]
}
```

**Cara pakai di UI:** pecah `options` dengan karakter baris baru (`\n`) →
`["Company", "Individual", "Partnership"]`, lalu render sebagai dropdown.
*(Alternatif: `GET /api/resource/DocType/Supplier` lalu baca `fields[]` dengan
`fieldname == "supplier_type"` — payload lebih besar. Jika role user tidak punya akses baca
`DocField`, fallback-nya adalah hardcode 3 opsi di frontend.)*

### 5.3 CREATE Supplier Group — `POST /api/resource/Supplier Group`

`Supplier Group` adalah **master konfigurasi & tree**. `name` = `supplier_group_name`
(`autoname: field:supplier_group_name`).

**Payload minimum:**

```json
{
  "supplier_group_name": "Distributor",
  "parent_supplier_group": "All Supplier Groups",
  "is_group": 0
}
```

**Contoh request:**

```bash
curl -X POST https://site-anda.com/api/resource/Supplier%20Group \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "supplier_group_name": "Distributor",
    "parent_supplier_group": "All Supplier Groups",
    "is_group": 0
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "data": {
    "name": "Distributor",
    "supplier_group_name": "Distributor",
    "parent_supplier_group": "All Supplier Groups",
    "is_group": 0
  }
}
```

> `name` = `Distributor` → simpan nilai ini. `parent_supplier_group` opsional (bila diisi harus
> node `is_group = 1`). `is_group = 1` = node parent (bisa punya child); `is_group = 0` = node
> daun (dipakai transaksi). Role buat: `Purchase Master Manager`.

### 5.4 UPDATE Supplier Group — `PUT /api/resource/Supplier Group/{name}`

Kirim **hanya field yang diubah** (mis. pindah parent / set `is_group` / `payment_terms`):

```bash
curl -X PUT "https://site-anda.com/api/resource/Supplier%20Group/Distributor" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "parent_supplier_group": "All Supplier Groups",
    "payment_terms": ""
  }'
```

**Contoh respons (HTTP 200):** objek `data` berisi dokumen terbaru.

> **Catatan nama:** `name` mengikuti `supplier_group_name` (autoname by fieldname — hanya berlaku
> saat insert). Mengganti `supplier_group_name` pada dokumen yang sudah ada **tidak** otomatis
> mengganti `name` — hindari rename via REST.

### 5.5 DELETE Supplier Group — `DELETE /api/resource/Supplier Group/{name}`

⚠️ **Supplier Group TIDAK punya field `disabled`** — jadi "non-aktif" native tidak tersedia
(berbeda dari Supplier/Customer/Address). Opsi:

- **Non-aktif (disarankan):** pindahkan Supplier yang memakai group ini ke group lain
  (`PUT /api/resource/Supplier/{name}` → ganti `supplier_group`), lalu berhenti memakai group lama.
- **DELETE** (hanya bila benar-benar yakin):

```bash
curl -X DELETE "https://site-anda.com/api/resource/Supplier%20Group/Distributor" \
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
> - Group yang **masih dipakai Supplier** (`supplier_group` merujuk ke sini) **tidak bisa** dihapus
>   — error *"linked with"*.
> - Role hapus: `Purchase Master Manager`.

---

## 6. Penanganan error umum

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | Role tidak punya akses | `{"message": "Not permitted"}` |
| 404 | Resource tidak ditemukan | `{"exc_type":"DoesNotExistError","message":"Resource Not Found"}` |
| 417 | Field wajib kosong (`reqd`) | `{"exc_type":"MandatoryError","message":"supplier_name is mandatory"}` |
| 417 | Validasi gagal (mis. duplicate, linked) | `{"exc_type":"ValidationError","message":"..."}` |

---

## 7. Koleksi Postman

Seluruh pemanggilan di atas tersedia dalam **satu koleksi Postman** yang siap import:
`docs/postman/postman_erpnext_api.json` (Collection v2.1). Koleksi ini berisi seluruh modul
API ERPNext (OAuth 2.0 + Supplier + Customer + Contact + Address) dan siap ditambah modul lain (Buying, Selling, dst.).
