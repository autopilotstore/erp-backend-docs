# PRD — REST API Doctype Product Bundle (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Product Bundle** (paket/produk bundel **statis**)
> di ERPNext (Frappe), diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Selling (ERPNext) — path `erpnext.selling.doctype.product_bundle`
- **Doctype:** `Product Bundle` (+ child table `Product Bundle Item`)
- **Versi API:** `/api/method/...` (API v1) — method whitelisted `frappe.client.*`, `name` dikirim di **body**
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

> **Ruang lingkup dokumen ini: DATA MASTER saja** (CRUD Product Bundle + field, validasi, error).
> Efek paket di transaksi (baris `packed_items` di Sales Order / Sales Invoice / POS Invoice /
> Delivery Note / Purchase, dsb.) **tidak** dibahas di sini.

> ⚠️ **Jangan tertukar dengan `Dynamic Product Bundle`.** ERPNext punya doctype bawaan **`Product Bundle`**
> (dokumen ini) = paket **statis**: komponen & qty sudah ditentukan di master, kasir tidak memilih.
> Untuk paket **dinamis** (isian dipilih kasir saat transaksi) lihat
> **[prd_item_dynamic_product_bundle.md](./prd_item_dynamic_product_bundle.md)** (`Dynamic Product Bundle Option/
> Item/Item Group`, custom app `baseapp`) — jangan mencampur keduanya.

---

## 1. Ruang lingkup

| # | Endpoint (POST `/api/method/...`) | Operasi | `name` di |
|---|---|---|---|
| 1 | `frappe.client.insert` | Buat Product Bundle (CREATE) | body (`doc`) |
| 2 | `frappe.client.get` | Ambil detail 1 Product Bundle + baris komponen (READ) | body |
| 3 | `frappe.client.get_list` | Daftar Product Bundle / pre-check duplikat (READ list) | body (filters) |
| 4 | `frappe.client.get_value` | Baca 1–2 field saja (§4.2) | body |
| 5 | `frappe.client.save` | Ubah Product Bundle **lengkap** (kirim dokumen hasil GET, §4.4) | body (`doc`) |
| 6 | `frappe.client.set_value` | Ubah field tunggal — `description`, `disabled`, dsb. (§4.4) | body |
| 7 | `frappe.client.get_count` | Total record sesuai filter — pagination (§4.2) | body |
| 8 | `frappe.client.set_value` | Non-aktifkan / aktifkan kembali (`disabled`, §4.5) | body |
| 9 | `frappe.client.delete` | HAPUS — **diblokir** bila item induknya sudah dipakai transaksi submitted (§4.5) | body |
| 10 | `erpnext.selling.doctype.product_bundle.product_bundle.get_new_item_code` (GET) | Dropdown **Item induk** yang boleh dibuat paket (§5.1) | query |
| 11 | `frappe.client.get_list` | Daftar **Item** (prasyarat & komponen) + cek `is_stock_item` (§5.2) | body |
| 12 | `frappe.client.get_list` | Daftar **Item Group** leaf (dropdown `item_group` saat membuat Item) (§5.3) | body |
| 13 | `frappe.client.get_list` | Daftar Product Bundle aktif (`disabled=0`) — dropdown POS/laporan (§5.4) | body |

> **Konvensi pemanggilan (penting):** seluruh operasi memakai method whitelisted **`frappe.client.*`**
> dengan `name` (dan filter) dikirim lewat **body JSON**, bukan di URL path — sama seperti PRD
> Warehouse / POS Profile / Item Group / Item. Nama Product Bundle = kode item induk sehingga bisa
> mengandung spasi / karakter khusus; mengirim `name` di body menghindari masalah URL-encoding di
> belakang nginx/proxy.
> **Format respons:** method `/api/method/...` membungkus hasil di **`"message"`** (bukan `"data"`
> seperti `/api/resource/...`).

> **Perbedaan utama dengan Item Group / Item:**
> - **`name = new_item_code`** (autoname dari field Link, **bukan** naming series). Jadi **satu Item
>   hanya boleh punya satu Product Bundle**, dan dokumen `Product Bundle` **tidak** otomatis membuat Item —
>   Item induk harus dibuat lebih dulu (§2.2).
> - **Tidak bisa di-rename:** `allow_rename = 0` → `frappe.client.rename_doc` ditolak
>   (❝Product Bundle not allowed to be renamed❞, §4.4).
> - **BUTUH role lain:** Product Bundle **bukan** wilayah `Item Manager` — lihat §2.1 no. 5.
> - Ada **soft-delete** (`disabled`) — pola sama seperti Warehouse, **beda** dengan Item Group yang
>   tidak punya `disabled`.

> Import masal (Data Import) **tidak** didokumentasikan di sini (aturan aplikasi: master data dibuat
> satu per satu lewat form/API).

---

## 2. Ringkasan field & data

### 2.1 Doctype `Product Bundle` (parent)

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `new_item_code` | Link → Item | **Item induk paket** (label form: *Parent Item*). Harus item **non-stok** & bukan fixed asset (§2.2). Sekaligus menjadi `name` dokumen. |
| 🔴 **WAJIB** | `items` | Table → Product Bundle Item | Minimal **1 baris** komponen. `reqd: 1` — baris kosong ditolak (§6). |
| 🟠 | `description` | Data | Keterangan paket **bebas** (tidak tersinkron ke field apa pun di Item). Boleh kosong. |
| 🟠 | `disabled` | Check | `1` = paket **non-aktif** (tidak dapat dipilih di transaksi). Default `0`. |
| ⚪ **Otomatis — jangan dikirim** | `name` | — | `name = new_item_code` (autoname). Hanya di-set saat insert. |
| ⚪ **Otomatis — jangan dikirim** | `owner`, `creation`, `modified`, `modified_by`, `docstatus`, `idx`, `doctype` | — | Field standar. `docstatus` selalu `0` (doctype **non-submittable**). `creation` + `modified` **wajib ikut dikirim** saat UPDATE lewat `save` (§4.4). |
| ✖️ **Tidak ada** | naming series, `company`, `currency`, `uom` | — | Tidak ada di doctype ini. UOM paket mengikuti UOM item induk (harga memakai `Item Price` item induk — lihat [prd_item_price.md](./prd_item_price.md)). |

**Catatan penting**

1. **`name` = `new_item_code`** dan **tidak bisa di-rename** (`allow_rename = 0`, lihat §4.4).
   Karena `name` adalah kunci dokumen, **satu Item hanya boleh jadi induk satu paket**. Paket kedua
   dengan item induk yang sama akan ditolak (`409 DuplicateEntryError`, §6) — cek duplikat lebih dulu
   lewat §4.1 Langkah 0.
2. **Tidak ada naming series.** Tidak ada field `naming_series`; jadi frontend tidak perlu mengirim apa pun
   untuk penomoran. (Beda dengan Item yang punya `naming_series` opsional.)
3. **Doctype non-submittable.** `docstatus` tetap `0`; tidak ada aksi submit/cancel/amend untuk
   Product Bundle itu sendiri.
4. **Harga paket = harga `Item` induk** (dari `Item Price` seperti item biasa). Field `rate` pada baris
   komponen **disembunyikan & tidak dipakai** selama `Selling Settings → Editable Bundle Item Rates`
   (`editable_bundle_item_rates`) = **0** (nilai di site dev saat dokumen ini dibuat: `0`). Bila setting
   tersebut diaktifkan, rate paket dihitung dari rate komponen saat transaksi — jangan menaruh logika
   harga di frontend, lihat [prd_item_price.md](./prd_item_price.md).
5. **Role (v16, dari DocPerm doctype ini):**

   | Role | Read | Write | Create | Delete |
   |---|---|---|---|---|
   | `Stock User` | ✅ | ✖️ | ✖️ | ✖️ |
   | `Sales User` | ✅ | ✅ | ✅ | ✅ |
   | `Stock Manager` | ✅ | ✅ | ✅ | ✅ |
   | `System Manager` | ✅ (bypass) | ✅ | ✅ | ✅ |
   | `Item Manager` | ✖️ | ✖️ | ✖️ | ✖️ |

   > **Perhatian:** `Item Manager` (role yang biasa dipakai untuk Item / Item Group) **tidak punya akses
   > sama sekali** ke Product Bundle → respons **403 Not permitted**. Minimal `Stock User` untuk baca,
   > dan `Sales User` / `Stock Manager` untuk tulis.
6. **Tidak ada `User Permission` khusus.** Product Bundle tidak dibatasi per user/company (tidak ada
   field `company`). Pembatasan akses cukup lewat role.

### 2.2 Prasyarat: Item induk & Item komponen

Product Bundle **hanya menautkan** Item yang sudah ada (tidak membuat Item). Urutan kerja frontend:
buat Item dulu (lihat **[prd_item.md](./prd_item.md)**), baru buat Product Bundle.

**Item induk (`new_item_code`) — wajib:**

| Syarat | Nilai | Alasan / pesan bila salah |
|---|---|---|
| Ada di master Item | — | `LinkValidationError`: ❝Could not find Parent Item: X❞ |
| **Bukan item stok** | `is_stock_item = 0` | `ValidationError`: ❝Parent Item X must not be a Stock Item❞ |
| **Bukan fixed asset** | `is_fixed_asset = 0` | `ValidationError`: ❝Parent Item X must not be a Fixed Asset❞ |
| Umumnya tetap dijual | `is_sales_item = 1` | Tidak divalidasi backend, tapi paket yang tidak bisa dijual tidak ada gunanya |

- Stok paket **tidak** dilacak; barang yang keluar adalah komponennya. Karena itu induk wajib
  `is_stock_item = 0` (*Maintain Stock* dimatikan di form Item).
- Ubah `is_stock_item` pada item yang **sudah** jadi induk paket **diblokir** backend
  (`ValidationError`, contoh pesan panjang di §6) — jadi pastikan nilainya benar sejak pembuatan Item.

**Item komponen (`items[].item_code`) — wajib:**

| Syarat | Nilai | Alasan / pesan bila salah |
|---|---|---|
| Ada di master Item | — | `LinkValidationError`: ❝Could not find Row #1: Item: X❞ |
| **Bukan Product Bundle aktif** | — | `ValidationError`: ❝Row #1: Child Item should not be a Product Bundle...❞ (**paket tidak boleh bersarang**) |
| `qty > 0` | — | `ValidationError`: ❝Row #1: Quantity cannot be a non-positive number...❞ |
| UOM tidak memaksa bilangan bulat (bila `qty` pecahan) | — | `ValidationError`: ❝Row 1: Quantity (1.5) cannot be a fraction. To allow this, disable 'Must be Whole Number' in UOM Nos.❞ |
| Utama: item stok | `is_stock_item = 1` | Tidak divalidasi backend. Komponen non-stok juga diterima (mis. jasa), tetapi stok yang bergerak hanya item stok. |

> **Catatan:** komponen boleh ber-`has_variants = 1` (template) menurut backend — field `link_filters`
> pada `Product Bundle Item.item_code` (`has_variants = 0`) **hanya menyaring dropdown di Desk**.
> Frontend **wajib** menyaring sendiri: kirim hanya Item dengan `has_variants = 0` (dan sebaiknya
> `disabled = 0`).

### 2.3 Child table `Product Bundle Item` (`items`)

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `item_code` | Link → Item | Item komponen. |
| 🔴 **WAJIB** | `qty` | Float | Qty per 1 paket. Harus **> 0**; harus bilangan bulat bila UOM-nya *Must Be Whole Number*. |
| 🟠 | `description` | Text Editor | `fetch_from item_code.description` (`fetch_if_empty: 1`) — **otomatis terisi** dari deskripsi Item bila user tidak mengubahnya. Aman dikirim/diubah manual (dipakai cetak/cetakan nota). |
| ⚪ **read-only** | `uom` | Link → UOM | `fetch_from item_code.stock_uom` (`fetch_if_empty: 1`), `read_only: 1`. **Jangan dikirim** — biarkan server mengisi. |
| ⚪ tidak dipakai | `rate` | Float | Field lama, `hidden: 1`, `print_hide: 1`. Tetap bisa disimpan (contoh `7000000`) tetapi **tidak** memengaruhi harga paket (lihat §2.1 no. 4). Kirim `0` / biarkan kosong. |
| ⚪ **Otomatis** | `name`, `parent`, `parentfield`, `parenttype`, `idx`, `owner`, `creation`, `modified`, ... | — | Field standar child. `name` = hash 10 karakter (contoh `n9hq2g57pd`); **jangan** dipakai sebagai identitas — gunakan `item_code` (dan `idx`) saat mengirim ulang baris. |

**Catatan child table**

1. **Replace-all.** `insert`/`save` memperlakukan `items` sebagai keseluruhan daftar: kirim **seluruh**
   baris yang diinginkan. Baris yang tidak ikut dikirim akan hilang (§4.4).
2. **`uom` & `description` otomatis.** Bila `uom` tidak dikirim, server mengisi dari `Item.stock_uom`
   (contoh `Nos`). Bila `description` tidak dikirim, terisi dari `Item.description`.
3. **`uom` yang salah TIDAK ditolak backend.** Uji nyata: baris komponen dengan item ber-`stock_uom = Nos`
   tetap tersimpan walau `uom` dikirim `"Kg"` (tidak ada validasi UOM). Karena itu **jangan** mengirim `uom`
   dari frontend — kirim hanya `item_code` + `qty` (+ `description` bila mau ubah teksnya).
4. **Baris duplikat diperbolehkan.** Item yang sama boleh muncul 2× dalam satu paket (contoh `qty` 1 dan
   3) — backend **tidak** menolak. Bila aturan bisnis melarang, frontend harus mencegahnya (gabungkan qty).
5. **`Product Bundle Item` tidak bisa dibaca lewat `get_list`.** Query langsung ke doctype child ini
   hanya mengembalikan `name` (field yang diminta diabaikan). **Baca komponen lewat `frappe.client.get`
   dokumen induknya** (`message.items`).

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Detail lengkap setup OAuth Client ada di **[`prd_oauth.md`](../prd_oauth.md)** (Authorization Code + Refresh Token).
Contoh di dokumen ini memakai `https://site-anda.com`; untuk site dev gunakan `https://erpnext.localhost`.

---

## 4. CRUD — Doctype Product Bundle

### 4.1 CREATE — `frappe.client.insert`

**Langkah 0 — Pre-check (wajib, 2 bagian)**

(a) Pastikan **item induk** belum dipakai paket lain (`name = new_item_code`):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Product Bundle",
    "fields": ["name","new_item_code","disabled"],
    "filters": [["new_item_code","=","PAKET-LAPTOP-BACKPACK"]],
    "limit_page_length": 1
  }'
```

- `message` kosong (`[]`) → lanjut CREATE.
- `message` terisi → blokir, tampilkan: *"Item PAKET-LAPTOP-BACKPACK sudah dipakai sebagai paket."*
  (tanpa pre-check, backend tetap menolak dengan `409 DuplicateEntryError` — §6.)

(b) Pastikan item induk memenuhi prasyarat (§2.2) — `is_stock_item = 0`, `is_fixed_asset = 0`
(pola query di §5.2). Tanpa langkah ini, CREATE akan gagal dengan `417 ValidationError`.

**Payload minimum (data wajib):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Product Bundle",
      "new_item_code": "PAKET-LAPTOP-BACKPACK",
      "items": [
        { "item_code": "LAPTOP-14INCH", "qty": 1 },
        { "item_code": "BACKPACK-CANVAS", "qty": 1 }
      ]
    }
  }'
```

**Contoh respons sukses (HTTP 200)** — hasil uji nyata di site dev:

```json
{
  "message": {
    "name": "PAKET-LAPTOP-BACKPACK",
    "owner": "Administrator",
    "creation": "2026-09-22 09:07:39.245281",
    "modified": "2026-09-22 09:07:39.245281",
    "modified_by": "Administrator",
    "docstatus": 0,
    "idx": 0,
    "new_item_code": "PAKET-LAPTOP-BACKPACK",
    "description": null,
    "disabled": 0,
    "doctype": "Product Bundle",
    "items": [
      {
        "name": "n9hq2g57pd",
        "idx": 1,
        "item_code": "LAPTOP-14INCH",
        "qty": 1.0,
        "description": "Laptop 14 inch",
        "rate": 0.0,
        "uom": "Nos",
        "parent": "PAKET-LAPTOP-BACKPACK",
        "parentfield": "items",
        "parenttype": "Product Bundle",
        "doctype": "Product Bundle Item"
      },
      {
        "name": "n9h400iltt",
        "idx": 2,
        "item_code": "BACKPACK-CANVAS",
        "qty": 1.0,
        "description": "Backpack Canvas",
        "rate": 0.0,
        "uom": "Nos",
        "parent": "PAKET-LAPTOP-BACKPACK",
        "parentfield": "items",
        "parenttype": "Product Bundle",
        "doctype": "Product Bundle Item"
      }
    ]
  }
}
```

> Perhatikan: `description` baris komponen terisi otomatis dari deskripsi Item, `uom` terisi `Nos`
> (dari `Item.stock_uom`), `rate` = `0.0`, dan `message.name` = `PAKET-LAPTOP-BACKPACK` (= `new_item_code`).
> Simpan `name` ini untuk operasi berikutnya.

**Contoh request (lengkap — description paket + description komponen diubah):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Product Bundle",
      "new_item_code": "PAKET-LAPTOP-TRAVEL",
      "description": "Laptop + tas travel",
      "disabled": 0,
      "items": [
        { "item_code": "LAPTOP-14INCH", "qty": 1, "description": "Laptop 14 inch (wajib)" },
        { "item_code": "BACKPACK-CANVAS", "qty": 2, "description": "Tas travel" }
      ]
    }
  }'
```

**Contoh respons (ringkas — hasil uji nyata):**

```json
{
  "message": {
    "name": "PAKET-LAPTOP-TRAVEL",
    "new_item_code": "PAKET-LAPTOP-TRAVEL",
    "description": "Laptop + tas travel",
    "disabled": 0,
    "items": [
      { "idx": 1, "item_code": "LAPTOP-14INCH", "qty": 1.0, "uom": "Nos", "description": "Laptop 14 inch (wajib)", "rate": 0.0 },
      { "idx": 2, "item_code": "BACKPACK-CANVAS", "qty": 2.0, "uom": "Nos", "description": "Tas travel", "rate": 0.0 }
    ]
  }
}
```

**Varian A — komponen sekaligus jasa/non-stok (campuran):**

```json
{
  "doctype": "Product Bundle",
  "new_item_code": "PAKET-INSTALASI",
  "items": [
    { "item_code": "LAPTOP-14INCH", "qty": 1 },
    { "item_code": "JASA-INSTALASI", "qty": 1 }
  ]
}
```

**Varian B — paket dengan komponen non-stok saja:**

```json
{
  "doctype": "Product Bundle",
  "new_item_code": "PAKET-SERVIS-BUNDLE",
  "items": [
    { "item_code": "JASA-SERVIS-A", "qty": 1 },
    { "item_code": "JASA-SERVIS-B", "qty": 1 }
  ]
}
```

> Cukup jadikan nilai `doc` pada `frappe.client.insert`, lalu baca `name` dari respons `message`.

### 4.2 READ — `frappe.client.get`, `get_value`, `get_count`

**Detail satu paket (termasuk baris komponen):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Product Bundle", "name": "PAKET-LAPTOP-BACKPACK" }'
```

Respons `message` = seluruh field parent + array `items` (bentuk sama seperti respons CREATE).
`name` produk yang tidak ada → **404** `DoesNotExistError` (❝Product Bundle X not found❞).

**Baca sebagian field saja (`frappe.client.get_value`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Product Bundle",
    "filters": { "name": "PAKET-LAPTOP-BACKPACK" },
    "fieldname": ["description","disabled"]
  }'
```

```json
{ "message": { "description": null, "disabled": 0 } }
```

**Total count (pagination / lazy loading):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_count \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Product Bundle", "filters": [["disabled","=",0]] }'
```

```json
{ "message": 10 }
```

### 4.3 READ (daftar) — `frappe.client.get_list`

```bash
# Daftar paket (halaman 1) — name = new_item_code
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Product Bundle",
    "fields": ["name","new_item_code","disabled","description"],
    "filters": [["disabled","=",0]],
    "order_by": "name asc",
    "limit_start": 0,
    "limit_page_length": 50
  }'
```

**Contoh respons (hasil uji nyata):**

```json
{
  "message": [
    { "name": "PAKET-LAPTOP-BACKPACK", "new_item_code": "PAKET-LAPTOP-BACKPACK", "disabled": 0, "description": null },
    { "name": "PAKET-LAPTOP-TRAVEL", "new_item_code": "PAKET-LAPTOP-TRAVEL", "disabled": 0, "description": "Laptop + tas travel" }
  ]
}
```

> - Halaman berikutnya: naikkan `limit_start` kelipatan `50`; `limit_page_length = 0` = ambil semua.
> - Daftar ini **tidak** memuat baris komponen. Untuk menampilkan isi paket, panggil `frappe.client.get`
>   per paket (N+1 call) atau ambil saat user membuka detail.
> - Filter yang berguna: `disabled` (0/1), `new_item_code` (= kode item induk), `description` (`like`).

### 4.4 UPDATE — `frappe.client.save` & `frappe.client.set_value`

**Aturan penting: `save` harus dikirim sebagai dokumen UTUH hasil GET.**

`frappe.client.save` membangun dokumen dari dict yang dikirim (bukan membaca DB lebih dulu), sehingga:

| Yang dikirim | Hasil |
|---|---|
| Hanya sebagian field (tanpa `creation`) | ❌ `417 CannotChangeConstantError` — ❝Value cannot be changed for **Created On**❞ |
| Ada `creation` tetapi `modified` hilang / basi | ❌ `417 TimestampMismatchError` — ❝Error: X (Product Bundle) has been modified after you have opened it... Please refresh to get the latest document.❞ |
| **Dokumen utuh hasil `get`** (dengan `creation` + `modified` terbaru) | ✅ HTTP 200 |

**Alur yang benar:** `get` → ubah field/baris yang perlu → `save` dengan objek `message` tadi apa adanya.

```bash
# Langkah 1 — ambil dokumen + simpan nilai message.modified
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Product Bundle", "name": "PAKET-LAPTOP-BACKPACK" }'
```

```bash
# Langkah 2 — kirim kembali dokumen lengkap (creation + modified ikut), ubah description & items
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Product Bundle",
      "name": "PAKET-LAPTOP-BACKPACK",
      "owner": "Administrator",
      "creation": "2026-09-22 09:07:39.245281",
      "modified": "2026-09-22 09:11:16.269914",
      "modified_by": "Administrator",
      "docstatus": 0,
      "idx": 0,
      "new_item_code": "PAKET-LAPTOP-BACKPACK",
      "description": "Paket hemat (full doc)",
      "disabled": 0,
      "items": [
        { "item_code": "LAPTOP-14INCH", "qty": 1 },
        { "item_code": "BACKPACK-CANVAS", "qty": 1 }
      ]
    }
  }'
```

```json
{
  "message": {
    "name": "PAKET-LAPTOP-BACKPACK",
    "description": "Paket hemat (full doc)",
    "modified": "2026-09-22 09:13:21.723318"
  }
}
```

> **Child table `items` = replace-all.** Bila request di atas hanya mengirim 1 baris (`LAPTOP-14INCH`,
> qty 2), maka baris `BACKPACK-CANVAS` **hilang** (terbukti di uji: hasil GET setelah save hanya berisi
> `[('LAPTOP-14INCH', 2.0)]`). Selalu kirim seluruh baris yang diinginkan — praktisnya, kirim kembali
> `message.items` hasil GET lalu ubah qty / tambah-buang baris.

**Perubahan kecil — `frappe.client.set_value`** (aman, tanpa mengirim dokumen utuh):

```bash
# Ubah deskripsi paket
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Product Bundle",
    "name": "PAKET-LAPTOP-BACKPACK",
    "fieldname": { "description": "Paket hemat laptop + tas" }
  }'
```

> - `set_value` memuat dokumen dari DB lalu menyimpan — jadi **tidak** perlu mengirim `creation`/`modified`,
>   dan tetap menjalankan validasi doctype (mis. parent harus non-stok).
> - Field standar (`creation`, `modified`, `owner`, `docstatus`, nama child table, dsb.) **ditolak**:
>   ❝Cannot edit standard fields❞.
> - Untuk mengubah **baris komponen**, `set_value` juga bisa dipakai per baris child
>   (`"doctype": "Product Bundle Item", "name": "<hash baris>"`) — namun untuk menambah/menghapus baris
>   tetap pakai `save` (replace-all).

**Jangan rename / jangan ganti `new_item_code`:**

```bash
# ❌ DITOLAK — allow_rename = 0
curl -X POST https://site-anda.com/api/method/frappe.client.rename_doc \
  -d '{ "doctype": "Product Bundle", "old_name": "PAKET-A", "new_name": "PAKET-B" }'
# → 417 ValidationError: "Product Bundle not allowed to be renamed"
```

- `frappe.client.rename_doc` **selalu gagal** untuk doctype ini (uji nyata).
- `set_value` pada `new_item_code` **berhasil secara teknis** (uji nyata) tetapi menghasilkan dokumen
  **inkonsisten**: `name` tetap `PAKET-UOM-TEST` sementara `new_item_code` menjadi `PB-RENAMED`
  (`name` hanya di-set saat insert). Bila `new_item_code` diubah, **nama dokumen tidak ikut berubah**
  sehingga dasbor/backlink menunjuk dokumen dengan nama lama.
- **Rekomendasi frontend:** perlakukan `new_item_code` sebagai **read-only setelah CREATE**. Bila kode
  item induk harus berubah, ubah kode **Item**-nya lebih dulu (renaming Item akan ikut memperbarui link
  `new_item_code`) atau **hapus lalu buat ulang** Product Bundle-nya.

### 4.5 Non-aktif (`disabled`) & HAPUS

**Non-aktif (soft-delete) — cara yang disarankan:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Product Bundle",
    "name": "PAKET-LAPTOP-BACKPACK",
    "fieldname": { "disabled": 1 }
  }'
```

```json
{ "message": { "name": "PAKET-LAPTOP-BACKPACK", "disabled": 1 } }
```

- Paket `disabled = 1` **tidak dikenali lagi sebagai paket** di transaksi (backend mencari Product Bundle
  dengan `disabled = 0`) dan ikut hilang dari daftar aktif (`get_list` filter `disabled=0`) serta dari
  dropdown §5.1.
- Aktifkan kembali: kirim `{ "disabled": 0 }` dengan pola yang sama (uji nyata: berhasil).
- Riwayat transaksi lama **tidak** terpengaruh.

**HAPUS — `frappe.client.delete`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.delete \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Product Bundle", "name": "PAKET-LAPTOP-TRAVEL" }'
```

```json
{ "message": null }
```

- Berhasil (HTTP 200, `message: null`) **bila** item induk paket belum muncul di transaksi **submitted**.
  Dokumen transaksi **draft tidak menghalangi** (uji nyata: Sales Order draft → delete tetap sukses).
- **DIBLOKIR** (HTTP 417) bila item induk sudah dipakai di dokumen **submitted** pada salah satu doctype
  berikut: Sales Order, Sales Invoice, POS Invoice, Purchase Receipt, Purchase Invoice, Stock Entry
  (child `Stock Entry Detail`), Stock Reconciliation, Purchase Order, Material Request. Contoh pesan asli:

```text
This Product Bundle is linked with <a href="/desk/sales-order/SAL-ORD-2026-00006">SAL-ORD-2026-00006</a>.
You will have to cancel these documents in order to delete this Product Bundle
```

- **Aturan produk: jangan hapus, non-aktifkan** (`disabled = 1`). Selain risiko riwayat, penghapusan
  juga akan sulit dibatalkan karena `name` tidak bisa di-rename.
- Penghapusan **Item** yang dipakai paket juga **tidak bisa**: baik item induk maupun item komponen yang
  masih tertaut di `Product Bundle` ditolak dengan `LinkExistsError` ❝You can disable this Item instead of
  deleting it.❞ → hapus/non-aktifkan Product Bundle-nya dulu, baru Item.

---

## 5. GET pendukung UI

### 5.1 GET Item induk yang boleh jadi paket — `get_new_item_code`

Method whitelisted bawaan doctype Product Bundle (dipakai juga oleh Desk saat memilih *Parent Item*).
Filter bawaan: **hanya item `is_stock_item = 0`, `is_fixed_asset = 0`, dan belum menjadi Product Bundle
(aktif)** — persis prasyarat §2.2.

```bash
curl -G "https://site-anda.com/api/method/erpnext.selling.doctype.product_bundle.product_bundle.get_new_item_code" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Product Bundle' \
  --data-urlencode 'txt=PAKET' \
  --data-urlencode 'searchfield=name' \
  --data-urlencode 'start=0' \
  --data-urlencode 'page_len=20' \
  --data-urlencode 'filters=[]'
```

**Contoh respons (HTTP 200)** — pasangan `[kode_item, nama_item]`:

```json
{
  "message": [
    ["PAKET-LAPTOP-BACKPACK", "Paket Laptop + Backpack"]
  ]
}
```

> - Ini endpoint **link-search** bergaya Desk: **semua 6 parameter wajib ada** (`doctype`, `txt`,
>   `searchfield`, `start`, `page_len`, `filters`). Bila `filters` tidak dikirim → **HTTP 500 TypeError**
>   (bukan 417/403).
> - `message` **kosong** bisa berarti "tidak ada item non-stok yang cocok" — contoh: saat
>   `PAKET-LAPTOP-BACKPACK` & `PAKET-LAPTOP-TRAVEL` masih aktif sebagai paket, keduanya **hilang** dari
>   hasil pencarian (`txt=PAKET` → `[]`); begitu salah satunya di-`disabled`, item itu muncul lagi.
> - ❗ Method ini **tidak** menyaring `Item.disabled`. Item yang dinon-aktifkan tetap muncul
>   (uji nyata: item non-stok `disabled=1` muncul di hasil). Frontend sebaiknya menyaring tambahan
>   `disabled = 0` di sisi UI, atau memakai pola §5.2 dengan filter lengkap.

### 5.2 GET Item (prasyarat induk & daftar komponen) — `frappe.client.get_list`

**(a) Verifikasi calon item induk** (dipakai di §4.1 Langkah 0b):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","is_stock_item","is_fixed_asset","has_variants","disabled"],
    "filters": [["name","=","PAKET-LAPTOP-BACKPACK"]],
    "limit_page_length": 1
  }'
```

**(b) Daftar komponen yang boleh dipilih** — hanya item yang **bukan** Product Bundle aktif:

```bash
# Langkah 1 — kumpulkan kode item yang sudah jadi paket aktif
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Product Bundle", "filters": [["disabled","=",0]], "pluck": "name", "limit_page_length": 0 }'

# Langkah 2 — daftar Item, kecualikan daftar di atas
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","stock_uom","is_stock_item"],
    "filters": [["disabled","=",0],["has_variants","=",0],["name","not in",["PAKET-LAPTOP-BACKPACK"]]],
    "order_by": "item_name asc",
    "limit_page_length": 50
  }'
```

> Backend tetap menolak komponen berupa paket aktif (§2.2) — penyaringan di UI hanya untuk UX,
> **bukan** pengganti validasi.

### 5.3 GET Item Group leaf — dropdown saat membuat Item

Item induk/komponen harus beralamat di Item Group **leaf** (`is_group = 0`) — lihat
**[prd_item_group.md](./prd_item_group.md)** §5.2:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Group",
    "fields": ["name"],
    "filters": [["is_group","=",0]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

### 5.4 GET daftar paket aktif — dropdown POS / laporan

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Product Bundle",
    "fields": ["name","new_item_code","description"],
    "filters": [["disabled","=",0]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

> Untuk menampilkan **isi** paket (nama komponen di layar kasir), panggil `frappe.client.get` per paket
> dan baca `message.items` (§2.3 no. 5). Bila ingin harga, ambil juga `Item Price` item induk —
> [prd_item_price.md](./prd_item_price.md).

---

## 6. Penanganan error umum

| Kode | `exc_type` | Kondisi | Contoh pesan (asli hasil uji) |
|---|---|---|---|
| 401 | `AuthenticationError` | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | `PermissionError` | Role tidak punya akses (mis. `Item Manager`) | `{"message": "Not permitted"}` |
| 404 | `DoesNotExistError` | Product Bundle tidak ditemukan | ❝Product Bundle X not found❞ |
| 409 | `DuplicateEntryError` | Item induk sudah menjadi paket (`name` = `new_item_code`) | ❝Product Bundle **PAKET-LAPTOP-BACKPACK** already exists❞ |
| 417 | `MandatoryError` | `new_item_code` kosong | ❝Error: Value missing for Product Bundle: Parent Item❞ |
| 417 | `MandatoryError` | `items` kosong / tidak dikirim | ❝Error: Data missing in table Items❞ |
| 417 | `LinkValidationError` | Item induk tidak ada | ❝Could not find Parent Item: ITEM-TIDAK-ADA-XYZ❞ |
| 417 | `LinkValidationError` | Item komponen tidak ada | ❝Could not find Row #1: Item: ITEM-TIDAK-ADA-XYZ❞ |
| 417 | `ValidationError` | Item induk adalah item **stok** | ❝Parent Item LAPTOP-14INCH must not be a Stock Item❞ |
| 417 | `ValidationError` | Item induk adalah **fixed asset** | ❝Parent Item PB-FA-3 must not be a Fixed Asset❞ |
| 417 | `ValidationError` | Komponen berupa Product Bundle aktif | ❝Row #1: Child Item should not be a Product Bundle. Please remove Item **HRM-PAKET-1** and Save❞ |
| 417 | `ValidationError` | `qty` ≤ 0 | ❝Row #1: Quantity cannot be a non-positive number. Please increase the quantity or remove the Item **LAPTOP-14INCH**❞ |
| 417 | `ValidationError` | `qty` pecahan pada UOM *Must Be Whole Number* | ❝Row 1: Quantity (1.5) cannot be a fraction. To allow this, disable '**Must be Whole Number**' in UOM **Nos**.❞ |
| 417 | `CannotChangeConstantError` | `save` tanpa `creation` (dokumen tidak utuh) | ❝Value cannot be changed for **Created On**❞ |
| 417 | `TimestampMismatchError` | `save` dengan `modified` basi / tanpa `modified` | ❝Error: X (Product Bundle) has been modified after you have opened it (...). Please refresh to get the latest document.❞ |
| 417 | `ValidationError` | `rename_doc` pada Product Bundle | ❝Product Bundle not allowed to be renamed❞ |
| 417 | `ValidationError` | Delete paket yang itemnya dipakai transaksi submitted | ❝This Product Bundle is linked with **SAL-ORD-2026-00006**. You will have to cancel these documents in order to delete this Product Bundle❞ |
| 417 | `ValidationError` | Ubah `is_stock_item` item yang sudah jadi induk paket | ❝As there are existing submitted transactions against item **PB-GUARD**, you can not change the value of **Maintain Stock**. *Example of a linked document:* Product Bundle PB-GUARD❞ |
| 417 | `LinkExistsError` | Hapus Item yang masih dipakai Product Bundle | ❝You can disable this Item instead of deleting it.❞ |
| 500 | `TypeError` | `get_new_item_code` tanpa parameter lengkap (mis. `filters`) | respons tanpa `message` (kesalahan pemanggilan, bukan validasi) |

> **Catatan:** karena seluruh pemanggilan memakai `/api/method/...`, hasil sukses dibungkus di `message`
> (bukan `data`). Body error berbentuk
> `{ "exc_type": ..., "exception": ..., "_server_messages": "[...]" }` — pesan yang enak ditampilkan ke user
> ada di dalam `_server_messages` (array string JSON; ambil `message` tiap elemen).
> Method `frappe.client.get` dengan `name` yang tidak ada → 404 `DoesNotExistError`.

---

## 7. Koleksi Postman & catatan lanjutan

1. **Koleksi Postman:** folder **`17. Product Bundle`** sudah ditambahkan ke
   `docs/postman/postman_erpnext_api.json` (Collection v2.1) berisi **22 request**:
   - **Pre-check & CREATE (§4.1)** → `17.1`–`17.4`: pre-check duplikat `new_item_code`, CREATE minimum,
     CREATE lengkap (`description` paket + `description` komponen), dan cek prasyarat item induk.
   - **READ (§4.2–§4.3)** → `17.5`–`17.8`: `get` (detail + komponen, menyimpan `pb_creation`/`pb_modified`),
     `get_value`, `get_list`, `get_count`.
   - **UPDATE (§4.4)** → `17.9`–`17.12`: `save` **tanpa** `creation` (contoh error
     `CannotChangeConstantError`), `save` dengan dokumen utuh (replace-all `items`), `set_value` deskripsi,
     dan contoh `rename_doc` yang **ditolak**.
   - **Non-aktif & DELETE (§4.5)** → `17.13`–`17.16`: `disabled=1`, aktifkan kembali (`disabled=0`),
     DELETE normal, serta contoh DELETE yang **diblokir** dokumen submitted.
   - **GET pendukung UI (§5)** → `17.17`–`17.20`: `get_new_item_code` (GET, 6 parameter), Item Group leaf,
     daftar Item komponen (kecuali paket aktif), daftar paket aktif.
   - **Contoh error (§6)** → `17.21`–`17.22`: induk = item stok, komponen = Product Bundle aktif.
   - **Variabel baru:** `pb_parent_item` (mis. `PAKET-LAPTOP-BACKPACK`), `pb_parent_item_2`
     (`PAKET-LAPTOP-TRAVEL`), `pb_component_item` (`LAPTOP-14INCH`), `pb_component_item_2`
     (`BACKPACK-CANVAS`), `pb_item_group` (`Elektronik`), `pb_creation` & `pb_modified` (dipakai `save`
     dokumen utuh). Test script pada request CREATE (`17.2`, `17.3`) dan READ (`17.5`) menyimpan
     `pb_parent_item` / `pb_creation` / `pb_modified`.
   - Nomor folder lain yang sudah terpakai: `11. Item` (direncanakan), `12. Stock Reconciliation`,
     `13. Stock Ledger`, `14. Pricing Rule`, `15. Price List & Item Price`,
     `16. Dynamic Product Bundle` (direncanakan).
2. **Data contoh yang dipakai uji & dokumen ini** (site dev `erpnext.localhost`, company
   `PT Rapupa Guna Teknologi`): Item Group `Elektronik` (leaf di bawah `Products`), Item `LAPTOP-14INCH`
   & `BACKPACK-CANVAS` (stok, UOM `Nos`), Item `PAKET-LAPTOP-BACKPACK` & `PAKET-LAPTOP-TRAVEL`
   (non-stok), dan 2 Product Bundle dengan kode yang sama.
3. **File terkait:** Item [prd_item.md](./prd_item.md) (`is_stock_item`, `is_fixed_asset`,
   `item_code` dari naming series, **`hashtags`** — child table hashtag produk, lihat
   [prd_item.md §2.8](./prd_item.md)) · **paket dinamis** [prd_item_dynamic_product_bundle.md](./prd_item_dynamic_product_bundle.md)
   · Item Group [prd_item_group.md](./prd_item_group.md) (leaf untuk `item_group`) ·
   Harga [prd_item_price.md](./prd_item_price.md) (`Item Price` item induk = harga paket) ·
   Promo [prd_item_pricing_rule.md](./prd_item_pricing_rule.md) · Warehouse
   [prd_warehouse.md](../setup/prd_warehouse.md) & stok [prd_stock_ledger.md](./prd_stock_ledger.md)
   (stok bergerak dari komponen, bukan dari item paket) · OAuth [prd_oauth.md](../prd_oauth.md).
