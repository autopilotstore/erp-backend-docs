# PRD — REST API Doctype Warehouse (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Warehouse** di ERPNext (Frappe),
> diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Stock (ERPNext)
- **Doctype:** `Warehouse`
- **Versi API:** `/api/method/...` (API v1) — method whitelisted `frappe.client.*`, `name` dikirim di **body**
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

---

## 1. Ruang lingkup

| # | Endpoint (POST `/api/method/...`) | Operasi | `name` di |
|---|---|---|---|
| 1 | `frappe.client.insert` | Buat Warehouse baru (CREATE) | body (`doc`) |
| 2 | `frappe.client.get` | Ambil detail 1 Warehouse (READ) | body |
| 3 | `frappe.client.get_list` | Daftar Warehouse (READ list) | body (filters) |
| 4 | `frappe.client.save` | Ubah Warehouse (UPDATE) | body (`doc`) |
| 5 | `frappe.client.set_value` | Non-aktifkan Warehouse (`disabled=1`) — pengganti DELETE | body |
| 6 | `frappe.client.get_list` | Daftar Warehouse Type (dropdown UI, §5.1) | body |
| 7 | `frappe.client.insert` | Buat Warehouse Type baru (CREATE, §5.1) | body (`doc`) |
| 8 | `frappe.client.get_list` | Daftar Company (dropdown UI, §5.2) | body |
| 9 | `frappe.client.get_list` | Daftar akun persediaan (dropdown UI, §5.3) | body |
| 10 | `frappe.client.get_count` | Total record Warehouse sesuai filter — pagination (§4.2) | body |
| 11 | `erpnext.stock.doctype.warehouse.warehouse.get_children` (GET) | Ambil node tree Warehouse (§5.4) | query |
| 12 | `frappe.client.insert` | Alokasikan user ke Warehouse (User Permission, §4.1) | body (`doc`) |
| 13 | `frappe.client.get_list` | Daftar user terhubung ke Warehouse (reverse lookup, §4.2) | body |
| 14 | `frappe.client.delete` | Cabut alokasi user (User Permission, §4.4) | body |
| 15 | `frappe.client.get_list` | Daftar User (dropdown alokasi, §5.5) | body |
| 16 | `frappe.client.insert` | Buat Address ter-link ke Warehouse (doctype Address, §4.1) | body (`doc`) |
| 17 | `frappe.client.get_list` | Daftar Address milik Warehouse (Dynamic Link, §4.2) | body |
| 18 | `frappe.client.set_value` | Ubah / non-aktifkan Address ter-link (§4.4) | body |
| 19 | `frappe.client.get_list` | Daftar Country & opsi `address_type` (dropdown, §5.6–5.7) | body |

> **Konvensi pemanggilan (penting):** seluruh operasi memakai method whitelisted **`frappe.client.*`**
> dengan `name` (dan filter) dikirim lewat **body JSON**, bukan di URL path. Alasannya: `name`
> Warehouse bisa mengandung karakter seperti `/` (mis. `gudang utara 1 rt 01/rw 02 - PTMJ`) —
> memanggil `/api/resource/Warehouse/{name}` dengan name seperti itu **tidak andal** di belakang
> nginx/proxy (error 404/400). Mengirim `name` di body menghindari masalah ini sepenuhnya.
> **Format respons:** method `/api/method/...` membungkus hasil di **`"message"`** (bukan `"data"`
> seperti `/api/resource/...`).

> Dokumen ini hanya membahas **data wajib terisi** + field penting. Export/import masal
> **tidak dipakai** di aplikasi ini (tidak didokumentasikan).

---

## 2. Ringkasan field & data

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `warehouse_name` | Data | Nama gudang. `reqd: 1`. |
| 🔴 **WAJIB** | `company` | Link → Company | `reqd: 1`, **read-only setelah dibuat**. |
| 🟠 **DISARANKAN** | `parent_warehouse` | Link → Warehouse | Node tree. Hanya node `is_group=1` di company yang sama. Kosong = root. |
| 🟠 **DISARANKAN** | `is_group` | Check | `1` = group warehouse; `0` = ledger warehouse. Default `0`. |
| 🟠 | `warehouse_type` | Link → Warehouse Type | Khusus non-group. §2.1 no. 4. |
| 🟠 | `account` | Link → Account | Akun persediaan (posting GL saat perpetual inventory). §2.2. |
| ⚪ **Tidak dipakai** | `address_line_1`, `address_line_2`, `city`, `state`, `pin` | Data | **Kosongkan** — alamat pindah ke doctype `Address` (§2.4). |
| 🟠 | `phone_no`, `mobile_no` | Data (Phone) | Telepon gudang (tetap inline di `tabWarehouse`). |
| ⚪ **Opsional** | `email_id` | Data | Email gudang (tersembunyi di form, tetap valid via API). |
| ⚪ **Opsional** | `default_in_transit_warehouse` | Link → Warehouse | Tampil bila `warehouse_type != "Transit"`. Harus type `Transit`, non-group, company sama. |
| ⚪ **Opsional** | `is_rejected_warehouse` | Check | `1` = gudang khusus barang reject/QC gagal. Default `0`. |
| ⚪ **Opsional** | `customer` | Link → Customer | Khusus Subcontracting Inward (§2.1 no. 5). Default kosong. |
| ⚪ **Otomatis — jangan dikirim** | `name` | — | `name = "{warehouse_name} - {abbr company}"` (autoname, tanpa naming series). |
| ⚪ **Read-only — jangan dikirim** | `lft`, `rgt`, `old_parent` | — | Dikelola sistem tree. |

> Catatan: **alamat gudang kini memakai doctype `Address`** (ter-link via Dynamic Link), bukan field
> inline. Field inline `address_line_1/city/state/pin` dibiarkan kosong. Telepon/email
> (`phone_no`/`mobile_no`/`email_id`) tetap inline di `tabWarehouse`. Detail di §2.4.

### 2.1 Catatan penting

1. **`name` dibentuk dari `warehouse_name` + abbr Company** — bukan naming series. Contoh:
   `warehouse_name="Gudang Pusat"`, company "PT Maju Jaya" (abbr `PTMJ`) →
   `name = "Gudang Pusat - PTMJ"`. Seluruh operasi memakai `name` ini (dikirim di body, §1).
   Karena deterministik, cek duplikat bisa dilakukan sebelum CREATE (§4.1 Langkah 0).
2. **`company` & `is_group`** dikunci di form Desk; `company` juga `read_only_depends_on` utk
   dokumen yang sudah ada. **Jangan** mengubah `company` lewat API. `warehouse_name` hanya bisa
   diedit saat dokumen baru — mengubahnya pada dokumen yang ada **tidak mengubah `name`** (autoname
   hanya saat insert). Bila harus rename, gunakan `frappe.client.rename_doc` (whitelisted).
3. **Tree (NestedSet)** — `parent_warehouse` harus node `is_group=1` di company yang sama;
   `lft/rgt` dikelola otomatis. DELETE diblokir bila ada child, qty di `Bin`, atau
   `Stock Ledger Entry` → pakai **non-aktif** (`disabled=1`) sebagai gantinya.
4. **`warehouse_type`** — menandai kategori gudang; hanya berlaku utk non-group. Default bawaan
   install: `Transit`. **Gudang toko/retail JANGAN memakai `Transit`** — `Transit` khusus gudang
   perantara saat Material Transfer. Untuk gudang toko, gunakan tipe `Store` (§5.1).
   Stok berkurang saat penjualan **otomatis** karena warehouse dipakai sebagai sumber di
   Delivery Note / Sales Invoice (`update_stock=1`) atau POS Profile (lihat §2.3).
5. **`customer`** — hanya untuk alur **Subcontracting Inward** (barang dikirim ke subkontraktor,
   lalu diterima kembali). Untuk penggunaan normal biarkan kosong. `depends_on: !disabled`.
6. **Role yang dibutuhkan** — baca: `Stock User` / `Sales User` / `Purchase User` /
   `Accounts User`; tulis/buat/hapus: `Item Manager`. (Warehouse Type: baca `Stock User` dsb.;
   tulis `System Manager` / `Item Manager` / `Stock Manager`.)

### 2.2 Akun persediaan (`account`) & klien non-akunting

**Apa itu `account`:** akun persediaan untuk **posting ke General Ledger** saat perpetual
inventory aktif. Boleh dikosongkan — sistem me-resolve otomatis dengan urutan
(`get_warehouse_account`):

1. `account` milik Warehouse sendiri;
2. akun `parent_warehouse` (diwarisi lewat tree);
3. **`Company.default_inventory_account`** (field *"Default Inventory Account"* di doctype
   Company);
4. satu-satunya akun `account_type=Stock` di company tersebut.

Bila tidak ketemu dan warehouse non-group → error: *"Please set Account in Warehouse … or
Default Inventory Account in Company …"*.

**Untuk klien yang TIDAK memakai fitur akunting:**
- **Biarkan `account` di Warehouse kosong.** Chart of Accounts default sudah membuat minimal satu
  akun Stock (mis. *"Stock In Hand - PTMJ"*), sehingga fallback no. 4 otomatis terpenuhi —
  CREATE tetap berhasil tanpa mengisi apa pun.
- Bila ingin eksplisit, set `Company.default_inventory_account = "Stock In Hand - {abbr}"`.
- Jika klien **tidak ingin posting GL sama sekali**, set `Company.enable_perpetual_inventory = 0`
  → validasi akun pada `warehouse.validate()` dilewati dan `account` tidak dipakai sama sekali.
  (Catatan: default setup ERPNext adalah `enable_perpetual_inventory = 1`.)

### 2.3 Gudang toko (retail/POS) & penautan Item

**Gudang toko = gudang ledger biasa** (`is_group=0`). Stok berkurang saat penjualan karena
transaksi memakainya sebagai warehouse sumber:
- **Delivery Note / Sales Invoice:** baris item memakai `warehouse` ini; pastikan submit dengan
  `update_stock = 1`.
- **POS (kasir toko):** buat `POS Profile` → `warehouse` = gudang toko ini, `update_stock = 1`
  (opsional `validate_stock_on_save = 1`).

**Penautan item ke warehouse TIDAK di doctype Warehouse** (tidak ada child table item di sini),
melainkan di sisi **Item**:
- `Item` → child table **`item_defaults`** (Item Defaults) → `default_warehouse` (per `company`,
  `reqd`). Ini default gudang item untuk semua transaksi.
- `Item` → child table **`reorder_levels`** (Item Reorder) → `warehouse`, `warehouse_reorder_level`,
  `warehouse_reorder_qty`, `material_request_type` (reorder per gudang).
- Per transaksi, field `warehouse` di tiap baris bisa di-override.
- Stok per item+warehouse (qty, reserved, projected) dilacak otomatis di doctype **`Bin`** —
  tidak diinput manual.

> Alur implementasi: buat Warehouse → buat Item → isi `item_defaults[].default_warehouse` →
> stok otomatis tercatat di `Bin` saat ada transaksi.

### 2.4 Alamat gudang — doctype `Address` (ter-link)

Alamat gudang disimpan di **doctype `Address`** dan dihubungkan ke Warehouse lewat child table
**`links`** (Dynamic Link) pada Address — **bukan** lewat field di Warehouse.

- **Cara menghubungkan:** `links: [{ "link_doctype": "Warehouse", "link_name": "<name warehouse>" }]`.
  `link_name` wajib memakai **`name` Warehouse** (dengan suffix `- abbr`), bukan `warehouse_name`.
- **Multi alamat:** satu warehouse boleh punya **banyak** Address (mis. primer + shipping).
  Tandai alamat utama dengan `is_primary_address = 1` (cukup satu primer per warehouse).
- **`address_type`:** gunakan opsi **`Warehouse`** (salah satu nilai Select doctype Address).
  `name` Address otomatis = `{address_title}-{address_type}` (mis. `Gudang Pusat-Warehouse`).
- **Field Address:** `address_line1`, `address_line2`, `city`, `state`, `pincode`, `country`
  (perhatikan: Address memakai `address_line1`/`pincode`, berbeda dari Warehouse
  `address_line_1`/`pin`). Wajib: `address_type`, `address_line1`, `city`, `country`.
- **Kontak tetap inline:** `phone_no` / `mobile_no` / `email_id` tetap di Warehouse (tidak pindah ke
  Address) — keputusan desain.
- Referensi lengkap doctype Address: **[prd_address.md](../contact/prd_address.md)**.

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Detail lengkap setup OAuth Client ada di **[`prd_oauth.md`](../prd_oauth.md)**.

---

## 4. CRUD — Doctype Warehouse

### 4.1 CREATE — `frappe.client.insert`

**Langkah 0 — Pre-check (wajib sebelum CREATE)**

Karena `name = warehouse_name + " - " + abbr`, duplikat terjadi bila `warehouse_name` + `company`
sama. Ambil dulu `abbr` Company untuk memprediksi `name`, lalu cek keberadaan — keduanya lewat
`frappe.client.get` / `frappe.client.get_list`:

```bash
# (a) Ambil abbr company (untuk memprediksi name)
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Company",
    "name": "PT Maju Jaya"
  }'
```

```bash
# (b) Cek duplikat warehouse_name + company
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "fields": ["name","warehouse_name","company"],
    "filters": [["warehouse_name","=","Gudang Raw Material"],["company","=","PT Maju Jaya"]],
    "limit_page_length": 1
  }'
```

**Hasil & aturan:**
- `message` kosong (`[]`) → lanjut ke CREATE.
- `message` terisi → blokir CREATE, tampilkan pesan: *"Warehouse {warehouse_name} sudah ada di company ini."*

> **Catatan backend:** tanpa pre-check, ERPNext tetap melempar `DuplicateEntryError` saat insert
> duplikat. Pre-check memberi pesan ramah & lebih cepat.

**Payload minimum (data wajib) — `frappe.client.insert`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Warehouse",
      "warehouse_name": "Gudang Raw Material",
      "company": "PT Maju Jaya"
    }
  }'
```

**Contoh request (lengkap — gudang ledger):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Warehouse",
      "warehouse_name": "Gudang Raw Material",
      "company": "PT Maju Jaya",
      "parent_warehouse": "All Warehouses - PTMJ",
      "is_group": 0,
      "warehouse_type": "Storage",
      "phone_no": "+622189000123",
      "mobile_no": "+6281234567890"
    }
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": {
    "name": "Gudang Raw Material - PTMJ",
    "owner": "Administrator",
    "creation": "2026-08-27 09:15:00.000000",
    "modified": "2026-08-27 09:15:00.000000",
    "modified_by": "Administrator",
    "docstatus": 0,
    "idx": 0,
    "warehouse_name": "Gudang Raw Material",
    "company": "PT Maju Jaya",
    "is_group": 0,
    "parent_warehouse": "All Warehouses - PTMJ",
    "warehouse_type": "Storage",
    "disabled": 0,
    "is_rejected_warehouse": 0,
    "phone_no": "+622189000123",
    "mobile_no": "+6281234567890"
  }
}
```

> `name` = `Gudang Raw Material - PTMJ` → simpan nilai ini; dipakai untuk operasi berikutnya
> (semuanya dikirim di body).

> Variants berikut cukup dijadikan nilai **`doc`** pada `frappe.client.insert` di atas
> (contoh request lengkap), lalu baca `name` dari respons `message`.

**Varian A — Group Warehouse (node parent tree):**

```json
{
  "doctype": "Warehouse",
  "warehouse_name": "Gudang Regional",
  "company": "PT Maju Jaya",
  "is_group": 1
}
```

> `is_group = 1` → `name = "Gudang Regional - PTMJ"`. Child warehouse mengacu ke sini via
> `parent_warehouse`. **Group tidak boleh punya `warehouse_type`** (depends_on `!is_group`).

**Varian B — Gudang toko (retail/POS):**

```json
{
  "doctype": "Warehouse",
  "warehouse_name": "Toko Cikarang",
  "company": "PT Maju Jaya",
  "is_group": 0,
  "parent_warehouse": "All Warehouses - PTMJ",
  "warehouse_type": "Store"
}
```

> `warehouse_type` **bukan** `Transit` — biarkan kosong atau gunakan tipe `Store` (§5.1).
> Lalu set `POS Profile` → `warehouse` = `Toko Cikarang - PTMJ`, `update_stock = 1`
> (lihat §2.3). Stok otomatis berkurang saat penjualan.

**Varian C — Transit / default in-transit:**

```json
{
  "doctype": "Warehouse",
  "warehouse_name": "Gudang Transit Jakarta",
  "company": "PT Maju Jaya",
  "is_group": 0,
  "parent_warehouse": "All Warehouses - PTMJ",
  "warehouse_type": "Transit"
}
```

> Lalu pada warehouse non-transit, set `default_in_transit_warehouse` (single field — pakai
> `frappe.client.set_value`):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "name": "Gudang Raw Material - PTMJ",
    "fieldname": { "default_in_transit_warehouse": "Gudang Transit Jakarta - PTMJ" }
  }'
```

**Varian D — Warehouse reject (barang QC gagal):**

```json
{
  "doctype": "Warehouse",
  "warehouse_name": "Gudang Reject",
  "company": "PT Maju Jaya",
  "is_group": 0,
  "parent_warehouse": "All Warehouses - PTMJ",
  "is_rejected_warehouse": 1
}
```

**Varian E — dengan akun persediaan eksplisit (bila ingin):**

```json
{
  "doctype": "Warehouse",
  "warehouse_name": "Gudang Pusat",
  "company": "PT Maju Jaya",
  "is_group": 0,
  "parent_warehouse": "All Warehouses - PTMJ",
  "account": "1201 - Stock In Hand - PTMJ"
}
```

> Umumnya **tidak perlu** mengisi `account` — fallback otomatis sudah menanganinya (lihat §2.2).

**Buat Address ter-link ke Warehouse (doctype Address) — setelah CREATE**

Alamat gudang disimpan di doctype **`Address`** dan ditautkan via `links` → Warehouse (detail §2.4).
Field `address_line_1`/`city`/`state`/`pin` di Warehouse **tidak dipakai** (kosongkan);
telepon/email tetap di Warehouse.

**Langkah 1 — CREATE Address ter-link (ulangi untuk multi alamat; cukup satu `is_primary_address=1`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Address",
      "address_title": "Gudang Raw Material",
      "address_type": "Warehouse",
      "address_line1": "Kawasan Industri MM2100 Blok A-1",
      "address_line2": "Lt. 2",
      "city": "Cikarang",
      "state": "Jawa Barat",
      "pincode": "17530",
      "country": "Indonesia",
      "is_primary_address": 1,
      "links": [
        { "link_doctype": "Warehouse", "link_name": "Gudang Raw Material - PTMJ" }
      ]
    }
  }'
```

> `name` Address otomatis = `{address_title}-{address_type}` → `Gudang Raw Material-Warehouse`.
> `link_name` wajib memakai **`name` Warehouse** (suffix `- abbr`); `address_type` = `Warehouse`.
> Multi alamat: ulangi dengan `address_title`/`address_type` berbeda, satu primer saja. Detail
> lengkap doctype Address: [prd_address.md](../contact/prd_address.md).

**Alokasikan user ke warehouse (User Permission) — wajib setelah CREATE**

Setelah warehouse dibuat (`name` sudah diketahui), frontend mengalokasikan **user mana yang boleh
mengakses warehouse ini** dengan membuat **`User Permission`** (`allow = "Warehouse"`,
`for_value = <name warehouse>`). Doctype Warehouse sendiri **tidak punya field alokasi user** —
koneksi user↔warehouse tersimpan di doctype `User Permission` (bukan di `tabWarehouse`).

**Langkah 1 — Pre-check duplikat (wajib sebelum insert):**

Backend menolak `User Permission` duplikat (kombinasi `user` + `allow` + `for_value` sama). Cek dulu:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "User Permission",
    "fields": ["name"],
    "filters": [["user","=","kasir.cikarang@perusahaan.co.id"],["allow","=","Warehouse"],["for_value","=","Toko Cikarang - PTMJ"]],
    "limit_page_length": 1
  }'
```

- `message` kosong (`[]`) → lanjut ke insert.
- `message` terisi → lewati (alokasi sudah ada) — alur idempoten.

**Langkah 2 — CREATE User Permission (ulangi per user):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "User Permission",
      "user": "kasir.cikarang@perusahaan.co.id",
      "allow": "Warehouse",
      "for_value": "Toko Cikarang - PTMJ",
      "is_default": 1,
      "apply_to_all_doctypes": 1
    }
  }'
```

> **Keterangan:**
> - `for_value` wajib memakai **`name` Warehouse** (dengan suffix `- abbr`), **bukan** `warehouse_name`.
> - `is_default = 1` → warehouse ini menjadi nilai default saat user membuat transaksi baru.
> - `apply_to_all_doctypes = 1` (default) → berlaku di semua dokumen yang memakai link Warehouse;
>   set `0` bila ingin batasi ke doctype tertentu (`applicable_for`).
> - Hanya role **System Manager** yang boleh membuat `User Permission` (permission doctype ini
>   hanya milik System Manager).
> - `User Permission` **tidak punya field `disabled`** — mencabut alokasi = **hapus record**
>   (`frappe.client.delete`, lihat §4.4).
> - Tanpa User Permission, user melihat **semua** warehouse yang diizinkan role-nya; dengan User
>   Permission, list dibatasi ke warehouse yang dialokasikan. Detail lengkap:
>   [prd_user.md §4.6](../contact/prd_user.md).

### 4.2 READ (satu record) & total count

**Langkah 1 — Ambil detail Warehouse (`frappe.client.get`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "name": "Gudang Raw Material - PTMJ"
  }'
```

Respons `message` berisi seluruh field Warehouse (seperti respons CREATE), termasuk `lft`/`rgt`.

**Langkah 2 — Daftar user yang terhubung ke warehouse (reverse lookup):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "User Permission",
    "fields": ["name","user","is_default"],
    "filters": [["allow","=","Warehouse"],["for_value","=","Toko Cikarang - PTMJ"]],
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "name": "abc123", "user": "kasir.cikarang@perusahaan.co.id", "is_default": 1 }
  ]
}
```

> `message` berisi daftar user yang boleh mengakses warehouse ini; `name` (id acak record User
> Permission) dipakai untuk operasi **cabut alokasi** (§4.4). Filter `allow = "Warehouse"`
> memastikan hanya alokasi warehouse yang diambil.

**Langkah 3 — Daftar alamat milik Warehouse (Dynamic Link):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Address",
    "fields": ["name","address_title","address_type","address_line1","city","country","is_primary_address"],
    "filters": [[["Dynamic Link","link_doctype","=","Warehouse"],["Dynamic Link","link_name","=","Gudang Raw Material - PTMJ"]]],
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    {
      "name": "Gudang Raw Material-Warehouse",
      "address_title": "Gudang Raw Material",
      "address_type": "Warehouse",
      "address_line1": "Kawasan Industri MM2100 Blok A-1",
      "city": "Cikarang",
      "country": "Indonesia",
      "is_primary_address": 1
    }
  ]
}
```

> Ambil terpisah — respons `frappe.client.get` Warehouse **tidak** menyertakan daftar address
> ter-link. Filter memakai child table `Dynamic Link` pada Address.

**Total count — `frappe.client.get_count`** (untuk pagination / lazy loading):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_count \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "filters": [["disabled","=",0]]
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": 24
}

> Respons berupa `message` = jumlah record yang cocok. Nilai ini dipakai menghitung total halaman
> saat lazy loading di §4.3.

### 4.3 READ (daftar) — `frappe.client.get_list`

```bash
# Daftar — halaman 1, filter company + non-disabled
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "fields": ["name","warehouse_name","company","is_group","parent_warehouse","warehouse_type","disabled"],
    "filters": [["company","=","PT Maju Jaya"],["disabled","=",0]],
    "limit_start": 0,
    "limit_page_length": 50
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    {
      "name": "Gudang Raw Material - PTMJ",
      "warehouse_name": "Gudang Raw Material",
      "company": "PT Maju Jaya",
      "is_group": 0,
      "parent_warehouse": "All Warehouses - PTMJ",
      "warehouse_type": "Storage",
      "disabled": 0
    }
  ]
}
```

> Halaman berikutnya naikkan `limit_start` kelipatan `50`; `limit_page_length=0` = ambil **semua**
> record.

### 4.4 UPDATE — `frappe.client.save`

Update memakai `save`: kirim dokumen (hasil `frappe.client.get` yang dimodifikasi); `name` ada di
dalam body.

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Warehouse",
      "name": "Gudang Raw Material - PTMJ",
      "warehouse_name": "Gudang Raw Material",
      "company": "PT Maju Jaya",
      "parent_warehouse": "All Warehouses - PTMJ",
      "is_group": 0,
      "warehouse_type": "Storage",
      "phone_no": "+622189000456",
      "mobile_no": "+6281234567890"
    }
  }'
```

**Contoh respons (HTTP 200):** objek `message` berisi dokumen terbaru (field berubah).

> **Catatan:** `save` membangun ulang dokumen dari dict — kirim dokumen yang konsisten/lengkap
> (idealnya hasil GET yang diubah; child table berlaku **replace-all**). Untuk perubahan kecil
> (satu-dua field) gunakan `frappe.client.set_value` (lihat §4.5) — lebih aman. Jangan mengubah
> `company`/`is_group`/`warehouse_name` begitu warehouse sudah dipakai transaksi (§2.1 no. 2).
> Mengubah `account` pada warehouse yang sudah punya Stock Ledger Entry memunculkan peringatan
> backend (akun lama vs baru) — pastikan keputusan bisnis dulu.

**Kelola alokasi user (saat UPDATE):**

- **Tambah alokasi** → ulangi `frappe.client.insert` `User Permission` (§4.1 Langkah 2).
- **Cabut alokasi** → hapus record `User Permission` (tidak ada field `disabled`):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.delete \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "User Permission",
    "name": "abc123"
  }'
```

> `name` = id record User Permission dari reverse lookup (§4.2 Langkah 2). Berbeda dari Warehouse,
> `User Permission` adalah **record assignment** — hapus aman (bukan master data "jangan hapus").

**Ubah alamat gudang (doctype Address):**

Alamat berada di doctype `Address`, bukan di Warehouse. Ubah lewat `frappe.client.set_value` (per
field) atau `frappe.client.save` (dokumen penuh), dengan `name` Address dari Langkah 3 (§4.2):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Address",
    "name": "Gudang Raw Material-Warehouse",
    "fieldname": { "address_line1": "Kawasan Industri MM2100 Blok B-2" }
  }'
```

> Non-aktifkan alamat: `fieldname: { "disabled": 1 }` pada Address. Detail:
> [prd_address.md](../contact/prd_address.md).

### 4.5 Non-aktifkan (disarankan) — `frappe.client.set_value`

**Aturan:** data Warehouse **tidak boleh dihapus**, hanya dinon-aktifkan. Non-aktif adalah perubahan
satu field → pakai `set_value`:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "name": "Gudang Reject - PTMJ",
    "fieldname": { "disabled": 1 }
  }'
```

**Contoh respons (HTTP 200):** objek `message` terbaru dengan `"disabled": 1`.

Untuk mengaktifkan kembali: `fieldname: { "disabled": 0 }`.

> **Efek non-aktif:** Warehouse tidak muncul sebagai pilihan pada transaksi baru (Stock Entry,
> Delivery Note, Purchase Receipt) dan tidak muncul di tree UI (filter default `disabled=0`).
> Data `Bin`/SLE tetap utuh.

> ⚠️ **Jangan gunakan `frappe.client.delete`** — backend memblokir delete bila masih ada qty di
> `Bin`, `Stock Ledger Entry`, atau child warehouse, dan menghapus `Item Default` yang
> mereferensikannya. Karena aturan master data "jangan hapus", delete tidak dipakai.

> **User Permission saat warehouse dinon-aktifkan:** record `User Permission` yang menunjuk ke
> warehouse ini **tidak ikut dihapus**. Jika ingin mencabut akses semua user sekaligus, hapus
> record-nya satu per satu via `frappe.client.delete` (§4.4). Warehouse `disabled` tetap tidak
> muncul sebagai pilihan pada transaksi baru.

---

## 5. GET pendukung UI

### 5.1 GET / CREATE Warehouse Type — dropdown UI

**Daftar (`frappe.client.get_list`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse Type",
    "fields": ["name","description"],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "name": "Store", "description": "Gudang toko / retail" },
    { "name": "Transit", "description": null }
  ]
}
```

**CREATE tipe baru** (`frappe.client.insert`; `name` diisi manual — `autoname: Prompt`):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Warehouse Type",
      "name": "Store",
      "description": "Gudang toko / retail"
    }
  }'
```

> `Warehouse Type` hanyalah master kecil (field `description`). Default bawa install: `Transit`.
> Role tulis: `System Manager` / `Item Manager` / `Stock Manager`.

### 5.2 GET Company — dropdown UI

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Company",
    "fields": ["name","abbr"],
    "filters": [["is_group","=",0]],
    "limit_page_length": 0
  }'
```

### 5.3 GET Account (akun persediaan) — dropdown UI

Untuk field `account`, hanya ambil akun `account_type=Stock` (non-group) di company yang sama:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Account",
    "fields": ["name"],
    "filters": [["account_type","=","Stock"],["is_group","=",0],["company","=","PT Maju Jaya"]],
    "limit_page_length": 0
  }'
```

### 5.4 GET node tree Warehouse — `get_children`

Untuk merender tree Warehouse (dipakai juga oleh Tree view Desk):

```bash
curl -G "https://site-anda.com/api/method/erpnext.stock.doctype.warehouse.warehouse.get_children" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Warehouse' \
  --data-urlencode 'parent=All Warehouses - PTMJ' \
  --data-urlencode 'company=PT Maju Jaya' \
  --data-urlencode 'is_root=false'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "value": "Gudang Raw Material - PTMJ", "expandable": 0 },
    { "value": "Toko Cikarang - PTMJ", "expandable": 0 }
  ]
}
```

> `expandable: 1` = node group (`is_group=1`). `parent` kosong + `is_root=true` mengambil node
> paling atas (umumnya `All Warehouses - {abbr}`). Default mengecualikan warehouse `disabled`.

### 5.5 GET User (dropdown alokasi) — `frappe.client.get_list`

Untuk memilih user saat mengalokasikan warehouse (§4.1), ambil user aktif:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "User",
    "fields": ["name","full_name","enabled"],
    "filters": [["enabled","=",1]],
    "order_by": "full_name asc",
    "limit_page_length": 0
  }'
```

> Kirim `name` (= email user) ke field `user` pada `User Permission`. Detail user/role:
> [prd_user.md §5.1](../contact/prd_user.md).

### 5.6 GET Country (dropdown alamat) — `frappe.client.get_list`

Untuk field `country` pada Address:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Country",
    "fields": ["name"],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

> Kirim `name` (= `country_name`) ke field `country` Address.

### 5.7 GET opsi `address_type` — `frappe.client.get_list` (DocField)

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "DocField",
    "fields": ["fieldname","fieldtype","options"],
    "filters": [["parent","=","Address"],["fieldname","=","address_type"]],
    "limit_page_length": 1
  }'
```

> Opsi ada di `options` (pecah dengan `\n`), termasuk `Warehouse`. Alternatif hardcode:
> `Billing`, `Shipping`, `Office`, `Personal`, `Plant`, `Postal`, `Shop`, `Subsidiary`, `Warehouse`,
> `Current`, `Permanent`, `Other`.

---

## 6. Penanganan error umum

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | Role tidak punya akses | `{"message": "Not permitted"}` |
| 404 | Resource tidak ditemukan | `{"exc_type":"DoesNotExistError","message":"Resource Not Found"}` |
| 417 | Field wajib kosong (`reqd`) | `{"exc_type":"MandatoryError","message":"warehouse_name is mandatory"}` |
| 417 | Validasi gagal (duplicate / akun inventory / tree) | `{"exc_type":"ValidationError","message":"Missing Inventory Account - PTMJ"}` |
| 417 | Duplikat `name` Address (autoname `{title}-{type}`) | `{"exc_type":"DuplicateEntryError","message":"... already exists"}` |

> **Catatan:** karena seluruh pemanggilan memakai `/api/method/...`, hasil sukses dibungkus di
> `message` (bukan `data`). Body error tetap berbentuk `{ "exc_type": ..., "exception": ..., "message": ... }`.
> Method `frappe.client.get` dengan `name` yang tidak ada → 404 `DoesNotExistError`.

---

## 7. Koleksi Postman

Seluruh pemanggilan di atas tersedia dalam **satu koleksi Postman** yang siap import:
`docs/postman/postman_erpnext_api.json` (Collection v2.1) — berisi seluruh modul API ERPNext
(OAuth 2.0 + Supplier + Customer + Contact + Address + …) dan siap ditambah modul Stock (Warehouse).
