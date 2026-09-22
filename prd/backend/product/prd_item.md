# PRD — REST API Doctype Item (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Item** di ERPNext (Frappe),
> diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Stock (ERPNext) — path `erpnext.stock.doctype.item`
- **Doctype:** `Item`
- **Versi API:** `/api/method/...` (API v1) — method whitelisted `frappe.client.*`, `name` dikirim di **body**
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

> Dokumen ini **hanya membahas master data Item** (tanpa varian & dengan varian), UOM conversion,
> barcode, foto produk, dan info stok (`tabBin`). **Stok** (Stock Ledger Entry, Opening Stock, Stock
> Entry, dsb.) dan **Price List / Item Price** akan dibuat di **file PRD terpisah** (lihat §9).

---

## 1. Ruang lingkup

| # | Endpoint (POST `/api/method/...`) | Operasi | `name` di |
|---|---|---|---|
| 1 | `frappe.client.insert` | Buat Item baru — tanpa varian (CREATE) | body (`doc`) |
| 2 | `erpnext.controllers.item_variant.create_variant` | Siapkan dokumen Item varian dari template (CREATE varian, §4.2) | body |
| 3 | `erpnext.controllers.item_variant.get_variant` | Cek varian sudah ada (pre-check varian, §4.2) | body |
| 4 | `frappe.client.insert` | Simpan varian hasil `create_variant` (CREATE) | body (`doc`) |
| 5 | `frappe.client.get` | Ambil detail 1 Item (READ) | body |
| 6 | `frappe.client.get_list` | Daftar Item / dropdown pendukung (READ list) | body (filters) |
| 7 | `frappe.client.get_count` | Total record Item sesuai filter — pagination | body |
| 8 | `frappe.client.save` | Ubah Item (UPDATE) | body (`doc`) |
| 9 | `frappe.client.set_value` | Ubah field tunggal — non-aktif, pindah `item_group`, ubah harga, dst. | body |
| 10 | `frappe.client.attach_file` | Upload foto produk (multi foto → `tabFile`, §4.7) | body (base64) |
| 11 | `frappe.client.get_list` | Daftar foto Item dari `tabFile` (READ list, §4.7) | body (filters) |
| 12 | `frappe.client.insert` | Tag foto template ke varian — buat record `File` baru menunjuk `file_url` yang sama (Pendekatan A, §4.7) | body (`doc`) |
| 13 | `frappe.client.delete` | Hapus foto (File) / Item — **dengan batasan** (§4.7, §4.8) | body |
| 14 | `frappe.client.submit` | Submit **Stock Reconciliation** — eksekusi zero stok sebelum non-aktif (§4.8) | body (`doc`) |
| 15 | `erpnext.stock.doctype.item.item.get_uom_conv_factor` | Resolve faktor konversi UOM (pendukung §5) | body |
| 16 | `erpnext.stock.doctype.item.item.get_item_attribute` | Autocomplete nilai Item Attribute (dropdown varian, §6) | body |
| 17 | `frappe.client.get_list` | Baca stok via **Bin / `tabBin`** (info stok, §4.9) | body (filters) |

> **Konvensi pemanggilan (penting):** seluruh operasi memakai method whitelisted **`frappe.client.*`**
> (kecuali method ERPNext khusus) dengan `name` (dan filter) dikirim lewat **body JSON**, bukan di
> URL path. `item_code` bisa mengandung spasi / karakter khusus — mengirim `name` di body menghindari
> masalah URL-encoding di belakang nginx/proxy.
> **Format respons:** method `/api/method/...` membungkus hasil di **`"message"`** (bukan `"data"`
> seperti `/api/resource/...`).

> **Perbedaan utama dengan Item Group:**
> - `name = item_code` — item di-generate oleh **frontend** (bukan autoname dari field lain). Lihat §2.1 no. 1.
> - Item **punya field `disabled`** → operasi non-aktif (soft-delete) berlaku (§4.8).
> - Stok Item **tidak disimpan di `tabItem`**, melainkan di `tabBin` per item+warehouse (§4.9) dan
>   riwayatnya di Stock Ledger Entry (PRD stok terpisah, §9).

---

## 2. Ringkasan field & data

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🟠 **DISARANKAN** | `item_code` | Data | Kode produk. **`reqd: 1` + `unique: 1`**, sekaligus menjadi `name`. Dibuat **manual oleh frontend** (dikirim di body). Bisa berisi spasi/karakter khusus. Bila dikosongkan, sistem hanya meng-generate otomatis bila `Stock Settings → Item Naming By = "Naming Series"` (series `STO-ITEM-.YYYY.-`). |
| 🟠 **DISARANKAN** | `item_name` | Data | Nama tampilan produk. Kosong → backend menyalin `item_code`. |
| 🔴 **WAJIB** | `item_group` | Link → Item Group | Grup produk (pilih node **leaf**/daun, `is_group=0`). Detail: [prd_item_group.md §5.2](./prd_item_group.md). |
| 🔴 **WAJIB** | `stock_uom` | Link → UOM | Satuan dasar stok (Unit of Measure). Semua qty stok & konversi dihitung relatif ke UOM ini. |
| 🟠 | `is_stock_item` | Check | `1` = produk stok (dikelola via Stock Ledger, punya `tabBin`). `0` = jasa/non-stok. |
| 🟠 | `is_dynamic_product_bundle` | Check | `1` = Item ini **paket dinamis** — komponen/isiannya dipilih kasir saat transaksi (berbeda dari `Product Bundle` bawaan ERPNext yang statis — paket statis dibahas di [prd_item_product_bundle.md](./prd_item_product_bundle.md)). **Custom field dari app `baseapp`** (fieldname tanpa prefix `custom_`, posisi setelah `is_stock_item`); hanya ada bila app tersebut terpasang. Default `0`. Struktur & CRUD pilihannya: [prd_item_dynamic_product_bundle.md](./prd_item_dynamic_product_bundle.md). |
| 🟠 | `is_sales_item` | Check | `1` = produk bisa dijual (muncul di Quotation/Sales Order/Sales Invoice/POS). |
| 🟠 | `is_purchase_item` | Check | `1` = produk bisa dibeli (muncul di Request for Quotation/Purchase Order/Purchase Receipt). |
| 🟠 | `purchase_uom` | Link → UOM | UOM default saat transaksi **beli** (bisa berbeda dari `stock_uom`; faktor konversi di `uoms`, §2.3/§5). |
| 🟠 | `has_variants` | Check | `1` = Item ini **template** varian (wajib isi `attributes`, §2.5). `0`/kosong = Item tunggal. |
| 🟠 | `is_fixed_asset` | Check | `1` = Item **aset tetap** (mis. mesin/kendaraan) yang dibeli untuk dikapitalisasi jadi **Asset** (modul Assets; `is_stock_item` ikut jadi `0`, butuh `asset_category`). Default `0` — untuk produk jualan biasa kirim `0`. |
| 🟠 | `weight_per_unit` | Float | Berat per unit produk (dipakai hitung `total_weight` pada transaksi) -> berat yang digunakan untuk perhitungan logistik/ongkos kirim. |
| 🟠 | `weight_uom` | Link → UOM | Satuan berat (mis. `Kg`, `Gram`) -> satuan produk yang digunakan untuk perhitungan logistik/ongkos kirim. |
| 🟠 | `image` | AttachImage | **Foto utama** produk — disimpan sebagai **string URL file** (mis. `/files/kaos.jpg` atau URL lengkap). Foto tambahan (multi) disimpan di `tabFile` (§4.7). |
| 🟠 | `allow_negative_stock` | Check | `1` = izinkan stok menjadi negatif (transaksi tetap jalan walau stok kurang). Default `0`. |
| 🟠 | `has_serial_no` | Check | `1` = tiap unit produk punya **nomor seri unik** (dilacak per unit via doctype `Serial No`; hanya untuk item stok). Default `0`. Menjadi **read-only** setelah Item punya riwayat stok (§2.1 no. 3). |
| 🟠 | `has_batch_no` | Check | `1` = produk dikelola per **Batch** (stok & `batch_no` dilacak per batch; item ber-batch memakai doctype `Batch`). **Wajib diisi `1` jika `has_expiry_date` diisi `1`** — backend menolak bila kedaluwarsa diaktifkan tanpa batch. Menjadi **read-only** setelah Item punya riwayat stok (§2.1 no. 3). |
| 🟠 | `has_expiry_date` | Check | `1` = produk punya tanggal kedaluwarsa (per batch). **Jika diisi `1`, `has_batch_no` wajib ikut diisi `1`** (lihat baris `has_batch_no`). Memunculkan field `shelf_life_in_days`. |
| 🟠 | `shelf_life_in_days` | Int | **Umur simpan dalam hari** — hanya relevan (muncul) saat `has_expiry_date = 1`. Berfungsi sebagai **preset/template**: ketika **Batch baru dibuat** untuk Item ini, `expiry_date` Batch dihitung **otomatis** oleh backend (mis. dari tanggal produksi/manufaktur + `shelf_life_in_days`). |
| 🟠 | `reorder_levels` | Table (child `Item Reorder`) | **Batas stok per (item, warehouse)** — ambang **min & maks**. Baris: `warehouse` (Link→Warehouse, `reqd`); `warehouse_reorder_level` (Float = **nilai ambang minimum**, satuan `stock_uom`); `material_request_type` (`reqd` — Select `Purchase`/`Transfer`/`Material Issue`/`Manufacture`, menentukan cara restock; `Purchase` = beli ke supplier); `warehouse_reorder_qty` (Float, **opsional** — untuk alur pemesanan nanti, PRD stok §9); `max_stock_level` (Float, **opsional** — **custom field dari app `baseapp`**, nilai ambang maksimum, satuan `stock_uom`; hanya ada bila `baseapp` terpasang, lihat §4.1). Hanya relevan untuk item stok (`is_stock_item=1`). Cara simpan: §4.1 (contoh CREATE). |
| 🟠 | `brand` | Link → Brand | Merek produk (opsional; bisa jadi sumber default via `brand_defaults`). |
| 🟠 | `description` | TextEditor | Deskripsi produk (HTML). Backend membersihkan HTML bila kosong/rapi. |
| 🟠 | `_user_tags` | Tags (kolom sistem) | **Tag** produk, dipisah koma (mis. `"best seller,baru"`). Bukan field definisi doctype — kolom sistem yang tersedia di semua tabel (§2.6). |
| 🟠 | `valuation_rate` | Currency | **Nilai persediaan per unit** (biaya masuk stok; dipakai hitung `stock_value` di `tabBin`). Bisa diisi 0 untuk item baru / zero valuation. |
| 🟠 | `standard_rate` | Currency | **Harga jual standar**. Mengisi field ini saat CREATE **otomatis membuat Item Price** di backend (detail: [prd_item_price.md §4.9](./prd_item_price.md)). |
| ⚪ **Otomatis — jangan dikirim** | `name` | — | `name = item_code`. |
| ⚪ **Read-only / dikelola sistem** | `valuation_rate` (stok berjalan), `last_purchase_rate`, `total_projected_qty` | — | Nilai dihitung/di-update dari transaksi stok. |
| ⚪ **Set saat varian** | `variant_of`, `variant_based_on`, `attributes` | — | Lihat §4.2. |
| ✖️ **Bukan bagian scope** | `opening_stock`, `taxes`, `item_defaults`, dst. | — | Field lanjutan boleh dipakai, tetapi mekanisme **stok** (opening stock, auto-reorder) dan **price list** didokumentasikan di PRD terpisah (§9). Penyimpanan ambang `reorder_levels` sudah dicakup di dokumen ini (lihat baris `reorder_levels` di atas & §4.1). |

> Catatan: daftar di atas adalah **data yang wajib/diperlukan** menurut kebutuhan aplikasi
> (per permintaan tim produk), bukan seluruh field Item. Field `reqd` sebenarnya oleh doctype hanya
> `item_code`, `item_group`, dan `stock_uom` (ditambah `item_name` fallback) — sisanya opsional di
> sisi backend, namun **frontend tetap disarankan mengirim** sesuai tabel di atas agar data konsisten.

### 2.1 Catatan penting

1. **`name` = `item_code` (dibuat frontend, unik).** Karena `unique: 1`, duplikat `item_code` ditolak
   di level database → cek duplikat sebelum CREATE (§4.1 Langkah 0). Autoname (naming series) hanya
   aktif bila `Stock Settings → Item Naming By = "Naming Series"` — untuk aplikasi ini **frontend
   yang menentukan `item_code`** dan mengirimnya di body. Mengubah `item_code` pada dokumen yang sudah
   ada **tidak mengubah `name`** — gunakan `frappe.client.rename_doc` bila perlu rename.
2. **`stock_uom` adalah pusat konversi.** Ubah `stock_uom` pada Item yang sudah punya transaksi stok
   **diblokir backend** (`check_stock_uom_with_bin`) — pastikan benar sejak awal. Baris `uoms` yang
   konversinya relatif ke `stock_uom` akan dikosongkan ulang bila `stock_uom` diganti (§2.3).
3. **`is_stock_item` / `has_variants` / `has_serial_no` / `has_batch_no` jadi read-only** setelah Item
   punya riwayat stok (`stock_ledger_created()`) — frontend tidak boleh mengubahnya pada Item ber-stok.
4. **Template varian tidak boleh punya stok.** Backend menolak membuat stock entry untuk template
   (`has_variants=1`) — hanya varian (`variant_of` terisi) yang bisa ditransaksikan. Template
   `is_stock_item=1` diperbolehkan, tapi **jangan pernah** melakukan transaksi stok atas nama template.
5. **Role yang dibutuhkan (v16)** — baca: `Stock Manager` / `Stock User` / `Sales User` /
   `Purchase User` / `Accounts User` / `Desk User`; **tulis/buat/hapus: `Item Manager`** (sama seperti
   Item Group, karena Item termasuk master Stock).
6. **Master Item bersifat global** (tidak per company). Default per company diatur lewat child table
   `item_defaults` (satu baris per `company`, mis. `default_warehouse`). Nilai default diresolusi
   dengan urutan prioritas: **Company → Brand → Item Group → Item**.

### 2.2 Lokasi tabel penyimpanan (data Item & pendukungnya)

| Tabel (`tab...`) | Doctype | Menyimpan |
|---|---|---|
| `tabItem` | `Item` | Master produk — **template & varian** dalam satu tabel (varian ditandai kolom `variant_of`, `has_variants`, `variant_based_on`). |
| `tabItem Variant Attribute` | `Item Variant Attribute` | Child table `attributes` — atribut & nilai per Item (template: tanpa nilai; varian: dengan nilai). |
| `tabItem Attribute` | `Item Attribute` | Master atribut (mis. `Ukuran`, `Warna`) — parent dari `tabItem Attribute Value`. |
| `tabItem Attribute Value` | `Item Attribute Value` | Child table nilai atribut pada master (mis. `S`, `M`, `L` + kolom `abbr`). |
| `tabUOM Conversion Detail` | `UOM Conversion Detail` | Child table `uoms` pada Item — faktor konversi UOM lain → `stock_uom` (§2.3, §5). |
| `tabUOM Conversion Factor` | `UOM Conversion Factor` | Master global pasangan konversi antar-UOM (mis. `Gram`→`Kg`) — dipakai `get_uom_conv_factor` (§5). |
| `tabItem Barcode` | `Item Barcode` | Child table `barcodes` — daftar barcode produk (§2.4, §4.6). |
| `tabFile` | `File` | **Semua attachment** (foto multi produk), ter-link ke Item via `attached_to_doctype` + `attached_to_name` (§4.7). |
| `tabBin` | `Bin` | **Stok per (item, warehouse)** — `actual_qty`, `projected_qty`, `stock_value`, dsb. (§4.9). |
| `tabItem Default` | `Item Default` | Child table `item_defaults` — default per company (`default_warehouse`, akun, dst.). |
| `tabItem Tax` | `Item Tax` | Child table `taxes` — template pajak default (cross-ref: [prd_item_group.md §2.3](./prd_item_group.md)). |
| `tabItem Reorder` | `Item Reorder` | Child table `reorder_levels` — batas stok (**min & maks**) per (item, warehouse). Kolom `max_stock_level` adalah **custom field dari `baseapp`**. Cara menyimpan di dokumen ini: §4.1; mekanisme reorder / order otomatis: PRD stok (§9). |

> Item Group disimpan di `tabItem Group` (lihat [prd_item_group.md §2.2](./prd_item_group.md));
> Warehouse di `tabWarehouse` (lihat [prd_warehouse.md](../setup/prd_warehouse.md)).

### 2.3 Child table `uoms` — `UOM Conversion Detail` (konversi UOM per Item)

- Field: `uom` (Link → UOM, `reqd`) + `conversion_factor` (Float).
- Satu baris untuk **`stock_uom` harus bernilai `conversion_factor = 1`** (backend memaksa, §5).
- `uom` **tidak boleh duplikat** dalam satu Item.
- Faktor konversi bisa **diisi otomatis** dari master global `UOM Conversion Factor`
  (`get_uom_conv_factor`) saat baris dikirim tanpa `conversion_factor` (§5).
- Contoh kasus & CRUD lengkap: **§5**.

### 2.4 Child table `barcodes` — `Item Barcode`

- Field: `barcode` (Data, `reqd`), `barcode_type` (pilihan: `EAN`, `UPC-A`, `CODE-39`, `EAN-13`,
  `EAN-8`, `GS1`, `GTIN`, `ISBN`, `UPC`, dsb.), `uom` (opsional — barcode per UOM).
- **Barcode bersifat unik global** — backend menolak barcode yang sudah dipakai Item lain
  (`DuplicateEntryError`/`InvalidBarcode`).
- CRUD lengkap: **§4.6**.

### 2.5 Child table `attributes` — `Item Variant Attribute` (+ master `Item Attribute`)

- Pada **template** (`has_variants=1`): baris `attributes` hanya berisi `attribute` (Link → Item
  Attribute) **tanpa nilai**.
- Pada **varian** (`variant_of` terisi): baris berisi `attribute` + `attribute_value` (nilai konkret),
  dan `attribute_value` tidak bisa diedit setelah varian dibuat.
- Master `Item Attribute` (doctype terpisah) menyimpan daftar nilai lewat child table
  `item_attribute_values` (kolom `attribute_value` + `abbr`). `abbr` dipakai membangun `item_code`
  varian otomatis (§4.2).
- Alur & contoh kasus lengkap: **§4.2**.

### 2.6 `_user_tags` (Tags)

- `_user_tags` adalah **kolom sistem** yang otomatis ada di setiap tabel Frappe (bukan field yang
  didefinisikan di `item.json`). Tipe tersimpan `Data`, isi berupa **tag dipisah koma**, mis.
  `"best seller,baru,diskon"`.
- Di form Desk dikelola lewat kontrol "Tags"; lewat API cukup dikirim sebagai field biasa pada
  `doc` (CREATE/UPDATE). Tidak ada enumerasi khusus — bebas teks.
- Dapat dipakai untuk pencarian cepat (filter `like`), namun **tidak wajib** diisi.

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Detail lengkap setup OAuth Client ada di **[`prd_oauth.md`](../prd_oauth.md)**.

---

## 4. CRUD — Doctype Item

### 4.1 CREATE — Item tanpa varian

**Langkah 0 — Pre-check (wajib sebelum CREATE)**

Karena `name = item_code` dan `item_code` `unique: 1`, cek keberadaan sebelum insert:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name"],
    "filters": [["name","=","MIN-001"]],
    "limit_page_length": 1
  }'
```

**Hasil & aturan:**
- `message` kosong (`[]`) → lanjut ke CREATE.
- `message` terisi → blokir CREATE, tampilkan pesan: *"Item {item_code} sudah ada."*

> Tanpa pre-check, ERPNext tetap melempar `DuplicateEntryError` saat insert duplikat (field `unique`).
> Pre-check memberi pesan ramah & lebih cepat.

**Payload minimum (data wajib) — `frappe.client.insert`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_code": "MIN-001",
      "item_group": "Minuman",
      "stock_uom": "Pcs"
    }
  }'
```

> `item_name` kosong → backend menyalin `item_code`; `is_stock_item`/`is_sales_item`/`is_purchase_item`
> default `0`.

**Contoh request (lengkap — produk stok yang dibeli & dijual):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_code": "MIN-001",
      "item_name": "Air Mineral 600ml",
      "item_group": "Minuman",
      "is_stock_item": 1,
      "is_sales_item": 1,
      "is_purchase_item": 1,
      "stock_uom": "Pcs",
      "purchase_uom": "Dus",
      "uoms": [
        { "uom": "Pcs", "conversion_factor": 1 },
        { "uom": "Dus", "conversion_factor": 12 }
      ],
      "has_variants": 0,
      "is_fixed_asset": 0,
      "weight_per_unit": 0.6,
      "weight_uom": "Kg",
      "image": "/files/min-001.jpg",
      "allow_negative_stock": 0,
      "has_serial_no": 0,
      "has_batch_no": 1,
      "has_expiry_date": 1,
      "shelf_life_in_days": 180,
      "brand": "Aqua",
      "description": "Air mineral kemasan botol 600ml",
      "_user_tags": "best seller,baru",
      "valuation_rate": 3500,
      "standard_rate": 5000,
      "item_defaults": [
        { "company": "PT Maju Jaya", "default_warehouse": "Toko Cikarang - PTMJ" }
      ]
    }
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": {
    "name": "MIN-001",
    "owner": "Administrator",
    "creation": "2026-09-01 09:20:00.000000",
    "item_code": "MIN-001",
    "item_name": "Air Mineral 600ml",
    "item_group": "Minuman",
    "is_stock_item": 1,
    "is_sales_item": 1,
    "is_purchase_item": 1,
    "stock_uom": "Pcs",
    "purchase_uom": "Dus",
    "uoms": [
      { "name": "abc001", "uom": "Pcs", "conversion_factor": 1 },
      { "name": "abc002", "uom": "Dus", "conversion_factor": 12 }
    ],
    "has_variants": 0,
    "is_fixed_asset": 0,
    "weight_per_unit": 0.6,
    "weight_uom": "Kg",
    "image": "/files/min-001.jpg",
    "has_serial_no": 0,
    "has_batch_no": 1,
    "has_expiry_date": 1,
    "shelf_life_in_days": 180,
    "valuation_rate": 3500,
    "standard_rate": 5000,
    "item_defaults": [
      { "name": "def001", "company": "PT Maju Jaya", "default_warehouse": "Toko Cikarang - PTMJ" }
    ]
  }
}
```

> `name` = `MIN-001` → simpan nilai ini; dipakai untuk operasi berikutnya (dikirim di body).

> **Catatan `has_batch_no` / `has_expiry_date` / `shelf_life_in_days`:** ketiga field ini opsional
> di sisi backend, tetapi bila `has_expiry_date` dikirim `1`, maka `has_batch_no` **wajib** ikut `1`
> (backend menolak bila tidak). `shelf_life_in_days` (mis. `180`) berfungsi sebagai **preset** — saat
> Batch baru dibuat untuk Item ini, `expiry_date` Batch dihitung otomatis dari tanggal produksi +
> `shelf_life_in_days`. Setelah Item punya riwayat stok, `has_batch_no` tidak bisa diubah (§2.1 no. 3).

> **Catatan `standard_rate`:** mengisi `standard_rate` saat CREATE **otomatis membuat Item Price**
> pada Price List default selling di backend (`after_insert → add_price`). Detail pengelolaan harga
> (Item Price, Price List, bulk price) ada di **[prd_item_price.md](./prd_item_price.md)** — di dokumen
> ini cukup kirim `standard_rate` sebagai harga jual standar awal.

**Varian A — Produk jasa (non-stok, tidak dibeli):**

```json
{
  "doctype": "Item",
  "item_code": "SRV-001",
  "item_name": "Biaya Instalasi",
  "item_group": "Jasa",
  "is_stock_item": 0,
  "is_sales_item": 1,
  "is_purchase_item": 0,
  "stock_uom": "Nos",
  "standard_rate": 150000
}
```

**Varian B — Produk hanya dibeli (bahan baku):**

```json
{
  "doctype": "Item",
  "item_code": "RM-001",
  "item_name": "Gula Pasir 1kg",
  "item_group": "Bahan Baku",
  "is_stock_item": 1,
  "is_sales_item": 0,
  "is_purchase_item": 1,
  "stock_uom": "Pcs",
  "valuation_rate": 15000
}
```

**Simpan batas stok (minimum & maksimum) — `reorder_levels` (per item + warehouse):**

Untuk menyimpan **nilai ambang stok per (item, warehouse)** — **minimum** (field standar
`warehouse_reorder_level`) dan **maksimum** (`max_stock_level`, custom field dari app **`baseapp`**) —
kelak dipakai untuk notifikasi re-stock / auto reorder (mekanismenya di PRD stok, §9), sertakan
child table **`reorder_levels`** (doctype `Item Reorder`) saat CREATE/UPDATE Item. Berlaku
**replace-all** (sama seperti `uoms`/`barcodes`): kirim seluruh baris yang diinginkan. Hanya untuk
item stok (`is_stock_item=1`); nilai ambang dalam **satuan `stock_uom`**.

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_code": "MIN-001",
      "item_group": "Minuman",
      "stock_uom": "Pcs",
      "reorder_levels": [
        { "warehouse": "Toko Cikarang - PTMJ", "warehouse_reorder_level": 20, "max_stock_level": 120, "material_request_type": "Purchase" },
        { "warehouse": "Gudang Pusat - PTMJ", "warehouse_reorder_level": 50, "max_stock_level": 300, "material_request_type": "Purchase" }
      ]
    }
  }'
```

> - `warehouse` **`reqd`** — wajib diisi. `warehouse_reorder_level` = **nilai ambang minimum**
>   (mis. `20` = min 20 Pcs), dalam satuan `stock_uom`.
> - `material_request_type` **`reqd`** — menentukan **jenis Material Request** yang kelak dibuat
>   sistem saat stok mencapai ambang (mekanisme auto-reorder, PRD stok §9). Opsi: `Purchase` =
>   restock lewat **pembelian ke supplier** (cocok untuk item `is_purchase_item=1`, seperti contoh
>   ini); `Transfer` = pindah gudang; `Material Issue` = pengeluaran material; `Manufacture` =
>   produksi.
> - `warehouse_reorder_qty` **opsional** — boleh dikosongkan dulu karena jumlah pemesanan dibahas di
>   PRD stok (§9).
> - `max_stock_level` **opsional** — **custom field dari app `baseapp`** (bukan field standar
>   ERPNext): nilai **ambang maksimum** per (item, warehouse), satuan `stock_uom`. Hanya tersedia
>   bila `baseapp` terpasang & sudah di-`bench migrate`. Murni untuk pencatatan/notifikasi — tidak
>   memicu logika stok apa pun.
> - Untuk item yang sudah ada, gunakan `frappe.client.save` dengan seluruh baris `reorder_levels`
>   (replace-all); baris ini ikut terbaca pada `frappe.client.get` (§4.3).
> - Menyimpan nilai di sini **belum memicu apa pun** di backend (tidak membuat Material Request
>   otomatis) — mekanisme reorder & notifikasi diatur belakangan (PRD stok, §9).

### 4.2 CREATE — Item dengan varian

> **Konsep:** Satu **template** (`has_variants=1`, berisi daftar atribut tanpa nilai) + beberapa
> **varian** (`variant_of=<template>`, berisi atribut + nilai konkret). Template & varian **disimpan
> di tabel yang sama** `tabItem`; atribut tiap Item di `tabItem Variant Attribute`; master daftar
> atribut/nilai di `tabItem Attribute` + `tabItem Attribute Value` (§2.2). Hanya **varian** yang bisa
> ditransaksikan stok/jual — template hanya sebagai kerangka.

**Contoh kasus:** Produk "Kaos Polos" punya atribut `Ukuran` (nilai `S`/`M`/`L`). Dibutuhkan 3 varian:
`KAOS-POLOS-S`, `KAOS-POLOS-M`, `KAOS-POLOS-L`.

**Langkah 1 — Buat master Item Attribute (sekali saja, bisa dipakai banyak template):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item Attribute",
      "attribute_name": "Ukuran",
      "item_attribute_values": [
        { "attribute_value": "Small", "abbr": "S" },
        { "attribute_value": "Medium", "abbr": "M" },
        { "attribute_value": "Large", "abbr": "L" }
      ]
    }
  }'
```

> `abbr` dipakai membangun `item_code` varian otomatis (mis. `M` → `KAOS-POLOS-M`). Tersimpan di
> `tabItem Attribute` (+ `tabItem Attribute Value`).

**Langkah 2 — Buat template (`has_variants=1`, `attributes` tanpa nilai):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_code": "KAOS-POLOS",
      "item_name": "Kaos Polos",
      "item_group": "Pakaian",
      "is_stock_item": 1,
      "is_sales_item": 1,
      "is_purchase_item": 1,
      "stock_uom": "Pcs",
      "has_variants": 1,
      "variant_based_on": "Item Attribute",
      "attributes": [ { "attribute": "Ukuran" } ]
    }
  }'
```

> Backend memaksa `attributes` wajib ada saat `has_variants=1` (*"Attribute table is mandatory"*).

**Langkah 3a — Buat varian via method resmi `create_variant`:**

```bash
curl -X POST https://site-anda.com/api/method/erpnext.controllers.item_variant.create_variant \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "item": "KAOS-POLOS",
    "args": { "Ukuran": "M" },
    "use_template_image": false
  }'
```

**Contoh respons (HTTP 200) — dokumen varian (BELUM tersimpan):**

```json
{
  "message": {
    "doctype": "Item",
    "item_code": "KAOS-POLOS-M",
    "item_name": "Kaos Polos-M",
    "variant_of": "KAOS-POLOS",
    "variant_based_on": "Item Attribute",
    "item_group": "Pakaian",
    "is_stock_item": 1,
    "is_sales_item": 1,
    "is_purchase_item": 1,
    "stock_uom": "Pcs",
    "attributes": [ { "attribute": "Ukuran", "attribute_value": "M" } ]
  }
}
```

> `create_variant` mengembalikan dokumen **belum disimpan** dengan `item_code`/`item_name` otomatis
> `{template}-{abbr}` dan field `reqd`/field terpilih (Item Variant Settings) tersalin dari template.
> **Simpan** dengan `frappe.client.insert` (tambahkan `"doctype": "Item"` pada hasil di atas):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_code": "KAOS-POLOS-M",
      "item_name": "Kaos Polos-M",
      "variant_of": "KAOS-POLOS",
      "variant_based_on": "Item Attribute",
      "item_group": "Pakaian",
      "is_stock_item": 1,
      "is_sales_item": 1,
      "is_purchase_item": 1,
      "stock_uom": "Pcs",
      "attributes": [ { "attribute": "Ukuran", "attribute_value": "M" } ]
    }
  }'
```

> **Pre-check opsional:** gunakan `erpnext.controllers.item_variant.get_variant` dengan
> `{"template": "KAOS-POLOS", "args": {"Ukuran": "M"}}` — `message` berisi `name` varian yang sudah
> ada (kalau belum ada → `null`/kosong), untuk menghindari `ItemVariantExistsError`.

**Langkah 3b (alternatif) — Buat varian manual (`insert` dengan `variant_of`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_code": "KAOS-POLOS-L",
      "item_name": "Kaos Polos-L",
      "item_group": "Pakaian",
      "is_stock_item": 1,
      "is_sales_item": 1,
      "is_purchase_item": 1,
      "stock_uom": "Pcs",
      "variant_of": "KAOS-POLOS",
      "attributes": [ { "attribute": "Ukuran", "attribute_value": "L" } ]
    }
  }'
```

> Backend memvalidasi: template harus `has_variants=1`, atribut harus valid untuk template, dan
> kombinasi atribut tidak boleh menghasilkan varian yang sudah ada
> (*"Item variant {x} exists with same attributes"* — `ItemVariantExistsError`).

**Langkah 4 — Verifikasi daftar varian dari template:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","variant_of"],
    "filters": [["variant_of","=","KAOS-POLOS"]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "name": "KAOS-POLOS-L", "item_name": "Kaos Polos-L", "variant_of": "KAOS-POLOS" },
    { "name": "KAOS-POLOS-M", "item_name": "Kaos Polos-M", "variant_of": "KAOS-POLOS" },
    { "name": "KAOS-POLOS-S", "item_name": "Kaos Polos-S", "variant_of": "KAOS-POLOS" }
  ]
}
```

> **Aturan penting varian:**
> - `stock_uom` varian harus sama dengan template (kecuali `Item Variant Settings → Allow Different UOM`).
> - `item_code`/`item_name` varian dibangun dari `abbr`; bila `abbr` diubah di master Item Attribute,
>   backend **otomatis me-rename** item_code varian terkait.
> - Template **tidak boleh punya stok/transaksi** — semua transaksi memakai varian.
> - `has_variants` pada template yang sudah dipakai varian **tidak boleh di-nonaktifkan** (ada
>   validasi terkait).

### 4.3 READ (satu record) & total count

**Langkah 1 — Ambil detail Item (`frappe.client.get`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "name": "MIN-001"
  }'
```

> Respons `message` berisi seluruh field Item (seperti respons CREATE), termasuk child table
> `uoms`, `barcodes`, `attributes`, `reorder_levels`, `item_defaults`, `taxes`. (Catatan: `uoms`
> pada respons menampilkan baris yang tersimpan — baris `stock_uom` dengan `conversion_factor=1`
> otomatis ditambahkan backend bila belum ada.)

**Total count — `frappe.client.get_count`** (untuk pagination / lazy loading):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_count \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "filters": [["is_sales_item","=",1],["disabled","=",0]]
  }'
```

**Contoh respons (HTTP 200):**

```json
{ "message": 128 }
```

> `message` = jumlah record yang cocok → dipakai menghitung total halaman pada lazy loading (§4.4).

### 4.4 READ (daftar) — `frappe.client.get_list`

```bash
# Daftar — halaman 1, produk aktif yang bisa dijual
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","item_group","stock_uom","image","standard_rate","disabled"],
    "filters": [["disabled","=",0],["is_sales_item","=",1]],
    "order_by": "name asc",
    "limit_start": 0,
    "limit_page_length": 50
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "name": "MIN-001", "item_name": "Air Mineral 600ml", "item_group": "Minuman",
      "stock_uom": "Pcs", "image": "/files/min-001.jpg", "standard_rate": 5000, "disabled": 0 }
  ]
}
```

> Halaman berikutnya naikkan `limit_start` kelipatan `50`; `limit_page_length=0` = ambil **semua**
> record. Filter umum: `disabled=0` (aktif), `is_sales_item=1`, `is_stock_item=1`, `has_variants=0`
> (non-template), `variant_of=<template>` (daftar varian), `item_group=<leaf>`.

### 4.5 UPDATE — `frappe.client.save` & `frappe.client.set_value`

Update memakai `save`: kirim dokumen (hasil `frappe.client.get` yang dimodifikasi); `name` ada di
body. Child table (`uoms`, `barcodes`, `attributes`, `reorder_levels`, `item_defaults`, `taxes`)
berlaku **replace-all** — kirim seluruh baris yang diinginkan.

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "name": "MIN-001",
      "item_code": "MIN-001",
      "item_name": "Air Mineral 600ml",
      "item_group": "Minuman",
      "is_stock_item": 1,
      "is_sales_item": 1,
      "is_purchase_item": 1,
      "stock_uom": "Pcs",
      "purchase_uom": "Dus",
      "uoms": [
        { "uom": "Pcs", "conversion_factor": 1 },
        { "uom": "Dus", "conversion_factor": 12 }
      ],
      "image": "/files/min-001.jpg",
      "standard_rate": 5500
    }
  }'
```

**Contoh respons (HTTP 200):** objek `message` berisi dokumen terbaru (field berubah).

> **Catatan:**
> - `save` membangun ulang dokumen dari dict — kirim dokumen yang konsisten/lengkap (idealnya hasil
>   GET yang diubah). Child table bersifat replace-all.
> - Ubah `standard_rate` di sini **tidak** otomatis mengubah Item Price yang sudah ada (hanya saat
>   CREATE). Pengelolaan harga: [prd_item_price.md](./prd_item_price.md).
> - Ubah `item_group` → pindah kategori produk (tidak ada efek samping stok).
> - Ubah `stock_uom` pada Item ber-stok **ditolak backend**.

**Perubahan kecil — `frappe.client.set_value`** (lebih aman untuk satu-dua field):

```bash
# Non-aktifkan item
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "name": "MIN-001",
    "fieldname": { "disabled": 1 }
  }'
```

```bash
# Pindahkan item ke group lain / ubah harga standar
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "name": "MIN-001",
    "fieldname": { "item_group": "Air Mineral", "standard_rate": 6000 }
  }'
```

### 4.6 Barcode — CRUD

Barcode disimpan di child table **`barcodes`** (doctype `Item Barcode`, tabel `tabItem Barcode`),
satu Item boleh punya **banyak barcode** (mis. per UOM). Backend memvalidasi barcode **unik global**.

**CREATE — sertakan `barcodes` pada `frappe.client.insert`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_code": "MIN-001",
      "item_group": "Minuman",
      "stock_uom": "Pcs",
      "barcodes": [
        { "barcode": "8991234567890", "barcode_type": "EAN-13" },
        { "barcode": "8991234567891", "barcode_type": "EAN-13", "uom": "Dus" }
      ]
    }
  }'
```

**READ — `barcodes` ikut dalam respons `frappe.client.get`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Item", "name": "MIN-001" }'
```

**Contoh respons (bagian `barcodes`):**

```json
{
  "message": {
    "name": "MIN-001",
    "barcodes": [
      { "name": "xyz001", "barcode": "8991234567890", "barcode_type": "EAN-13", "uom": "" },
      { "name": "xyz002", "barcode": "8991234567891", "barcode_type": "EAN-13", "uom": "Dus" }
    ]
  }
}
```

**UPDATE (tambah/hapus/ganti) — `frappe.client.save` dengan `barcodes` replace-all:**

```bash
# Tambah barcode baru + hapus barcode lama: kirim seluruh baris yang diinginkan
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "name": "MIN-001",
      "item_code": "MIN-001",
      "item_group": "Minuman",
      "stock_uom": "Pcs",
      "barcodes": [
        { "barcode": "8991234567890", "barcode_type": "EAN-13" },
        { "barcode": "8991234567893", "barcode_type": "EAN-13", "uom": "Pcs" }
      ]
    }
  }'
```

> **Hapus satu barcode:** cukup tidak sertakan baris tersebut saat `save` (replace-all). Tidak ada
> endpoint khusus per baris child table.
>
> **Cari Item dari barcode (scan):**
> ```bash
> curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
>   -H 'Authorization: Bearer <access_token>' \
>   -H 'Content-Type: application/json' \
>   -d '{
>     "doctype": "Item",
>     "fields": ["name","item_name","stock_uom"],
>     "filters": [["barcodes.barcode","=","8991234567890"]],
>     "limit_page_length": 1
>   }'
> ```
> `barcodes.barcode` adalah **child filter** — didukung `frappe.client.get_list` (join child table).

### 4.7 Foto produk — CRUD (`image` + multi foto di `tabFile`)

**Model penyimpanan:**
- **Foto utama** → field `image` pada Item, disimpan sebagai **string URL file** (mis. `/files/min-001.jpg`).
  Diisi lewat `insert`/`save`/`set_value` seperti field biasa.
- **Foto tambahan (multi)** → doctype **`File`** (tabel **`tabFile`**), tiap file adalah record
  ter-link ke Item via `attached_to_doctype="Item"` + `attached_to_name=<item_code>`. Satu record
  `File` menunjuk **satu** Item; file fisik yang sama bisa direferensikan oleh **banyak record `File`**
  (dipakai untuk "tag" foto ke varian, lihat di bawah).

**CREATE — upload foto tambahan ke `tabFile` (`frappe.client.attach_file`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.attach_file \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "docname": "MIN-001",
    "filename": "foto-belakang.jpg",
    "filedata": "<isi file dalam base64>",
    "is_private": 0
  }'
```

> - `filedata` = isi file (base64); `is_private=0` → file publik (`/files/...`), `1` → privat.
> - Alternatif: bila file sudah ter-upload di server, kirim `file_url` (mis. `/files/foto-belakang.jpg`)
>   tanpa `filedata`.
> - Respons `message` berisi dokumen `File` (termasuk `name`, `file_name`, `file_url`).

**Foto per varian — tag foto template ke varian (Pendekatan A):**

Satu file fisik cukup di-upload **sekali** (mis. ke template), lalu dibuat **record `File` berulang
per varian** yang memakainya — record baru menunjuk `file_url` yang sama **tanpa re-upload isi file**.
Dengan begitu tiap varian (Item terpisah) punya daftar foto sendiri di `tabFile`.

Contoh kasus: 5 foto di-upload ke template `KAOS-POLOS` → **3 foto** untuk varian `KAOS-POLOS-M`,
**2 foto** untuk varian `KAOS-POLOS-L`.

**Langkah 1 — Upload 5 foto ke template (`frappe.client.attach_file`, ulangi per foto):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.attach_file \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "docname": "KAOS-POLOS",
    "filename": "kaos-depan.jpg",
    "filedata": "<isi file dalam base64>",
    "is_private": 0
  }'
```

**Langkah 2 — Tag foto ke varian: buat record `File` baru yang menunjuk `file_url` yang sama
(`frappe.client.insert` pada doctype `File`):**

```bash
# 3 foto untuk varian 1 — contoh satu record; ulangi untuk tiap foto yang ditag
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "File",
      "file_name": "kaos-depan.jpg",
      "file_url": "/files/kaos-depan.jpg",
      "attached_to_doctype": "Item",
      "attached_to_name": "KAOS-POLOS-M",
      "is_private": 0
    }
  }'
```

> Lakukan hal yang sama untuk varian 2 (`attached_to_name: "KAOS-POLOS-L"`) dengan `file_url` 2 foto
> lainnya. Isi file **tidak di-upload ulang** — record `File` hanyalah metadata yang menunjuk file
> yang sama. `file_url` bisa diambil dari respons Langkah 1.

**READ — ambil foto per varian (`frappe.client.get_list`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "File",
    "fields": ["name","file_name","file_url","file_size","is_private"],
    "filters": [
      ["attached_to_doctype","=","Item"],
      ["attached_to_name","=","KAOS-POLOS-M"]
    ],
    "order_by": "creation asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200) — 3 foto untuk varian 1:**

```json
{
  "message": [
    { "name": "f1", "file_name": "kaos-depan.jpg", "file_url": "/files/kaos-depan.jpg", "file_size": 102400, "is_private": 0 },
    { "name": "f2", "file_name": "kaos-samping.jpg", "file_url": "/files/kaos-samping.jpg", "file_size": 98304, "is_private": 0 },
    { "name": "f3", "file_name": "kaos-belakang.jpg", "file_url": "/files/kaos-belakang.jpg", "file_size": 110592, "is_private": 0 }
  ]
}
```

> **Catatan hapus (konsistensi):** satu file fisik bisa direferensikan oleh beberapa record `File`
> (template + beberapa varian). Menghapus record dari satu varian **tidak menghapus file fisik**
> selama masih ada record lain yang memakainya. Untuk benar-benar menghapus file fisik, hapus **semua
> record `File`** dengan `file_url` tersebut (di seluruh varian/template yang memakainya).

**READ — daftar semua foto Item dari `tabFile`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "File",
    "fields": ["name","file_name","file_url","file_size","is_private"],
    "filters": [
      ["attached_to_doctype","=","Item"],
      ["attached_to_name","=","MIN-001"]
    ],
    "order_by": "creation asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "name": "a1b2c3d4e5", "file_name": "foto-belakang.jpg",
      "file_url": "/files/foto-belakang.jpg", "file_size": 245760, "is_private": 0 }
  ]
}
```

**UPDATE — ganti foto:** upload file baru (`attach_file`), lalu **hapus** file lama
(`frappe.client.delete`). File di ERPNext tidak mengubah isi (binary) — ganti = upload baru + hapus lama.

**DELETE — hapus foto dari `tabFile`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.delete \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "File",
    "name": "a1b2c3d4e5"
  }'
```

> **Hapus foto utama:** set `image` menjadi `""` via `set_value` (nilai string pada field `image`,
> bukan hapus File).

### 4.8 Non-aktif / hapus

**Aturan bisnis (wajib diikuti frontend):**

1. **Non-aktif = soft-delete.** Set `disabled=1` via `frappe.client.set_value` (contoh di §4.5).
   Item non-aktif tidak muncul di transaksi baru; riwayat tetap tersimpan. Aktifkan kembali dengan
   `disabled=0`.
2. **Template mengikuti varian-nya.** Jika yang dinonaktifkan adalah **template** (`has_variants=1`),
   maka **semua varian-nya** (`variant_of=<template>`) **wajib ikut dinonaktifkan** dalam proses yang
   sama. (Menonaktifkan varian tersendiri tidak memengaruhi template.)
3. **Cek stok sebelum non-aktif.** Untuk **semua item dalam grup** (template + seluruh varian-nya,
   atau item tunggal), frontend wajib mengecek stok `actual_qty > 0` pada `tabBin`. Bila ada yang
   stoknya > 0, **backend mengirim list** item+warehouse yang stoknya > 0 (Langkah 2).
4. **Konfirmasi user.** Jika ada stok > 0, frontend wajib bertanya ke user:
   *"Stok produk berikut akan dibuat menjadi 0: `<item1>`, `<item2>`, ... — lanjutkan?"*
   - **YA** → panggil **API zero stok** (Stock Reconciliation, Langkah 4), **tunggu sampai submit
     sukses**, baru lanjut non-aktif (Langkah 5).
   - **TIDAK** → **batalkan seluruh proses non-aktif** (tidak ada item yang dinonaktifkan).
5. Jika **tidak ada** stok > 0 → langsung non-aktif (Langkah 5) tanpa konfirmasi.

> **Urutan (kunci):** zero stok **selalu sebelum** `disabled=1`.

**Langkah 1 — Kumpulkan grup item (template + varian):**

Jika item tunggal → grup = `[item_code]`. Jika template → ambil semua varian-nya:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name"],
    "filters": [["variant_of","=","KAOS-POLOS"]],
    "limit_page_length": 0
  }'
```

Grup non-aktif = template + seluruh `name` hasil ini.

**Langkah 2 — Cek stok > 0 (backend mengirim list produk ber-stok):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Bin",
    "fields": ["item_code","warehouse","actual_qty","stock_uom"],
    "filters": [
      ["item_code","in",["KAOS-POLOS","KAOS-POLOS-S","KAOS-POLOS-M","KAOS-POLOS-L"]],
      ["actual_qty",">",0]
    ],
    "order_by": "item_code asc, warehouse asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "item_code": "KAOS-POLOS-M", "warehouse": "Toko Cikarang - PTMJ", "actual_qty": 12, "stock_uom": "Pcs" },
    { "item_code": "KAOS-POLOS-L", "warehouse": "Gudang Pusat - PTMJ", "actual_qty": 30, "stock_uom": "Pcs" }
  ]
}
```

> - Respons berupa **baris per (item, warehouse)** dengan `actual_qty > 0`. Frontend mengagregasi
>   total per `item_code` untuk dialog konfirmasi (mis. `KAOS-POLOS-M` = 12 Pcs, `KAOS-POLOS-L` = 30 Pcs).
> - `message` kosong (`[]`) → tidak ada stok → **lewati Langkah 3–4**, langsung Langkah 5.

**Langkah 3 — Konfirmasi user (frontend):**

- Jika `message` dari Langkah 2 **tidak kosong**, tampilkan daftar item + total stok-nya dan tanya:
  *"Stok produk berikut akan dibuat menjadi 0 — lanjutkan?"* dengan tombol **Ya / Batal**.
- **Batal** → hentikan proses (tidak ada item yang dinonaktifkan).
- **Ya** → lanjut Langkah 4 (zero stok) → Langkah 5 (non-aktif).

**Langkah 4 — Zero stok (API Stock Reconciliation):**

Mengosongkan stok memakai doctype **`Stock Reconciliation`** (tabel `tabStock Reconciliation`):
buat draft lalu **submit** — submit menulis Stock Ledger sehingga `actual_qty` di `tabBin` menjadi 0.
Buat **satu dokumen** berisi **semua baris** `{item_code, warehouse}` dari Langkah 2 dengan `qty: 0`.

**4a. Buat draft (`frappe.client.insert`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Stock Reconciliation",
      "company": "PT Maju Jaya",
      "purpose": "Stock Reconciliation",
      "posting_date": "2026-09-02",
      "posting_time": "09:30:00",
      "set_posting_time": 1,
      "items": [
        { "item_code": "KAOS-POLOS-M", "warehouse": "Toko Cikarang - PTMJ", "qty": 0 },
        { "item_code": "KAOS-POLOS-L", "warehouse": "Gudang Pusat - PTMJ", "qty": 0 }
      ]
    }
  }'
```

> - `company` wajib; `expense_account` / `cost_center` **diisi otomatis backend** dari Company bila
>   kosong (untuk perpetual inventory, `expense_account` diambil dari `Company.stock_adjustment_account`).
> - `qty: 0` → stok disesuaikan ke 0. Bila `qty` diisi tanpa `valuation_rate`, backend memakai nilai
>   stok/Item saat ini. Baris yang tidak berubah diabaikan backend (`remove_items_with_no_change`).

**4b. Submit (`frappe.client.submit`)** — pakai `name` dari respons 4a:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.submit \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Stock Reconciliation",
      "name": "MAT-RECO-00001"
    }
  }'
```

> **PENTING:** proses non-aktif **tidak boleh dilanjutkan** sebelum submit Stock Reconciliation
> sukses (HTTP 200, respons ber `docstatus: 1`). Setelah sukses, `actual_qty` item-item tersebut di
> `tabBin` menjadi 0 — boleh diverifikasi ulang dengan Langkah 2 (hasil harus `[]`).
> Detail mekanisme Stock Reconciliation (akun, posting date, reposting) ada di **PRD stok terpisah** (§9).

**Langkah 5 — Non-aktifkan seluruh item grup (`frappe.client.set_value disabled=1`):**

Non-aktifkan **varian terlebih dahulu, lalu template** (agar tidak ada varian aktif di bawah template
non-aktif). Bila item tunggal, cukup item tersebut.

```bash
# per item — ulangi untuk tiap varian, lalu template
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "name": "KAOS-POLOS-M",
    "fieldname": { "disabled": 1 }
  }'
```

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "name": "KAOS-POLOS",
    "fieldname": { "disabled": 1 }
  }'
```

**Hapus permanen — `frappe.client.delete`** (hanya bila benar-benar diperlukan):
- `name` di body; **diblokir** bila Item sudah dipakai transaksi/stok (`LinkExistsError` /
  `StockExistsForTemplateError`), atau masih punya child table yang di-referensikan.
- Karena data master "jangan hapus" + item bisa punya riwayat, **jangan hapus Item yang sudah
  dipakai** — cukup non-aktifkan.
- Khusus **template** (`has_variants=1`): tidak bisa dihapus bila masih punya varian.

### 4.9 tabBin — info stok per (Item, Warehouse)

**Bin** (doctype `Bin`, tabel **`tabBin`**) adalah **snapshot stok** — satu record per kombinasi
**`item_code` + `warehouse`** (unik). **Tidak diedit manual oleh frontend** — nilainya dikelola
otomatis oleh Stock Ledger Entry & reposting. Detail mekanisme stok ada di **PRD stok terpisah** (§9);
di sini cukup cara membacanya.

Field utama `tabBin`:

| Field | Arti |
|---|---|
| `item_code`, `warehouse`, `stock_uom` | Kunci record + satuan stok (diambil dari Item). |
| `actual_qty` | Qty fisik tersedia di gudang saat ini. |
| `ordered_qty` | Qty masih dalam pesanan beli (belum diterima). |
| `indented_qty` | Qty yang diminta (Material Request, belum jadi pesanan). |
| `planned_qty` | Qty direncanakan produksi. |
| `reserved_qty` | Qty ter-reservasi oleh Sales Order / Delivery Note. |
| `reserved_qty_for_production` | Qty cadangan untuk Work Order. |
| `reserved_qty_for_sub_contract` | Qty cadangan untuk subkontrak. |
| `reserved_qty_for_production_plan` | Qty cadangan dari Production Plan. |
| `projected_qty` | **Qty proyeksi** (bisa dipakai untuk janji pengiriman). Rumus: `actual + ordered + indented + planned − reserved − reserved_production − reserved_subcontract − reserved_production_plan`. |
| `stock_value` | Nilai total stok = `actual_qty × valuation_rate`. |
| `valuation_rate` | Nilai per unit saat ini. |

**READ — baca stok Item di semua warehouse (`frappe.client.get_list`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Bin",
    "fields": ["item_code","warehouse","actual_qty","projected_qty","ordered_qty","reserved_qty","stock_value","valuation_rate"],
    "filters": [["item_code","=","MIN-001"]],
    "order_by": "warehouse asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "item_code": "MIN-001", "warehouse": "Gudang Pusat - PTMJ", "actual_qty": 120,
      "projected_qty": 132, "ordered_qty": 24, "reserved_qty": 12,
      "stock_value": 420000, "valuation_rate": 3500 }
  ]
}
```

> **Baca stok di satu warehouse:** tambahkan filter `["warehouse","=","Gudang Pusat - PTMJ"]`.
> **Alternatif method whitelisted:** `erpnext.stock.get_item_details.get_bin_details`
> (`{item_code, warehouse, company}` → `actual_qty`/`projected_qty`/`reserved_qty`) dan
> `erpnext.stock.get_item_details.get_projected_qty` (`{item_code, warehouse}`).
> Bin **tidak boleh dibuat/diubah lewat API aplikasi** — cukup dibaca.

---

## 5. UOM Conversion — contoh kasus & CRUD

### 5.1 Konsep dua lapis

1. **Per Item (utama)** — child table **`uoms`** (doctype `UOM Conversion Detail`, tabel
   `tabUOM Conversion Detail`, parenttype `Item`): pasangan `uom` + `conversion_factor` relatif ke
   `stock_uom`. Baris `stock_uom` **wajib `conversion_factor=1`**; `uom` tidak boleh duplikat.
2. **Global (pendukung)** — doctype **`UOM Conversion Factor`** (tabel `tabUOM Conversion Factor`):
   pasangan `from_uom`→`to_uom` + `value` per kategori. Dipakai mengisi `conversion_factor` otomatis
   (method `get_uom_conv_factor`) bila baris `uoms` dikirim tanpa faktor.

**Contoh kasus:** `stock_uom="Pcs"`. `uoms` berisi `Pcs=1`, `Dus=12`, `Karton=120`. Saat transaksi
beli 5 `Dus` → **stok bertambah 60 Pcs** (5 × 12). Bila ada harga beli per Dus, harga per Pcs dihitung
lewat `conversion_factor`. Global: `get_uom_conv_factor("Kg","Gram")` → `1000`.

### 5.2 CRUD `uoms` (per Item)

**CREATE / UPDATE — sertakan `uoms` pada `insert`/`save` (replace-all):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_code": "RM-002",
      "item_group": "Bahan Baku",
      "stock_uom": "Pcs",
      "uoms": [
        { "uom": "Pcs", "conversion_factor": 1 },
        { "uom": "Dus", "conversion_factor": 12 },
        { "uom": "Karton", "conversion_factor": 120 }
      ]
    }
  }'
```

> **READ** — `uoms` ikut dalam respons `frappe.client.get` (semua baris tersimpan; baris `stock_uom`
> dengan faktor 1 otomatis ditambahkan backend bila belum ada). Bila `conversion_factor` dikosongkan,
> backend mencoba mengisinya dari `UOM Conversion Factor` global.

**Cek faktor konversi secara global (`get_uom_conv_factor`):**

```bash
curl -X POST https://site-anda.com/api/method/erpnext.stock.doctype.item.item.get_uom_conv_factor \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "uom": "Kg",
    "stock_uom": "Gram"
  }'
```

**Contoh respons (HTTP 200):** `{ "message": 1000 }`

### 5.3 CRUD `UOM Conversion Factor` (master global, opsional)

Untuk menambah pasangan konversi global (mis. `Karton`→`Dus`), pakai `frappe.client.*` biasa:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "UOM Conversion Factor",
      "category": "Quantity",
      "from_uom": "Dus",
      "to_uom": "Karton",
      "value": 10
    }
  }'
```

> `category` = kategori UOM (mis. `Mass`, `Quantity`, `Length`, `Volume`). Pencarian / daftar lewat
> `frappe.client.get_list`. Pada umumnya aplikasi **cukup memakai `uoms` per Item** — master global
> hanya dibutuhkan bila ingin faktor konversi dipakai lintas item.

### 5.4 Validasi backend terkait UOM

| Kondisi | Error |
|---|---|
| `conversion_factor` baris `stock_uom` ≠ 1 | `ValidationError` — *"Conversion factor for default Unit of Measure must be 1"* |
| `uom` duplikat dalam `uoms` | `ValidationError` — *"Unit of Measure {0} has been entered more than once in Conversion Factor Table"* |
| Ubah `stock_uom` pada item ber-stok | `ValidationError` — stok tidak bisa diubah satuan dasarnya |

---

## 6. GET pendukung UI

Semua pakai `frappe.client.get_list` (POST, body filters) kecuali disebut lain.

### 6.1 Item Group (dropdown `item_group`) — hanya leaf

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

> Detail & aturan tree: [prd_item_group.md §5.2](./prd_item_group.md). Hanya **leaf** yang boleh
> dipilih sebagai `item_group` produk yang ditransaksikan.

### 6.2 UOM (dropdown `stock_uom` / `purchase_uom` / `weight_uom`)

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "UOM",
    "fields": ["name","symbol"],
    "filters": [["enabled","=",1]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

### 6.3 Brand (dropdown `brand`)

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Brand",
    "fields": ["name"],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

### 6.4 Item Attribute & nilai (dropdown varian)

```bash
# Daftar master attribute + nilai-nya
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Attribute",
    "fields": ["name","item_attribute_values"],
    "filters": [["disabled","=",0]],
    "limit_page_length": 0
  }'
```

```bash
# Autocomplete nilai attribute (untuk isi `attribute_value` di varian)
curl -X POST https://site-anda.com/api/method/erpnext.stock.doctype.item.item.get_item_attribute \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "parent": "Ukuran",
    "attribute_value": "M"
  }'
```

### 6.5 Company / Warehouse / Cost Center / Account (dropdown `item_defaults`)

Untuk baris `item_defaults` per company (mis. `default_warehouse`):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "fields": ["name"],
    "filters": [["company","=","PT Maju Jaya"],["disabled","=",0],["is_group","=",0]],
    "limit_page_length": 0
  }'
```

> Warehouse: [prd_warehouse.md](../setup/prd_warehouse.md). Company / Cost Center / Account mengikuti
> pola yang sama dengan filter `company` (lihat [prd_item_group.md §5.3/§5.5](./prd_item_group.md)).

### 6.6 Item Tax Template (dropdown `taxes`, opsional)

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Tax Template",
    "fields": ["name","company"],
    "filters": [["company","=","PT Maju Jaya"],["disabled","=",0]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

### 6.7 Menyaring daftar Item per Item Group (filter `item_group`)

Untuk menampilkan/menyaring Item berdasarkan kategori (mis. dropdown produk per grup / filter daftar),
pakai field `item_group` — field **langsung** di doctype `Item` (berlaku juga di `get_count` /
`get_value`). Item hanya menunjuk ke Item Group **leaf** (`is_group=0`), jadi:

```bash
# (a) Item pada 1 leaf group — langsung
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","item_group","stock_uom"],
    "filters": [["item_group","=","Minuman"],["is_stock_item","=",1],["disabled","=",0]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

```bash
# (b) Item di bawah group induk + seluruh sub-group — 2 langkah.
#     Langkah 1: resolve leaf keturunan group tsb (rentang lft/rgt subtree; alternatif:
#     frappe.desk.treeview.get_children, lihat prd_item_group.md §4 & §5.1). Contoh rentang
#     utk subtree "Food & Beverage" (angka lft/rgt diambil dari record group tsb):
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Group",
    "fields": ["name","is_group"],
    "filters": [["lft",">=",4],["rgt","<=",11]],
    "order_by": "lft asc",
    "limit_page_length": 0
  }'
```

```bash
#     Langkah 2: filter Item dengan `item_group in [ ... ]` (daftar nama hasil Langkah 1)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","item_group"],
    "filters": [["item_group","in",["Minuman","Air Mineral"]],["is_stock_item","=",1]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

> **Penting:** `item_group = <group induk>` (node `is_group=1`) **mengembalikan kosong** — Item tidak
> pernah menunjuk node group, hanya leaf. Bila UI memakai tree picker, gunakan pola (b). Detail tree
> Item Group & aturan leaf: [prd_item_group.md §4 & §5](./prd_item_group.md).

---

## 7. Penanganan error umum

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | Role tidak punya akses (bukan `Item Manager` utk tulis) | `{"message": "Not permitted"}` |
| 404 | Resource tidak ditemukan | `{"exc_type":"DoesNotExistError","message":"Resource Not Found"}` |
| 417 | Field wajib kosong (`reqd`) | `{"exc_type":"MandatoryError","message":"item_group is mandatory"}` / `"stock_uom is mandatory"` |
| 417 | Duplikat `item_code` (`unique`) | `{"exc_type":"DuplicateEntryError","message":"... already exists"}` |
| 417 | Barcode duplikat global / tipe invalid | `{"exc_type":"DuplicateEntryError"|"InvalidBarcode","message":"..."}` |
| 417 | Varian sudah ada dgn kombinasi atribut sama | `{"exc_type":"ItemVariantExistsError","message":"Item variant KAOS-POLOS-M exists with same attributes"}` |
| 417 | Nilai atribut tidak valid utk template | `{"exc_type":"InvalidItemAttributeValueError","message":"..."}` |
| 417 | Template tidak boleh punya stok / `has_variants` salah | `{"exc_type":"ValidationError","message":"..."}` |
| 417 | Ubah `stock_uom` pada item ber-stok | `{"exc_type":"ValidationError","message":"..."}` |
| 417 | `conversion_factor` stock_uom ≠ 1 / `uoms` duplikat | `{"exc_type":"ValidationError","message":"..."}` (lihat §5.4) |
| 417 | Hapus Item yang sudah dipakai transaksi/stok | `{"exc_type":"LinkExistsError","message":"Cannot delete because of linked records"}` |
| 417 | `attributes` kosong saat `has_variants=1` | `{"exc_type":"ValidationError","message":"Attribute table is mandatory"}` |

> **Catatan:** karena seluruh pemanggilan memakai `/api/method/...`, hasil sukses dibungkus di
> `message` (bukan `data`). Body error tetap berbentuk `{ "exc_type": ..., "exception": ..., "message": ... }`.
> Method `frappe.client.get` dengan `name` yang tidak ada → 404 `DoesNotExistError`.

---

## 8. Koleksi Postman

Seluruh pemanggilan di atas tersedia dalam **satu koleksi Postman** yang siap import:
`docs/postman/postman_erpnext_api.json` (Collection v2.1). Koleksi ini berisi seluruh modul API
ERPNext (OAuth 2.0 + Supplier + Customer + Contact + Address + Lead + Employee + User + Warehouse
+ POS Profile + Item Group + **Item**).

Folder **`11. Item`** berisi **39 request** yang mencakup:
- Pre-check & CREATE (minimum, lengkap, jasa, bahan baku) — `11.1`, `11.1b`–`11.3b`
- CREATE varian: master Item Attribute, template, `create_variant` + simpan, pre-check `get_variant`,
  manual, daftar varian — `11.4`–`11.8`
- READ single / count / list — `11.9`–`11.11`
- UPDATE (`save`) & `set_value` — `11.12`–`11.13`
- Barcode: create / read / update / scan — `11.14`–`11.17`
- Foto: upload (`attach_file`), tag ke varian (`insert` File), GET per varian, hapus — `11.18`–`11.21`
- Non-aktif (§4.8): GET varian template, cek stok > 0 (`tabBin`), zero stok (insert + submit Stock
  Reconciliation), non-aktif varian & template — `11.22`–`11.25b`
- tabBin (stok) — `11.26`
- UOM: `uoms`, `get_uom_conv_factor`, `UOM Conversion Factor` global — `11.27`–`11.29`
- GET pendukung UI: Item Group leaf, UOM, Brand, Item Attribute, Warehouse/Company, Item Tax
  Template — `11.30`–`11.35`

**Variabel yang perlu diisi** (Collection Variables):
- `item_id` / `item_code` — name hasil CREATE (mis. `MIN-001`)
- `item_template` — template varian (mis. `KAOS-POLOS`)
- `item_variant` / `item_variant_2` — varian (mis. `KAOS-POLOS-M`, `KAOS-POLOS-L`)
- `item_attribute` / `item_attribute_value` — master atribut & nilainya (mis. `Ukuran`, `M`)
- `item_group` — leaf Item Group (mis. `Minuman`)
- `item_barcode` — barcode (mis. `8991234567890`)
- `file_name` / `file_url` / `file_id` — foto (dari respons `attach_file` / `insert` File)
- `stock_reco_id` — name Stock Reconciliation (mis. `MAT-RECO-00001`)
- `company` / `warehouse` — company & warehouse default (mis. `PT Maju Jaya`, `Toko Cikarang - PTMJ`)

Cara pakai sama dengan folder lain: isi variabel di atas, jalankan folder `0. OAuth 2.0` (atau
Get New Access Token), lalu jalankan request pada folder `11. Item`. Request CREATE otomatis
menyimpan `name` hasil insert ke variabel `item_id` / `item_template` / `item_variant` lewat
test script.

---

## 9. Catatan & PRD lanjutan

1. **Stok & Price List — PRD terpisah (menyusul).** Detail berikut **tidak** didokumentasikan di file
   ini dan akan dibuat sebagai file PRD sendiri:
   - **Stok:** Opening Stock, Stock Ledger Entry, Stock Entry (Receipt/Issue/Transfer), Stock
     Reconciliation (mekanisme lanjutan: akun, posting date, reposting), **mekanisme reorder &
     notifikasi re-stock** — termasuk cara mengisi stok awal saat item baru dibuat (`opening_stock` +
     `valuation_rate`, atau via Stock Entry). Di file ini: info baca `tabBin` (§4.9), **alur zero
     stok via Stock Reconciliation saat non-aktif** (§4.8), dan **penyimpanan ambang stok minimum
     `reorder_levels`** (ringkasan §2 & contoh CREATE §4.1).
   - **Price List / Item Price:** **sudah tersedia** → [prd_item_price.md](./prd_item_price.md)
     (doctype `Price List` + `Item Price`): pengelolaan `standard_rate`, harga per Price List
     (selling/buying), harga per UOM, harga khusus customer/supplier, masa berlaku, bulk update, dan
     catatan bahwa `standard_rate` saat CREATE otomatis membuat Item Price (§4.9 file tersebut).
   > Referensi silang pada file ini: **"PRD Price List"** → [prd_item_price.md](./prd_item_price.md)
   > (sudah ada). **"PRD stok"** → `prd_stock.md` (belum dibuat, folder yang sama) — bagian terkait
   > belum tersedia.
2. **File terkait:** **Price List / Item Price [prd_item_price.md](./prd_item_price.md)**,
   **Pricing Rule (diskon & promo) [prd_item_pricing_rule.md](./prd_item_pricing_rule.md)**,
   **Product Bundle (paket statis) [prd_item_product_bundle.md](./prd_item_product_bundle.md)**,
   **Dynamic Product Bundle (paket dengan isian dipilih kasir)
   [prd_item_dynamic_product_bundle.md](./prd_item_dynamic_product_bundle.md)**,
   Item Group [prd_item_group.md](./prd_item_group.md), Warehouse
   [prd_warehouse.md](../setup/prd_warehouse.md), OAuth [prd_oauth.md](../prd_oauth.md).
   > Field Item yang dipakai Pricing Rule: `max_discount` (batas diskon per item) dan
   > `item_group`/`brand` (dasar `apply_on` pada Pricing Rule).
   > Field Item untuk paket dinamis: `is_dynamic_product_bundle` (custom field `baseapp`) —
   > lihat [prd_item_dynamic_product_bundle.md](./prd_item_dynamic_product_bundle.md).
