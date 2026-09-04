# PRD — REST API Doctype Address (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Address** di ERPNext (Frappe),
> diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** CRM (ERPNext)
- **Doctype:** `Address`
- **Versi API:** `/api/resource/...` (API v1)
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

---

Dokumentasi REST API untuk doctype **Address**, mengikuti pola yang sama dengan
[Supplier](./prd_supplier.md), [Customer](./prd_customer.md), dan [Contact](./prd_contact.md).
Autentikasi (OAuth 2.0) dan penanganan error umum berlaku sama (lihat
[prd_supplier.md §3](./prd_supplier.md) / [`prd_oauth.md`](../prd_oauth.md) dan
[prd_supplier.md §6](./prd_supplier.md)).

> **Peran Address:** Address adalah tempat **kanonik** untuk lokasi fisik (alamat) milik pihak
> (Supplier/Customer). Address dihubungkan ke party lewat child table **`links`** (Dynamic Link)
> — itulah yang membuat alamat muncul di tab "Address" party. Satu Address bisa ter-link ke
> banyak party sekaligus.

### Ruang lingkup — Address

| # | Endpoint | Metode | Keterangan |
|---|---|---|---|
| 1 | `/api/resource/Address` | POST | Buat Address baru (CREATE) |
| 2 | `/api/resource/Address/{name}` | GET | Ambil detail 1 Address (READ) |
| 3 | `/api/resource/Address` | GET | Daftar Address (READ list) |
| 4 | `/api/resource/Address/{name}` | PUT | Ubah Address (UPDATE) |
| 5 | `/api/resource/Address/{name}` | PUT | Non-aktifkan Address (`disabled=1`) — pengganti DELETE |
| 6 | `/api/resource/Country` | GET | Daftar Country (dropdown UI) |
| 7 | `/api/resource/DocField` | GET | Opsi Address Type (dropdown UI) |
| 8 | `/api/method/frappe.core.doctype.data_export.exporter.export_data` | GET | Export Address ke Excel/CSV (§2.6) |
| 9 | `/api/method/frappe.client.get_count` | GET | Total record Address sesuai filter — untuk pagination / lazy loading (§2.2) |

---

## 1. Ringkasan field & data wajib — Address

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `address_type` | Select | `reqd: 1`. Opsi: `Billing`, `Shipping`, `Office`, `Personal`, `Plant`, `Postal`, `Shop`, `Subsidiary`, `Warehouse`, `Current`, `Permanent`, `Other`. |
| 🔴 **WAJIB** | `address_line1` | Data | `reqd: 1`. |
| 🔴 **WAJIB** | `city` | Data | `reqd: 1`. |
| 🔴 **WAJIB** | `country` | Link → Country | `reqd: 1`. |
| 🟠 **DISARANKAN** | `links[]` | Table → Dynamic Link | Menautkan Address ke party (Supplier/Customer). `link_doctype` & `link_name` keduanya `reqd: 1`. |
| 🟠 **DISARANKAN** | `address_title` | Data | **Wajib hanya jika `links` kosong** (`mandatory_depends_on`). Jika `links` ada & title kosong, terisi otomatis dari `link_name`. |
| 🟢 Opsional | `address_line2`, `county`, `state`, `pincode`, `email_id`, `phone`, `fax`, `is_primary_address`, `is_shipping_address`, `disabled` | — | sesuai kebutuhan |
| ⚪ **Otomatis — jangan dikirim** | `name`, `display` | — | `name` dibentuk `autoname()`; `display` dihitung controller saat read. |

### 1.1 Catatan penting — Address

1. **Aturan `name` (penting):** `autoname()` membentuk **`name = "{address_title}-{address_type}"`**
   — mis. `PT Maju Jaya-Billing`. Jika nama itu sudah ada, sistem menambah akhiran `-#` →
   `PT Maju Jaya-Billing-1`, `-2`, dst. **Bukan** seri seperti `SUP-`/`CUST-`. Karena nama bisa
   mengandung spasi, gunakan URL-encode (`%20`) pada GET/PUT/DELETE.
2. **Relasi ke party:** Address ter-link ke party lewat `links[]`. Menghapus baris link = alamat
   tidak lagi muncul di tab "Address" party (tapi data Address tetap ada, tidak terhapus).
3. **`is_primary_address` / `is_shipping_address`:** penanda preferensi party. Hanya satu Address
   per party yang bisa jadi primary — controller `validate_preferred_address()` me-reset flag di
   address lain yang sama party-nya.
4. **`disabled`:** Address **punya field `disabled`** (Check). Non-aktifkan membuat Address
   **tidak terpilih** sebagai default billing/shipping (`get_default_address()` memfilter
   `disabled=0`).
5. **Role:** baca/tulis luas — `Sales User`, `Purchase User`, `Accounts User`, `Maintenance User`;
   hapus: `System Manager`.
6. **Saat party di-DELETE**, Address ter-link ikut terhapus (`delete_contact_and_address`) —
   karena itu jangan DELETE party (lihat [prd_supplier.md §4.5](./prd_supplier.md) /
   [prd_customer.md §2.5](./prd_customer.md)).

---

## 2. CRUD — Doctype Address

### 2.1 CREATE — `POST /api/resource/Address`

**Payload minimum (data wajib + tautan party):**

```json
{
  "address_title": "PT Maju Jaya",
  "address_type": "Billing",
  "address_line1": "Jl. Sudirman No. 123",
  "city": "Jakarta",
  "country": "Indonesia",
  "is_primary_address": 1,
  "links": [
    { "link_doctype": "Supplier", "link_name": "SUP-00001" }
  ]
}
```

**Contoh request:**

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
      { "link_doctype": "Supplier", "link_name": "SUP-00001" }
    ]
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "data": {
    "name": "PT Maju Jaya-Billing",
    "owner": "Administrator",
    "address_title": "PT Maju Jaya",
    "address_type": "Billing",
    "address_line1": "Jl. Sudirman No. 123",
    "city": "Jakarta",
    "country": "Indonesia",
    "is_primary_address": 1,
    "disabled": 0,
    "links": [
      {
        "name": "abc123",
        "link_doctype": "Supplier",
        "link_name": "SUP-00001",
        "parent": "PT Maju Jaya-Billing",
        "parentfield": "links",
        "parenttype": "Address"
      }
    ]
  }
}
```

> `name` = `PT Maju Jaya-Billing` → simpan nilai ini (URL-encode `PT%20Maju%20Jaya-Billing`);
> dipakai untuk GET/PUT/DELETE berikutnya. Jika duplikat, `name` jadi `PT Maju Jaya-Billing-1`.
> `links` bisa berisi lebih dari satu party (mis. Supplier + Customer sekaligus).

### 2.2 READ (satu record) — alur lengkap

**Langkah 1 — Ambil detail Address:**

```bash
curl -X GET "https://site-anda.com/api/resource/Address/PT%20Maju%20Jaya-Billing" \
  -H 'Authorization: Bearer <access_token>'
```

Respons `data` berisi seluruh field Address, termasuk child table **`links`** (daftar party yang
terhubung: `link_doctype` + `link_name`).

> **Catatan:** `data` **tidak** menyertakan nama display party (mis. `supplier_name`) — hanya
> `link_name` (kunci). Ambil detail party di langkah 2 bila perlu.

**Contoh respons (HTTP 200) — cuplikan `links`:**

```json
{
  "data": {
    "name": "PT Maju Jaya-Billing",
    "address_title": "PT Maju Jaya",
    "address_type": "Billing",
    "address_line1": "Jl. Sudirman No. 123",
    "city": "Jakarta",
    "country": "Indonesia",
    "links": [
      { "link_doctype": "Supplier", "link_name": "SUP-00001" }
    ]
  }
}
```

**Langkah 2 — Ambil info party dari `links[]` (opsional):**

Untuk tiap `links[i]`, GET doctype-nya dengan `link_name` untuk mendapat nama display — pola sama
dengan [prd_contact.md §2.2](./prd_contact.md) langkah 2:

```bash
curl -X GET "https://site-anda.com/api/resource/Supplier/SUP-00001" \
  -H 'Authorization: Bearer <access_token>'
```

**Total record count — `GET /api/method/frappe.client.get_count`**

Untuk kebutuhan pagination / lazy loading (mis. menampilkan "900 dari 1000 Address"), ambil
**total record yang cocok dengan filter** lewat method whitelisted `get_count`. Respons list
(`GET /api/resource/Address`) **tidak** menyertakan total — hitung terpisah dengan endpoint ini.

```bash
curl -G "https://site-anda.com/api/method/frappe.client.get_count" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Address' \
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

### 2.3 READ (daftar) — `GET /api/resource/Address`

Parameter pilihan: `fields`, `filters`, `order_by`, `limit_start`, `limit_page_length`.

> **Lazy loading / pagination:** `limit_page_length` = jumlah record per halaman,
> `limit_start` = index awal (offset). Contoh ambil **per 50 data**:

```bash
# Halaman 1 — record 1–50
curl -G "https://site-anda.com/api/resource/Address" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","address_title","address_type","address_line1","city","country","is_primary_address","disabled"]' \
  --data-urlencode 'filters=[["disabled","=",0]]' \
  --data-urlencode 'limit_start=0' \
  --data-urlencode 'limit_page_length=50'
```

> Halaman berikutnya naikkan `limit_start` kelipatan 50 (`50`, `100`, dst.) dengan
> `limit_page_length=50` tetap. Jumlah halaman = `total / 50`, dengan `total` diperoleh
> dari `get_count` (§2.2). `limit_page_length=0` = ambil **semua** record (tanpa LIMIT),
> seperti contoh di bawah ini.

Daftar address milik sebuah party → filter child table `links` (Dynamic Link), contoh lengkap ada
di **[prd_supplier.md §4.2](./prd_supplier.md) langkah 3** (Supplier) /
**[prd_customer.md §2.2](./prd_customer.md) langkah 3** (Customer):

```bash
curl -G "https://site-anda.com/api/resource/Address" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","address_title","address_type","address_line1","city","country","is_primary_address","disabled"]' \
  --data-urlencode 'filters=[[["Dynamic Link","link_doctype","=","Supplier"],["Dynamic Link","link_name","=","SUP-00001"]]]' \
  --data-urlencode 'limit_page_length=0'
```

### 2.4 UPDATE — `PUT /api/resource/Address/{name}`

Kirim **hanya field yang diubah**. `links` bersifat **replace-all** (bukan append) — bila mengubah
link, sertakan semua link lama + baru.

```bash
curl -X PUT "https://site-anda.com/api/resource/Address/PT%20Maju%20Jaya-Billing" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "address_line2": "Gedung C, Lt. 8",
    "pincode": "10230"
  }'
```

**Contoh respons (HTTP 200):** objek `data` berisi dokumen terbaru.

### 2.5 Non-aktifkan (disarankan) — `PUT /api/resource/Address/{name}`

**Aturan:** data Address **tidak boleh dihapus**, hanya dinon-aktifkan. Non-aktifkan dengan field
`disabled`:

```bash
curl -X PUT "https://site-anda.com/api/resource/Address/PT%20Maju%20Jaya-Billing" \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "disabled": 1 }'
```

**Contoh respons (HTTP 200):** objek `data` terbaru dengan `"disabled": 1`.

Untuk mengaktifkan kembali: `PUT` dengan `{ "disabled": 0 }`.

> **Efek non-aktif:** Address tidak dipilih sebagai default billing/shipping (`get_default_address`
> memfilter `disabled=0`) dan tidak muncul sebagai opsi preferred. **Data tidak dihapus.**

> ⚠️ **Jangan gunakan `DELETE /api/resource/Address/{name}`** — alamat hilang permanen dari tab
> "Address" semua party yang ter-link, dan gagal bila sudah dipakai transaksi (SO/PO/Invoice)
> (*"linked with"*). Karena aturan "jangan hapus", endpoint DELETE tidak dipakai.

### 2.6 Export ke Excel / CSV — `GET /api/method/frappe.core.doctype.data_export.exporter.export_data`

Pola sama dengan [Supplier §4.6](./prd_supplier.md). Endpoint whitelisted, tanpa kustomisasi.

**Export Excel (.xlsx):**

```bash
curl -G "https://site-anda.com/api/method/frappe.core.doctype.data_export.exporter.export_data" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Address' \
  --data-urlencode 'file_type=Excel' \
  --data-urlencode 'with_data=1' \
  --data-urlencode 'select_columns={"Address":["name","address_title","address_type","address_line1","address_line2","city","state","pincode","country","is_primary_address","is_shipping_address","disabled"]}' \
  --data-urlencode 'all_doctypes=0' \
  -o address.xlsx
```

**Export CSV:**

```bash
curl -G "https://site-anda.com/api/method/frappe.core.doctype.data_export.exporter.export_data" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Address' \
  --data-urlencode 'file_type=CSV' \
  --data-urlencode 'with_data=1' \
  -o address.csv
```

**Contoh respons:** file biner (attachment) — `filename="Address.xlsx"` / `Address.csv`.

**Parameter & catatan:** sama dengan [prd_supplier.md §4.6](./prd_supplier.md) (tabel parameter). Khusus Address:
- **Child table:** `links` (Dynamic Link ke party) — dengan `all_doctypes=1` (default) kolom child
  ikut diexport. Gunakan `all_doctypes=0` + `select_columns` agar rapi (contoh di atas).
- **Filter per party:** gabung `filters` Dynamic Link, mis.
  `[["links","link_doctype","=","Supplier"],["links","link_name","=","SUP-00001"]]`.
- **Izin export:** role harus punya hak **Export** pada doctype Address (default: `Sales User`,
  `Purchase User`, `Accounts User`, `Maintenance User`, dst.).
- **Alternatif template/import:** `download_template` ([§4.6](./prd_supplier.md)) — sama.
- **Alternatif frontend:** `GET /api/resource/Address?fields=...&limit_page_length=0` → JSON →
  generate di klien.
- **Postman:** request `4.8` di folder `4. Address` (lihat §4).

### 2.7 Import masal (bulk) — CSV / XLSX

Alur sama dengan [Supplier §4.7](./prd_supplier.md): upload file → buat `Data Import` → `form_start_import`.

**Payload Data Import (Langkah 2):**

```bash
curl -X POST https://site-anda.com/api/resource/Data%20Import \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "reference_doctype": "Address",
    "import_type": "Insert New Records",
    "import_file": "/private/files/address_import.csv",
    "mute_emails": 1,
    "status": "Pending"
  }'
```

> Detail upload (Langkah 1) & start/pantau import (Langkah 3): lihat **[prd_supplier.md §4.7](./prd_supplier.md)**.

**Catatan khusus Address:**
- **Child table:** `links` (Dynamic Link ke party) — kolom `links.link_doctype` /
  `links.link_name`.
- **`name` autoname:** `{address_title}-{address_type}` tetap berlaku; kosongkan kolom `name`
  pada Insert, atau isi `address_title` + `address_type`.
- **Izin:** role harus punya hak **Import** pada doctype Address.
- **Postman:** request `4.9` di folder `4. Address` (lihat §4).

---

## 3. GET pendukung UI — Address

Untuk dropdown/daftar address milik sebuah party, pakai **filter Dynamic Link** (contoh di
§2.3 / [prd_supplier.md §4.2](./prd_supplier.md) langkah 3 /
[prd_customer.md §2.2](./prd_customer.md) langkah 3).

### 3.1 GET Country (dropdown negara) — `GET /api/resource/Country`

Field `country` di Address adalah **Link → Country** (doctype `Country` berisi daftar negara
standar). Ambil daftar untuk dropdown:

```bash
curl -G "https://site-anda.com/api/resource/Country" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["name","country_name","code"]' \
  --data-urlencode 'order_by=country_name asc' \
  --data-urlencode 'limit_page_length=0'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    { "name": "Indonesia", "country_name": "Indonesia", "code": "ID" },
    { "name": "Singapore", "country_name": "Singapore", "code": "SG" }
  ]
}
```

> `name` (kunci) = `country_name`. Kirim `name` pada field `country` Address.

### 3.2 GET opsi `address_type` — via metadata `DocField`

`address_type` adalah **Select** dengan opsi tetap. Ambil opsi-opsinya lewat API (pola sama
seperti [`supplier_type` §5.2](./prd_supplier.md)):

```bash
curl -G "https://site-anda.com/api/resource/DocField" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'fields=["fieldname","fieldtype","options"]' \
  --data-urlencode 'filters=[["parent","=","Address"],["fieldname","=","address_type"]]'
```

**Contoh respons (HTTP 200):**

```json
{
  "data": [
    {
      "fieldname": "address_type",
      "fieldtype": "Select",
      "options": "Billing\nShipping\nOffice\nPersonal\nPlant\nPostal\nShop\nSubsidiary\nWarehouse\nCurrent\nPermanent\nOther"
    }
  ]
}
```

**Cara pakai di UI:** pecah `options` dengan `\n` → array opsi dropdown. *(Fallback: hardcode 12
opsi dari §1 bila role tidak punya akses baca `DocField`.)*

### 3.3 Catatan — `city`

Field `city` di Address adalah **Data teks bebas** — tidak ada master/sumber data; user mengisi
manual (mis. nama kota/kabupaten). Tidak perlu fetch dari endpoint mana pun.

---

## 4. Catatan tambahan — Address

- **Autentikasi & token:** sama — lihat [prd_supplier.md §3](./prd_supplier.md) dan [`prd_oauth.md`](../prd_oauth.md).
- **Error umum:** sama — lihat [prd_supplier.md §6](./prd_supplier.md).
- **Role:** baca/tulis luas (`Sales User`, `Purchase User`, `Accounts User`, `Maintenance User`);
  hapus `System Manager`.
- **Koleksi Postman:** contoh Address ada di folder **`4. Address`** pada
  `docs/postman/postman_erpnext_api.json`.
