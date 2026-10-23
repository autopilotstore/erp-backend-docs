# PRD — Laporan Daftar Item & Stok per Gudang ("Stock Report") — REST API `frappe.client.*` (ERPNext / Frappe)

> Dokumen spesifikasi **laporan daftar item + stok** untuk tim **UI/Frontend**, memakai **API bawaan
> Frappe** (`frappe.client.*`) — **tanpa kode baru** di sisi backend (tidak ada app/method custom,
> tidak ada doctype/custom field baru, **tidak perlu** `bench migrate`).

- **Endpoint yang dipakai:** `POST /api/method/frappe.client.get_list` dan
  `POST /api/method/frappe.client.get_count` (API v1 — parameter di **body JSON**)
- **Doctype yang dibaca:** `Item`, `Item Group`, `Item Variant Attribute`, `Bin`, `Item Reorder`
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token (atau API key/secret)
- **Format respons:** JSON, dibungkus `"message"` (standar `/api/method/...`)
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (dev: `http://erpnext.localhost:8000`)
- **Periode:** **tidak ada** parameter tanggal — yang ditampilkan adalah **saldo stok saat ini**
  (`tabBin`). Untuk laporan periode (saldo awal/masuk/keluar) gunakan
  [`prd_stock_balance.md`](./prd_stock_balance.md).

> **Prasyarat:** app custom **`baseapp`** terpasang di site. Tiga hal bergantung padanya:
> filter wajib `is_product_bundle` & `is_dynamic_product_bundle` (custom field `baseapp`) dan kolom
> `max_stock_level` pada child `Item Reorder` (custom field `baseapp`). Lihat
> [`prd_item.md` §2.1 no. 9](./prd_item.md) & [`prd_baseapp.md`](../baseapp/prd_baseapp.md).

> **Sifat dokumen:** **read-only** — hanya pemanggilan **GET data** (list & count). Tidak ada
> CREATE/UPDATE/DELETE. Dropdown pendukung (warehouse, item group) memakai PRD masing-masing (§5.5).

> **Singkatnya:**
> - **Populasi baris = pasangan item × 1 warehouse dari `tabBin`** → item yang **belum terdaftar**
>   di gudang itu **tidak ditampilkan** (keputusan 2026-10-22). Item yang punya baris `Bin`
>   bersaldo `0` **tetap** tampil.
> - **Daftar item bisa disortir menurut kolom `Item`** (`item_name`, `item_group_name`, `name`, dst.)
>   **maupun menurut stok** (`actual_qty`, `projected_qty`) — tetapi **sumber paginasinya berbeda**
>   (§2.10): sortir kolom `Item` → paginasi di `Item`; sortir stok → paginasi di `Bin`.
> - Semua filter (6 filter wajib §2.1 + pencarian nama/atribut/item group) berlaku di **kedua** mode.
> - **9 filter kondisi stok** tersedia (stok kosong, minus, menyentuh min/maks, baru habis hari ini,
>   menjadi minus, stok berubah, dst.) → dihitung sebagai **cohort** lebih dulu (3 di server, 6 di
>   frontend/rumus), lalu diiris dengan filter wajib & filter user (§2.13).
> - Untuk katalog gudang yang kecil (±≤ 2.000 item), tersedia **Mode C** yang paling sederhana:
>   muat semua sekali, sortir & paginasi di frontend (§4.8).

---

## 1. Ruang lingkup

Ada **8 pola panggilan** (semua memakai 2 method bawaan yang sama). Pola mana yang dipakai
bergantung pada **kolom sortir** yang dipilih user (§2.10) dan **filter kondisi stok** yang aktif
(§2.13).

| Pola | Panggilan | Operasi | Kapan dipakai | § |
|---|---|---|---|---|
| **P1** | `frappe.client.get_list` → **`Bin`** | **Fondasi**: semua pasangan item × 1 warehouse (`limit_page_length: 0`) + stok (`actual_qty`, `projected_qty`) | **Selalu** (1× per gudang, sebelum daftar dimuat) | §2.2 / §4.1 |
| **P2** | `frappe.client.get_list` → **`Item`** | **Halaman daftar** terurut menurut **kolom `Item`**, dibatasi `["name","in",[kode P1]]` | Mode A (sortir `item_name`/`item_group_name`/`name`/…) — per halaman | §2.3 / §4.2 |
| **P3** | `frappe.client.get_count` → **`Item`** | **Total** untuk Mode A (filter identik P2) | Mode A — 1× per kombinasi filter | §2.4 |
| **P4** | `Item.get_list` (`fields:["name"]`, tanpa paginasi) → `Bin.get_list` (halaman terurut stok) → `Item.get_list` (detail halaman) | **Halaman daftar** terurut menurut **stok** | Mode B (sortir `actual_qty`/`projected_qty`) | §2.5 / §4.4 |
| **P5** | `frappe.client.get_count` → **`Bin`** | **Total** untuk Mode B | Mode B — 1× per kombinasi filter | §2.6 |
| **P6** | `frappe.client.get_list` → **`Item`** (`limit_page_length: 0`) | **Seluruh dataset gudang** untuk **Mode C** (sortir & paginasi di frontend) | Mode C — katalog kecil | §4.8 |
| **P7** | `frappe.client.get_list` → **`Item`** (+ nested `reorder_levels`) | **Level min/max per gudang** untuk filter **F2/F3/F5/F6** | Bila filter kondisi stok min/max dipakai | §2.13 |
| **P8** | `frappe.client.get_list` → **`Stock Ledger Entry`** | **Aktivitas stok hari ini** (`qty_after_transaction`, `actual_qty`) untuk **F4/F5/F6/F8/F9** | Bila filter kondisi stok berbasis "hari ini" dipakai | §2.13 |

> **Konvensi pemanggilan (penting):** semua request adalah **POST dengan body JSON**
> (`-H "Content-Type: application/json" -d '{...}'`). `filters` bertipe list-of-list, `fields`
> bertipe campuran (string + dict seperti `{"attributes":[...]}`), dan `item_code in [...]` bisa
> berisi ratusan kode → body JSON jauh lebih aman daripada query string. Semua parameter
> (`doctype`, `fields`, `filters`, `order_by`, dst.) dikirim di **body**, bukan di URL.

### 1.1 Alur per mode

```text
MODE A — sortir kolom Item (item_name, item_group_name, name, variant_of, ...)
  P1. Bin.get_list   {warehouse}                              -> fondasi (kode + stok, sekali)
  P2. Item.get_list  {filter wajib + filter user + name in [kode P1]}
                     order_by "item_name asc, name asc"
                     limit_start / limit_page_length          -> 1 halaman (terurut)
  P3. Item.get_count {filter IDENTIK P2}                      -> total (sekali per filter)
  stok tiap baris   -> diambil dari respons P1 (tidak ada panggilan tambahan)

MODE B — sortir stok (actual_qty / projected_qty)
  P1.  Bin.get_list   {warehouse}                             -> fondasi (sekali)
  P4a. Item.get_list  {filter wajib + filter user, fields:["name"], limit 0}
                                                              -> kode yang lolos filter (sekali per filter)
  P4b. Bin.get_list   {warehouse + item_code in [kode P4a]}
                      order_by "actual_qty desc, item_code asc"
                      limit_start / limit_page_length         -> 1 halaman (terurut stok)
  P4c. Item.get_list  {name in [item_code halaman P4b]}       -> detail item untuk halaman itu
  P5.  Bin.get_count  {warehouse + item_code in [kode P4a]}   -> total (sekali per filter)

MODE C — katalog kecil: muat semua, sortir & paginasi di frontend
  P1. Bin.get_list   {warehouse, limit 0}                     -> kode + stok semua item gudang
  P6. Item.get_list  {filter wajib + filter user + name in [kode P1], limit 0}
                                                              -> seluruh dataset (2 panggilan total)

FILTER KONDISI STOK (opsional, dijalankan sebelum Mode A/B/C) — §2.13
  P7. Item.get_list  {reorder_levels.warehouse = W}           -> min/max per item (F2/F3/F5/F6)
  P8. SLE.get_list   {warehouse, is_cancelled=0, posting_date = hari ini}
                                                              -> aktivitas hari ini (F4/F5/F6/F8/F9)
  hitung cohort per filter -> iris dengan filter wajib + filter user -> cohort_final
  cohort_final dipakai sebagai ["name","in",...] (Mode A) atau ["item_code","in",...] (Mode B)
```

### 1.2 Mengapa perlu dua sumber paginasi

1. **`Item` tidak punya relasi ke `Warehouse`.** Tidak ada field/child table di `Item` yang berisi
   gudang tempat item terdaftar (`item_defaults.default_warehouse` = *default* per item dan **kosong**
   pada data nyata; `reorder_levels.warehouse` hanya terisi untuk item yang dikonfigurasi reorder).
   Jadi "item terdaftar di gudang X" **hanya** bisa diturunkan dari `tabBin` → pola **P1**.
2. **`tabBin` tidak bisa dijangkau dari `get_list` Item.** Bin bukan child table `Item` dan bukan
   link field → **tidak ada** `JOIN`, sehingga `Item.get_list` tidak bisa mengurutkan/memfilter
   berdasarkan stok (`order_by` hanya menerima `field`, `link_field.field`, atau
   `child_table.field` — §2.9 & §6).
3. **Paginasi terjadi di SQL** (`LIMIT` + `OFFSET`) → **urutan harus sudah ada saat query dijalankan**.
   Karena itu, kolom yang dipakai untuk mengurutkan **harus dimiliki oleh tabel yang memaginasi**:
   - sortir kolom `Item` → `Item` yang memaginasi (**Mode A**), dan daftar `item_code` gudang
     dikirim sebagai filter `["name","in",[...]]`;
   - sortir stok → `Bin` yang memaginasi (**Mode B**), dan daftar `item_code` hasil filter `Item`
     dikirim sebagai filter `["item_code","in",[...]]`.
4. **`get_list` tidak mengembalikan total** → total selalu dari `get_count` (P3 atau P5), yang tidak
   terpengaruh paginasi.

> Konsekuensi yang perlu dipahami UI: pada **Mode B**, langkah **P4a** (daftar `item_code` hasil
> filter) **tidak dipaginasi** dan dijalankan **sekali per perubahan filter** — inilah "harga" dari
> sortir stok + filter item tanpa `JOIN` server.

### 1.3 Yang **bukan** bagian dokumen ini

| Hal | Dokumen |
|---|---|
| Laporan saldo per periode (opening/in/out/balance, nilai & selling price) | [`prd_stock_balance.md`](./prd_stock_balance.md) |
| Kartu stok / mutasi per transaksi, batch/serial | [`prd_stock_ledger.md`](./prd_stock_ledger.md) |
| Mengubah stok (Stock Entry, Material Request, Stock Reconciliation) | [`prd_stock_entry.md`](./prd_stock_entry.md), [`prd_stock_reconciliation.md`](./prd_stock_reconciliation.md) |
| Master item (CREATE/UPDATE, varian, barcode, hashtag, batas min/max stok) | [`prd_item.md`](./prd_item.md) |
| Harga jual / Item Price | [`prd_item_price.md`](./prd_item_price.md) |
| Daftar harga berdasarkan promo (Pricing Rule) | [`prd_item_pricing_rule.md`](./prd_item_pricing_rule.md) |

---

## 2. Parameter & field

Simbol: **🔴 = wajib**, **🟠 = opsional**, **⚪ = output (baca saja)**.

### 2.1 Filter wajib (dipakai di P2, P3, P4a, P4c, P6, P7, dan irisan cohort §2.13)

Enam kondisi berikut **selalu** dikirim di `filters` setiap panggilan yang menyentuh doctype `Item`
(dokumen ini menyebutnya **filter wajib**). Tujuannya: laporan hanya untuk **produk yang
benar-benar dijual/dikelola stoknya** — bukan template varian, bukan paket (bundle), bukan aset.

```json
[
  ["is_stock_item", "=", 1],
  ["is_product_bundle", "=", 0],
  ["is_dynamic_product_bundle", "=", 0],
  ["has_variants", "=", 0],
  ["is_fixed_asset", "=", 0],
  ["disabled", "=", 0]
]
```

| Filter | Arti | Catatan |
|---|---|---|
| `is_stock_item = 1` | item dikelola stok | item non-stok (jasa, paket) tersaring |
| `is_product_bundle = 0` | bukan item induk **Product Bundle** (paket statis) | custom field `baseapp` (lihat [`prd_item_product_bundle.md`](./prd_item_product_bundle.md)) |
| `is_dynamic_product_bundle = 0` | bukan item induk **paket dinamis** | custom field `baseapp` (lihat [`prd_item_dynamic_product_bundle.md`](./prd_item_dynamic_product_bundle.md)) |
| `has_variants = 0` | bukan **template** varian | ⚠️ **varian tetap ikut** (varian punya `has_variants=0` + `variant_of` terisi) → itulah alasan `attribute_value` dibutuhkan |
| `is_fixed_asset = 0` | bukan aset tetap | |
| `disabled = 0` | **item aktif saja** | bila kelak item non-aktif perlu ditampilkan: hapus baris ini, tambahkan `"disabled"` ke `fields` (§2.8) dan ke whitelist sortir (§2.9) |

> Fakta data nyata (site dev, 2026-10-14): filter wajib ini menghasilkan **68 item**; dari seluruh
> `has_variants=0` (87 item) **36 di antaranya varian** dan **13 template terbuang**.

### 2.2 P1 — `Bin.get_list`: fondasi (item terdaftar di gudang + stok)

**Endpoint:** `POST /api/method/frappe.client.get_list`

| Status | Parameter | Nilai | Keterangan |
|---|---|---|---|
| 🔴 Wajib | `doctype` | `"Bin"` | |
| 🔴 Wajib | `filters` | `[["warehouse","=","<warehouse>"]]` | **tepat 1** warehouse, harus **leaf** (§2.11 no. 4) |
| 🔴 Wajib | `fields` | `["item_code","actual_qty","projected_qty"]` | boleh tambah `stock_value`, `valuation_rate`, `ordered_qty`, `reserved_qty` |
| 🔴 Wajib | `limit_page_length` | `0` | `0` = **semua** baris (populasi gudang) |
| 🟠 Opsional | `order_by` | `"item_code asc"` | opsional; default `creation desc` |

Respons: array baris (1 baris = 1 item × 1 gudang). `tabBin` punya **unique index
`(item_code, warehouse)`** (index `unique_item_warehouse`) → **tidak ada duplikat item per gudang**.

Hasil P1 dipakai untuk **3 hal**:
1. daftar `item_code` → filter `["name","in",[...]]` pada **Mode A** (P2/P3), dan → *stock map*;
2. **stok gudang terpilih** yang ditampilkan (frontend memetakan `item_code` → `actual_qty`);
3. **batas populasi**: item yang tidak ada di P1 **tidak boleh** muncul di daftar, walau lolos
   filter item (keputusan 2026-10-22 — item **belum terdaftar** di gudang tidak ditampilkan).

> Konsekuensi: item yang punya baris `Bin` **bersaldo 0** tetap tampil (`actual_qty: 0`), sedangkan
> item yang **belum pernah** punya baris `Bin` di gudang itu tidak tampil sama sekali.

### 2.3 P2 — `Item.get_list`: halaman daftar terurut kolom `Item` (Mode A)

**Endpoint:** `POST /api/method/frappe.client.get_list`

| Status | Parameter | Isi | Keterangan |
|---|---|---|---|
| 🔴 Wajib | `doctype` | `"Item"` | |
| 🔴 Wajib | `fields` | §2.8 | minimal `name`, `item_name`, `has_variants`, `item_group.item_group_name`, `{"attributes": [...]}`, `{"reorder_levels": [...]}` |
| 🔴 Wajib | `filters` | **filter wajib §2.1** + **`["name","in",[kode P1]]`** + filter user | lihat §2.7 untuk filter atribut |
| 🟠 Opsional | `order_by` | §2.9 | **wajib dikirim** untuk paginasi stabil; **selalu** akhiri dengan `name asc` |
| 🟠 Opsional | `limit_start` | Int ≥ 0 (default `0`) | offset: `(page − 1) × limit_page_length` |
| 🟠 Opsional | `limit_page_length` | Int (default `20`) | jumlah baris/halaman; **`0` = semua** (hindari, §5.3 no. 11) |
| 🟠 Opsional | `group_by` | `"name"` | **tidak diperlukan** dengan kontrak filter atribut berpasangan (§2.7); hanya pengaman bila ada duplikat |

**Filter user yang didukung (AND dengan filter wajib):**

| Filter | Bentuk | Keterangan |
|---|---|---|
| **item_name** | `["item_name","like","%<kata>%"]` | pencarian bebas; kolom `item_name` ber-index |
| **attribute** (nama atribut) | `["attributes.attribute","=","Color"]` | **wajib berpasangan** dengan filter nilai |
| **attribute_value** | `["attributes.attribute_value","in",["Blue","Red"]]` | `=` tunggal atau `in` multi — **selalu** bersama `attributes.attribute` (§2.7) |
| **item_group_name** | `["item_group.item_group_name","=","Minuman"]` | bentuk **bertitik** (bukan `item_group_name` polos) & **exact** — §2.11 no. 5 |
| **item_group** (kode) | `["item_group","in",[...leaf...]]` | alternatif bila frontend sudah me-resolve leaf |

### 2.4 P3 — `Item.get_count`: total Mode A

**Endpoint:** `POST /api/method/frappe.client.get_count`

| Status | Parameter | Isi |
|---|---|---|
| 🔴 Wajib | `doctype` | `"Item"` |
| 🔴 Wajib | `filters` | **identik** dengan P2 (filter wajib + `name in [kode P1]` + filter user) |
| 🟠 Opsional | `group_by` | `"name"` — hanya bila UI memakai filter atribut **tanpa** pasangan (angka jadi jumlah item unik) |

Respons: `{"message": <angka>}` = **jumlah item di gudang itu yang lolos filter**.
Diuji: `get_count` = `len(kode P1)` bila tidak ada filter user (dev `Finished Goods - APSD`:
**9 = 9**). Dipakai untuk `total_pages = ceil(total_count / limit_page_length)`; dijalankan **1×**
per kombinasi filter, bukan per halaman.

### 2.5 P4 — sortir stok: kode filter → `Bin` halaman → detail `Item` (Mode B)

**Langkah P4a — resolve kode yang lolos filter (tanpa paginasi, 1× per perubahan filter):**

```json
{ "doctype": "Item", "fields": ["name"],
  "filters": [ /* filter wajib §2.1 */, /* filter user §2.3 */ ],
  "limit_page_length": 0 }
```

Hasil: daftar `name` (= `item_code`). Diuji: filter `item_name like "%a%"` → **59 kode**.

**Langkah P4b — halaman terurut stok:**

| Status | Parameter | Nilai |
|---|---|---|
| 🔴 Wajib | `doctype` | `"Bin"` |
| 🔴 Wajib | `filters` | `[["warehouse","=","<W>"], ["item_code","in",[<kode P4a>]]]` |
| 🔴 Wajib | `fields` | `["item_code","actual_qty","projected_qty"]` (+ opsional kolom Bin lain) |
| 🔴 Wajib | `order_by` | `"actual_qty desc, item_code asc"` (atau `projected_qty`, arah `asc`/`desc`) |
| 🟠 Opsional | `limit_start` / `limit_page_length` | offset & ukuran halaman |

Diuji: `SKU010 (65)`, `HRM-AM-476125 (7)`, `SKU004 (0)` — urutan **global**, paginasi SQL.

**Langkah P4c — detail item untuk halaman:**

```json
{ "doctype": "Item",
  "fields": [ /* sama dengan §2.8 */ ],
  "filters": [["name","in",["<item_code halaman P4b>"]]],
  "limit_page_length": 0 }
```

> Respons P4c urut menurut `Item` (default `creation desc`) — **frontend wajib menyusun ulang**
> mengikuti urutan/ranking dari P4b. Frontend juga cukup mengisi kolom stok dari P4b.

> ⚠️ Bila P4a menghasilkan **list kosong**, **lewati** P4b/P4c/P5 — jangan kirim
> `["item_code","in",[]]` (Frappe mengubah list kosong menjadi `item_code IN ('') OR item_code IS NULL`,
> lihat [`prd_item.md` §4.5.1](./prd_item.md)).

### 2.6 P5 — `Bin.get_count`: total Mode B

```json
{ "doctype": "Bin",
  "filters": [["warehouse","=","<W>"], ["item_code","in",[<kode P4a>]]] }
```

Respons: `{"message": <angka>}` = **jumlah item di gudang itu yang lolos filter** (sama artinya
dengan P3, hanya dihitung dari `Bin`). Diuji: filternya sama dengan P4 → **4**.
Karena `tabBin` unik per item × gudang, angka ini = jumlah item unik (tidak perlu `group_by`).

### 2.7 Kontrak filter atribut (WAJIB berpasangan)

Filter atribut **selalu** dikirim sebagai **pasangan**: nama atribut + nilainya, dalam **satu**
`filters`:

```json
["attributes.attribute", "=", "Color"],
["attributes.attribute_value", "in", ["Blue", "Red"]]
```

**Alasan (terverifikasi):** filter `attribute_value` **tanpa** pasangan `attribute` membangkitkan
`LEFT JOIN` ke `tabItem Variant Attribute` sehingga **1 item bisa muncul berkali-kali** (1 baris per
child row yang cocok). Bukti uji di site dev:

| Filter | Baris dikembalikan | Item unik | `get_count` |
|---|---|---|---|
| `attribute_value in ["Blue","Big"]` (**tanpa** pasangan) | **12** | **9** | **12** ❌ |
| `attribute = "Color"` **+** `attribute_value in ["Green","Blue"]` | **4** | **4** | **4** ✅ |
| `attribute = "Color"`, `attribute_value = "Blue"` | 3 | 3 | 3 ✅ |

Dengan pasangan tersebut duplikat **tidak mungkin** terjadi: pada satu item, satu nama atribut hanya
punya **satu** baris (`Item Variant Attribute` = kunci `(item, attribute)`) → join menghasilkan
maksimal 1 baris per item → **paginasi & total konsisten tanpa `group_by`**.

> ⚠️ Filter atribut juga berlaku di **P4a** (Mode B) — kalau tidak berpasangan, daftar `item_code`
> hasil P4a bisa berisi duplikat sehingga `IN (...)` membengkak dan total P5 salah.

### 2.8 Field per baris daftar item

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| ⚪ Output | `name` | Link → Item | `item_code` (kunci baris) — penghubung ke `Bin` |
| ⚪ Output | `item_name` | Data | **nama tampilan** (unik antar item aktif, [`prd_item.md` §2.1 no. 8](./prd_item.md)) |
| ⚪ Output | `has_variants` | Check | `0` **selalu** (filter wajib) — untuk konsistensi/log |
| ⚪ Output | `item_group` | Link → Item Group | kode group (leaf), mis. `Minuman` |
| ⚪ Output | `item_group_name` | Data | **label** group — dari field bertitik `item_group.item_group_name` (auto `LEFT JOIN` `tabItem Group`) |
| ⚪ Output | `attributes` | Array `{attribute, attribute_value}` | isi child **`Item Variant Attribute`**; varian → terisi; item tunggal → `[]` |
| ⚪ Output | `reorder_levels` | Array `{warehouse, warehouse_reorder_level, max_stock_level}` | isi child **`Item Reorder`** — **semua** gudang item tsb (§2.11 no. 2); `[]` = belum dikonfigurasi |
| ⚪ Output | `variant_of` | Link → Item / `null` | 🟠 **opsional** — untuk memisahkan varian vs item tunggal |
| ⚪ Output | `disabled` | Check | 🟠 **opsional** — hanya bila UI juga ingin menampilkan item non-aktif (§2.1) |
| ⚪ Derivatif (frontend) | `stock` | Float | dari P1 / P4b (`actual_qty` gudang terpilih) — **selalu ada** karena populasi = `Bin` |

### 2.9 Kolom sortir yang didukung (whitelist) & aturan tiebreaker

`frappe.client.get_list` **hanya** menerima tiga bentuk `order_by`:
`field` (kolom doctype itu), `link_field.field` (mis. `item_group.item_group_name`), atau
`child_table.field` (mis. `attributes.attribute`). Nama tabel eksplisit (`tabBin.actual_qty`) dan
kolom milik doctype lain **ditolak** (§6).

| Mode | Tabel paginasi | `order_by` yang boleh | Contoh |
|---|---|---|---|
| **A** | `Item` | `name`, `item_name`, `item_group`, `item_group.item_group_name`, `variant_of`, `stock_uom`, `disabled`, `creation`, `modified` | `"item_name asc, name asc"` · `"item_group.item_group_name asc, name asc"` |
| **B** | `Bin` | `actual_qty`, `projected_qty`, `stock_value`, `valuation_rate`, `ordered_qty`, `reserved_qty`, `item_code` | `"actual_qty desc, item_code asc"` · `"projected_qty asc, item_code asc"` |

**Aturan tiebreaker (wajib):** selalu tambahkan kolom unik sebagai kunci terakhir —
`name asc` untuk Mode A, `item_code asc` untuk Mode B. Tanpa itu, baris dengan nilai sama dapat
berpindah antar halaman saat data berubah (baris ganda/hilang).

> Kolom sortir `stok` **tidak** boleh dikirim sebagai `order_by` pada P2
> (`"actual_qty desc"` → `417 PermissionError`, `"tabBin.actual_qty desc"` → `417 ValidationError`, §6).

### 2.10 Memilih sumber paginasi (ringkasan untuk UI)

| Kolom sortir di UI | Mode | Panggilan | Total dari |
|---|---|---|---|
| `item_name`, `item_group_name`, `name`, `variant_of`, dst. | **A** | P1 → P2 (per halaman) + P3 (sekali) | P3 (`Item.get_count`) |
| `actual_qty`, `projected_qty`, `stock_value` | **B** | P1 → P4a (sekali per filter) → P4b + P4c (per halaman) + P5 (sekali) | P5 (`Bin.get_count`) |
| apa pun (katalog gudang ≤ ±2.000 item) | **C** | P1 + P6 (2 panggilan, sekali) → sortir/paginasi di frontend | hasil gabungan |
| butuh 1 panggilan untuk semua kombinasi | — | custom endpoint `baseapp` (§5.6) | — |

> Saran UX: sediakan pilihan sortir `name`, `item_name`, `item_group_name`, `stok`, dan arah
> `asc`/`desc`. Saat kolom sortir berpindah antara "kolom Item" dan "stok", UI berpindah mode —
> jangan mencampur (mis. mengurutkan stok di P2).

### 2.11 Catatan penting

1. **Stok adalah saldo saat ini, bukan periode.** Sumber `tabBin` (1 baris per item × gudang).
   Untuk laporan periode → `prd_stock_balance.md`.
2. **`reorder_levels` = SEMUA gudang item tersebut** (nested child query difilter per "parent",
   bukan per gudang) → frontend **wajib** memilih baris dengan `warehouse` = gudang terpilih.
   Efek positif: query hanya mengambil baris milik item di halaman.
3. **Populasi = `Bin`.** Item yang belum pernah punya catatan stok di gudang itu **tidak tampil**
   (keputusan 2026-10-22). Item bersaldo 0 yang punya baris `Bin` **tetap tampil**.
4. **`warehouse` harus leaf.** Warehouse **group** (`is_group=1`) tidak punya baris `Bin` →
   P1 kosong → daftar kosong (bukan error). UI harus memakai dropdown leaf
   (`prd_warehouse.md` §5); bila perlu seluruh subtree, resolve leaf-nya lebih dulu.
5. **`item_group_name` bersifat exact.** `["item_group.item_group_name","=","Minuman"]` hanya cocok
   untuk group **leaf bernama tepat itu**; group induk (mis. `All Item Groups`, `Pakaian`) → **0
   baris**. Bila UI memilih group induk, resolve leaf-nya dulu
   ([`prd_item.md` §6.7](./prd_item.md)) atau pakai `["item_group","in",[...leaf...]]`.
6. **`max_stock_level` = custom field `baseapp`** (Float, informatif; tidak memicu logika reorder).
   `warehouse_reorder_level` = ambang **minimum**. Item tanpa baris reorder → tampilkan `-`, **bukan**
   `0` (0 berarti "dikonfigurasi bernilai 0").
7. **`group_by: "name"` bukan bagian kontrak** (§2.7 sudah aman tanpa itu) — hanya alat bantu
   diagnosis bila ada duplikat (§5.3 no. 1).
8. **Jangan memakai `fields` bertitik selain `item_group`.** Uji `variant_of.item_name` menghasilkan
   SQL `SELECT tabItem.item_name` (kolom tabel dasar, **bukan** tabel gabungan) → nilainya salah
   tanpa peringatan.
9. **Halaman kosong → lewati panggilan lanjutan.** Jangan kirim `["item_code","in",[]]` /
   `["name","in",[]]` (§2.5).
10. **`get_count` menghitung baris hasil join**, bukan item unik — konsekuensi langsung dari §5.3 no. 1.

### 2.12 Tabel-tabel terkait

| Tabel (`tab...`) | Doctype | Peran di dokumen ini |
|---|---|---|
| `tabBin` | Bin | **Sumber populasi + stok**: `actual_qty`, `projected_qty` (+ `stock_value`, `valuation_rate`). Unik per item × gudang. |
| `tabItem` | Item | Nama, group, `variant_of`, flag filter wajib; tabel yang memaginasi **Mode A**. Detail: [`prd_item.md`](./prd_item.md). |
| `tabItem Group` | Item Group | Label group (`item_group_name`) via field bertitik + tujuan sortir. Detail: [`prd_item_group.md`](./prd_item_group.md). |
| `tabItem Variant Attribute` | Item Variant Attribute | Isi `attributes[]` + sumber filter atribut berpasangan. |
| `tabItem Reorder` | Item Reorder | Child `Item.reorder_levels`: `warehouse`, `warehouse_reorder_level`, `warehouse_reorder_qty`, `max_stock_level`. |
| `tabWarehouse` | Warehouse | Penentuan gudang **leaf**. Detail: [`prd_warehouse.md`](../setup/prd_warehouse.md). |

### 2.13 Filter kondisi stok (9 filter tambahan)

Sembilan filter berikut **menentukan cohort `item_code` lebih dulu** (Tahap 1), lalu cohort diiris
dengan filter wajib §2.1 + filter user, baru ditampilkan/dipaginasi menurut §2.10 (Tahap 2).
**Semua filter di bawah memerlukan `warehouse`** — tanpa gudang, filter kondisi stok tidak bermakna.
Bila beberapa filter dipilih sekaligus → **irisan** (AND) cohort masing-masing.

| Kode | Filter | Definisi (kontrak) | Dihitung di | Sumber |
|---|---|---|---|---|
| **F1** | Stok kosong | `Bin.actual_qty = 0` | **server** (`Bin`) | **P1** |
| **F7** | Stok minus | `Bin.actual_qty < 0` | **server** (`Bin`) | **P1** |
| **F2** | Menyentuh minimum (kondisi sekarang) | `Bin.actual_qty <= warehouse_reorder_level` gudang terpilih | **frontend** | **P1** + **R** |
| **F3** | Menyentuh maksimum (kondisi sekarang) | `Bin.actual_qty >= max_stock_level` gudang terpilih | **frontend** | **P1** + **R** |
| **F5** | Menyentuh minimum **hari ini** | **masuk zona hari ini + berakhir di zona**: posisi **awal hari** `> min` **dan** posisi **akhir hari** `Bin.actual_qty <= min` | **frontend** (rumus) | **P1** + **T** + **R** |
| **F6** | Menyentuh maksimum **hari ini** | idem: posisi **awal hari** `< max` **dan** posisi **akhir hari** `Bin.actual_qty >= max` | **frontend** (rumus) | **P1** + **T** + **R** |
| **F4** | Baru habis hari ini | `Bin.actual_qty = 0` **dan** saldo akhir **sebelum hari ini** `> 0` | **frontend** (rumus `opening`) | **P1** + **T** |
| **F8** | Stoknya menjadi minus hari ini | `Bin.actual_qty < 0` **dan** saldo akhir sebelum hari ini `>= 0` | **frontend** (rumus `opening`) | **P1** + **T** |
| **F9** | Stoknya berubah | ada ≥ 1 baris SLE hari ini (`is_cancelled=0`) | **server** (`SLE`) | **T** |

**Aturan partisipasi item (keputusan 2026-10-23):**

- Item **tanpa baris `Item Reorder`** untuk gudang terpilih → **dikecualikan** dari F2/F3/F5/F6.
- Nilai `warehouse_reorder_level = 0` atau `max_stock_level = 0` **dianggap "tidak dikonfigurasi"**
  → item tersebut juga **dikecualikan** dari filter yang bersangkutan (mencegah kasus "semua item
  menyentuh maksimum" ketika `max = 0`).
- F1/F7/F4/F8/F9 tidak memerlukan konfigurasi reorder.

**Sumber data tambahan** (selain **P1** yang sudah ada di §2.2):

- **R — level reorder per gudang** (`Item` + nested child):
  ```json
  { "doctype": "Item",
    "fields": ["name", {"reorder_levels": ["warehouse","warehouse_reorder_level","max_stock_level"]}],
    "filters": [["reorder_levels.warehouse","=","<W>"]],
    "limit_page_length": 0 }
  ```
  Respons memuat **semua** gudang per item (§2.11 no. 2) → frontend memilih baris `warehouse = W`.
- **T — aktivitas hari ini** (`Stock Ledger Entry`, versi **agregat** — cukup untuk F4/F5/F6/F8/F9):
  ```json
  { "doctype": "Stock Ledger Entry",
    "fields": ["item_code", {"SUM": "actual_qty", "as": "qty_today"}],
    "filters": [["warehouse","=","<W>"], ["is_cancelled","=",0],
                ["posting_date","=","<hari ini YYYY-MM-DD>"]],
    "group_by": "item_code", "limit_page_length": 0 }
  ```
  Satu baris per item yang **punya aktivitas hari ini** (termasuk yang net-nya `0`), dengan
  `qty_today` = Σ `actual_qty` hari ini. Item yang **tidak** muncul = tidak ada mutasi hari ini.
  Bila UI juga perlu **waktu kejadian** (mis. menampilkan jam tersentuhnya), ambil versi rinci dengan
  `fields: ["item_code","posting_datetime","qty_after_transaction","actual_qty"]` +
  `order_by: "item_code asc, posting_datetime asc"`.

**Rumus saldo sebelum hari ini (terverifikasi):**

```text
opening = Bin.actual_qty − Σ(SLE.actual_qty hari ini, is_cancelled = 0)
```

> Bukti identitas: untuk **semua** baris `Bin` di gudang uji,
> `Bin.actual_qty − Σ(SLE.actual_qty, is_cancelled=0)` = **0**. Jadi tidak perlu menarik riwayat
> SLE sebelum hari ini untuk menghitung `opening`.

**Rumus F5/F6 (masuk zona hari ini dan **berakhir** di zona):**

```text
F5: opening > min  AND  Bin.actual_qty <= min
F6: opening < max  AND  Bin.actual_qty >= max
```

- `opening` = posisi **awal hari**; `Bin.actual_qty` = posisi **akhir hari** (sekarang).
- Item yang hari ini sempat masuk zona lalu **naik lagi keluar zona** → **tidak** memenuhi F5/F6
  (mis. `10 → 3 → 8` dengan min `5`).
- Item yang **sudah** berada di zona sejak sebelum hari ini → tidak memenuhi F5/F6 (memenuhi F2/F3),
  walau tetap di zona sepanjang hari.
- `Bin.actual_qty` dipakai sebagai "akhir hari" — identik dengan baris SLE terakhir hari ini selama
  tidak ada dokumen bertanggal setelah hari ini.

**Alur Tahap 1 → Tahap 2:**

1. **P1** (selalu) + **R** bila F2/F3/F5/F6 dipakai + **T** bila F4/F5/F6/F8/F9 dipakai.
2. Hitung cohort tiap filter aktif, lalu **irisan**-kan (AND).
3. **Iris dengan filter wajib + filter user** — satu panggilan:
   `Item.get_list` `fields:["name"]` + filter wajib §2.1 + filter user §2.3 +
   `["name","in",[cohort]]`, `limit_page_length: 0` → **`cohort_final`** (sekaligus membuang item
   non-stok/`disabled`/template varian).
4. Tampilkan & paginasi memakai `cohort_final` sebagai `["name","in",...]` (**Mode A**) atau
   `["item_code","in",...]` (**Mode B**). **`total_count = len(cohort_final)`** → `get_count` tidak
   diperlukan lagi.

**Catatan penting filter kondisi stok:**

1. **Jangan** mengirim nama kolom sebagai nilai filter. Uji `["actual_qty","<=","warehouse_reorder_level"]`
   menghasilkan SQL `actual_qty <= %(param)s` dengan nilai **string** `"warehouse_reorder_level"`;
   pada filter child menjadi `IFNULL(level,'') < 'actual_qty'` (perbandingan **string**).
   **Tidak ada error**, tetapi hasilnya salah — perbandingan dua kolom wajib di frontend.
2. **SLE yang dibatalkan wajib disaring**: `["is_cancelled","=",0]`
   ([`prd_stock_ledger.md` §2.1](./prd_stock_ledger.md)).
3. **"Hari ini" = tanggal server** (`frappe.utils.today()`); `posting_time` tidak dipakai.
4. **Role:** membaca `Stock Ledger Entry` memerlukan `Stock User` atau `Accounts Manager`
   ([`prd_stock_ledger.md` §2.1 no. 5](./prd_stock_ledger.md)).
5. **Volume:** `T` bisa ribuan baris/hari/gudang → bila berat, batasi filter atau pindahkan ke
   custom API (§5.6).
6. **F2/F3 = kondisi sekarang**, **F5/F6 = masuk zona hari ini dan berakhir di zona**. Contoh: min `5`,
   awal hari `10`, hari ini terjual → `3` → **F5 ya**; tetapi bila setelah itu di-restock menjadi `8`
   → **F5 tidak** (berakhir di luar zona). Stok yang sudah `3` sejak kemarin lalu hari ini `2` →
   **F5 tidak** (bukan transisi hari ini), tetapi **F2 tetap ya**.

> Dokumen ini **tidak** menulis ke tabel mana pun dan **tidak** menambahkan method/field baru.

---

## 3. Autentikasi

Seluruh request wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Alternatif (integrasi server-to-server): `Authorization: token <api_key>:<api_secret>`. Cara yang
dianjurkan untuk aplikasi klien tetap **OAuth 2.0** — setup lengkap di
**[`prd_oauth.md`](../prd_oauth.md)**.

> **Permission otomatis (berbeda dari custom API):** karena memakai `frappe.client.*`, Frappe
> memeriksa **DocPerm** dan **User Permission** sendiri. Konsekuensi:
> - user tanpa hak baca `Item`/`Bin` → `403`;
> - user dengan User Permission (`allow = Warehouse`) **hanya melihat gudang yang diizinkan** →
>   P1 atas gudang di luar izinnya kosong → daftar kosong (bukan error).
> - Bandingkan custom API di [`prd_stock_balance.md` §5.4](./prd_stock_balance.md) yang harus
>   memeriksa permission **manual**.

---

## 4. Contoh pemakaian

> **Data contoh** (fiktif, konsisten di §4): company **PT Maju Jaya**, gudang
> **`Toko Cikarang - PTMJ`**; `Bin` gudang tsb berisi 3 item:
> `Air Mineral 600ml` = 110 (group `Minuman`), `Kemeja Flanel-M` = 12, `Kemeja Flanel-L` = 0
> (keduanya varian dari template `Kemeja Flanel`, group `Pakaian`, atribut `Color`/`Size`).
> Item `Kaos Kaki` ada di master tetapi **tidak punya baris `Bin`** di gudang ini → **tidak tampil**.

### 4.1 P1 — fondasi: item terdaftar di gudang + stok

```bash
curl -X POST "https://site-anda.com/api/method/frappe.client.get_list" \
  -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" \
  -d '{
        "doctype": "Bin",
        "fields": ["item_code", "actual_qty", "projected_qty"],
        "filters": [["warehouse", "=", "Toko Cikarang - PTMJ"]],
        "limit_page_length": 0
      }'
```

**Respons sukses (HTTP 200):**

```json
{
  "message": [
    { "item_code": "Air Mineral 600ml", "actual_qty": 110.0, "projected_qty": 110.0 },
    { "item_code": "Kemeja Flanel-M",   "actual_qty": 12.0,  "projected_qty": 12.0  },
    { "item_code": "Kemeja Flanel-L",   "actual_qty": 0.0,   "projected_qty": 0.0   }
  ]
}
```

> Simpan 3 hal: daftar `item_code`, *stock map* (`item_code` → `actual_qty`), dan jumlahnya
> (= populasi gudang). `Kaos Kaki` tidak ada di sini → tidak boleh muncul di daftar.

### 4.2 Mode A — sortir `item_name`

```bash
curl -X POST "https://site-anda.com/api/method/frappe.client.get_list" \
  -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" \
  -d '{
        "doctype": "Item",
        "fields": [
          "name", "item_name", "has_variants", "item_group",
          "item_group.item_group_name",
          { "attributes": ["attribute", "attribute_value"] },
          { "reorder_levels": ["warehouse", "warehouse_reorder_level", "max_stock_level"] }
        ],
        "filters": [
          ["is_stock_item", "=", 1],
          ["is_product_bundle", "=", 0],
          ["is_dynamic_product_bundle", "=", 0],
          ["has_variants", "=", 0],
          ["is_fixed_asset", "=", 0],
          ["disabled", "=", 0],
          ["name", "in", ["Air Mineral 600ml", "Kemeja Flanel-M", "Kemeja Flanel-L"]]
        ],
        "order_by": "item_name asc, name asc",
        "limit_start": 0,
        "limit_page_length": 50
      }'
```

**Respons sukses (HTTP 200)** — perhatikan urutan `item_name`:

```json
{
  "message": [
    {
      "name": "Air Mineral 600ml",
      "item_name": "Air Mineral 600ml",
      "has_variants": 0,
      "item_group": "Minuman",
      "item_group_name": "Minuman",
      "attributes": [],
      "reorder_levels": []
    },
    {
      "name": "Kemeja Flanel-L",
      "item_name": "Kemeja Flanel - L",
      "has_variants": 0,
      "item_group": "Pakaian",
      "item_group_name": "Pakaian",
      "attributes": [
        { "attribute": "Color", "attribute_value": "Merah" },
        { "attribute": "Size", "attribute_value": "L" }
      ],
      "reorder_levels": [
        { "warehouse": "Toko Cikarang - PTMJ", "warehouse_reorder_level": 5.0, "max_stock_level": 50.0 },
        { "warehouse": "Gudang Pusat - PTMJ",  "warehouse_reorder_level": 20.0, "max_stock_level": 200.0 }
      ]
    },
    {
      "name": "Kemeja Flanel-M",
      "item_name": "Kemeja Flanel - M",
      "has_variants": 0,
      "item_group": "Pakaian",
      "item_group_name": "Pakaian",
      "attributes": [
        { "attribute": "Color", "attribute_value": "Merah" },
        { "attribute": "Size", "attribute_value": "M" }
      ],
      "reorder_levels": [
        { "warehouse": "Toko Cikarang - PTMJ", "warehouse_reorder_level": 5.0, "max_stock_level": 50.0 },
        { "warehouse": "Gudang Pusat - PTMJ",  "warehouse_reorder_level": 20.0, "max_stock_level": 200.0 }
      ]
    }
  ]
}
```

> - Kolom stok **tidak** ada di respons ini → tempelkan dari P1 (*stock map*) memakai `name`.
> - `Kemeja Flanel-*` membawa reorder level **2 gudang**; UI memilih baris dengan
>   `warehouse == "Toko Cikarang - PTMJ"` → `warehouse_reorder_level: 5`, `max_stock_level: 50`.
> - **Selalu** kirim `order_by` + tiebreaker `name asc` (default server `creation desc`).

### 4.3 Mode A — sortir `item_group_name`

```bash
# ganti order_by saja
"order_by": "item_group.item_group_name asc, name asc"
```

> Diuji di site dev: `order_by "item_group.item_group_name asc, name asc"` **diterima** dan
> mengurutkan berdasarkan label group. (Pada site dev, group item sudah dikolaps oleh `baseapp`
> menjadi `Non Category`, jadi seluruh baris bisa terlihat "sama" — itu wajar.)

### 4.4 Mode B — sortir stok `desc` (dengan filter pencarian)

```bash
# P4a — resolve kode yang lolos filter (sekali per perubahan filter, tanpa paginasi)
curl -X POST "https://site-anda.com/api/method/frappe.client.get_list" \
  -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" \
  -d '{
        "doctype": "Item",
        "fields": ["name"],
        "filters": [
          ["is_stock_item","=",1], ["is_product_bundle","=",0],
          ["is_dynamic_product_bundle","=",0], ["has_variants","=",0],
          ["is_fixed_asset","=",0], ["disabled","=",0],
          ["item_name","like","%kemeja%"]
        ],
        "limit_page_length": 0
      }'

# P4b — halaman terurut stok untuk gudang tsb
curl -X POST "https://site-anda.com/api/method/frappe.client.get_list" \
  -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" \
  -d '{
        "doctype": "Bin",
        "fields": ["item_code", "actual_qty", "projected_qty"],
        "filters": [["warehouse", "=", "Toko Cikarang - PTMJ"],
                    ["item_code", "in", ["Kemeja Flanel-M", "Kemeja Flanel-L"]]],
        "order_by": "actual_qty desc, item_code asc",
        "limit_start": 0, "limit_page_length": 50
      }'

# P4c — detail item untuk halaman P4b
curl -X POST "https://site-anda.com/api/method/frappe.client.get_list" \
  -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" \
  -d '{
        "doctype": "Item",
        "fields": ["name", "item_name", "item_group.item_group_name",
                   { "attributes": ["attribute", "attribute_value"] }],
        "filters": [["name", "in", ["Kemeja Flanel-M", "Kemeja Flanel-L"]]],
        "limit_page_length": 0
      }'
```

**Respons P4b (HTTP 200)** — sudah terurut stok:

```json
{
  "message": [
    { "item_code": "Kemeja Flanel-M", "actual_qty": 12.0, "projected_qty": 12.0 },
    { "item_code": "Kemeja Flanel-L", "actual_qty": 0.0,  "projected_qty": 0.0  }
  ]
}
```

> - `Kemeja Flanel-L` tetap tampil karena **punya baris `Bin`** (saldo 0).
> - P4c bisa mengembalikan urutan berbeda (`creation desc`) → UI menyusun ulang mengikuti urutan
>   P4b. Alternatif tanpa panggilan P4c: kirim `fields` lengkap di **P4a** (sekali), lalu petakan
>   detail dari sana.
> - **P5** untuk total: `get_count("Bin", {"warehouse": …, "item_code": ["Kemeja Flanel-M","Kemeja Flanel-L"]})` → `2`.

### 4.5 Mode B — "hanya yang ada stok" + sortir `asc`

```json
{ "doctype": "Bin",
  "fields": ["item_code", "actual_qty"],
  "filters": [["warehouse", "=", "Toko Cikarang - PTMJ"],
              ["item_code", "in", ["Air Mineral 600ml", "Kemeja Flanel-M", "Kemeja Flanel-L"]],
              ["actual_qty", ">", 0]],
  "order_by": "actual_qty asc, item_code asc",
  "limit_page_length": 0 }
```

> `actual_qty > 0` adalah filter **`Bin`** (boleh), diuji di site dev: menyisakan hanya baris
> bersaldo positif. Untuk "hanya yang minus": `["actual_qty","<",0]`.

### 4.6 Filter pencarian / atribut / item group

```json
"filters": [
  ["is_stock_item","=",1], ["is_product_bundle","=",0],
  ["is_dynamic_product_bundle","=",0], ["has_variants","=",0],
  ["is_fixed_asset","=",0], ["disabled","=",0],
  ["item_name","like","%kemeja%"],
  ["attributes.attribute","=","Color"],
  ["attributes.attribute_value","in",["Merah"]],
  ["item_group.item_group_name","=","Pakaian"]
]
```

> Semua **AND**. Filter yang sama dipakai di **P2/P3** (Mode A) dan di **P4a** (Mode B) agar
> populasi & total konsisten. Jangan menaruh filter ini di `Bin` (§6).

### 4.7 Paginasi

| Halaman | `limit_start` | `limit_page_length` |
|---|---|---|
| 1 | `0` | 50 |
| 2 | `50` | 50 |
| `n` | `(n − 1) × 50` | 50 |

- Mode A: paginasi di **P2** (`Item`), tiebreaker `name asc`.
- Mode B: paginasi di **P4b** (`Bin`), tiebreaker `item_code asc`; P4c selalu mengambil
  **seluruh** `item_code` halaman itu (tanpa offset).
- `limit_page_length` tidak dibatasi server (boleh `0` = semua), tetapi UI disarankan ≤ 500/halaman.
- Halaman melebihi total → `{"message": []}` (bukan error).

### 4.8 Mode C — muat semua & sortir di frontend (katalog kecil)

```bash
# 1) fondasi (P1)
curl -X POST "https://site-anda.com/api/method/frappe.client.get_list" \
  -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" \
  -d '{ "doctype": "Bin", "fields": ["item_code","actual_qty","projected_qty"],
        "filters": [["warehouse","=","Toko Cikarang - PTMJ"]], "limit_page_length": 0 }'

# 2) seluruh detail item di gudang itu (P6)
curl -X POST "https://site-anda.com/api/method/frappe.client.get_list" \
  -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" \
  -d '{ "doctype": "Item",
        "fields": ["name","item_name","item_group.item_group_name",
                   {"attributes":["attribute","attribute_value"]},
                   {"reorder_levels":["warehouse","warehouse_reorder_level","max_stock_level"]}],
        "filters": [["is_stock_item","=",1],["is_product_bundle","=",0],
                    ["is_dynamic_product_bundle","=",0],["has_variants","=",0],
                    ["is_fixed_asset","=",0],["disabled","=",0],
                    ["name","in",["<item_code dari P1>"]],
                    ["item_name","like","%<kata>%"]],
        "limit_page_length": 0 }'
```

Lalu di frontend: gabungkan per `item_code`, isi `stock` dari P1, **sortir kolom apa pun**
(termasuk stok), dan paginasi lokal. `total_count = jumlah baris hasil gabungan`.

> Ukuran nyata di site dev (`Finished Goods - APSD`, 9 item): P1 = **627 byte**, P6 = **1.204 byte**.
> Aman sampai beberapa ribu item; untuk katalog puluhan ribu gunakan Mode A/B atau custom API.

### 4.9 Verifikasi angka (QA)

| Uji | Cara |
|---|---|
| Total Mode A | P3 (`Item.get_count`) **harus** sama dengan jumlah baris P2 bila `limit_page_length: 0` dan `filters` identik (dev: **9 = 9**) |
| Total Mode B | P5 (`Bin.get_count`) **harus** sama dengan jumlah baris P4b dengan `limit_page_length: 0` |
| Konsistensi lintas mode | P3 = P5 untuk filter yang sama (dev, `item_name like "%a%"`: keduanya **4**) |
| Urutan | Mode A: cek `item_name`/`item_group_name` naik/turun; Mode B: cek `actual_qty` menurun + `item_code` sebagai tiebreaker |
| Populasi | item yang ada di master tetapi **tidak ada di P1** harus **tidak** muncul di daftar |
| Stok | bandingkan dengan tab/`Bin` di Desk atau report Stock Balance (`prd_stock_balance.md`) |
| Reorder level | bandingkan dengan tab **Reorder Levels** form Item (per gudang) |
| Atribut | bandingkan dengan tab **Attributes** form Item varian |
| Duplikat | `len(rows) == len(set(item_code))` di setiap halaman (§5.3 no. 1) |

### 4.10 Contoh filter kondisi stok yang dihitung di server (F1, F7, F9)

**F1 — stok kosong** dan **F7 — stok minus** hanya menambah filter pada **P1**:

```bash
# F1: stok kosong
curl -X POST "https://site-anda.com/api/method/frappe.client.get_list" \
  -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" \
  -d '{ "doctype": "Bin", "fields": ["item_code","actual_qty"],
        "filters": [["warehouse","=","Toko Cikarang - PTMJ"], ["actual_qty","=",0]],
        "limit_page_length": 0 }'

# F7: stok minus
curl -X POST "https://site-anda.com/api/method/frappe.client.get_list" \
  -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" \
  -d '{ "doctype": "Bin", "fields": ["item_code","actual_qty"],
        "filters": [["warehouse","=","Toko Cikarang - PTMJ"], ["actual_qty","<",0]],
        "limit_page_length": 0 }'
```

Hasil uji di gudang `Finished Goods - APSD`: F1 → **1 item** (`SKU004`), F7 → **5 item**
(`SKU001`, `SKU002`, `SKU003`, `SKU006`, `SKU007`).

**F9 — stoknya berubah hari ini** (1 panggilan, tanpa perlu detail per baris):

```bash
curl -X POST "https://site-anda.com/api/method/frappe.client.get_list" \
  -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" \
  -d '{ "doctype": "Stock Ledger Entry",
        "fields": ["item_code", {"SUM": "actual_qty", "as": "qty_today"}],
        "filters": [["warehouse","=","Finished Goods - APSD"],
                    ["is_cancelled","=",0], ["posting_date","=","2020-12-01"]],
        "group_by": "item_code", "limit_page_length": 0 }'
```

> **Respons yang diharapkan:** 1 baris per item yang bergerak hari itu, mis.
> `AABB` (`qty_today: 2.0`) dan `SKU010` (`qty_today: -13.0`) pada tanggal simulasi `2020-12-01`.
> Seluruh riwayat gudang itu menyentuh **8 item** (uji `group_by` = sama dengan `count(distinct item_code)`).

Setelah cohort didapat, lakukan **Tahap 2** persis seperti §2.10 — cohort dikirim sebagai
`["name","in",[...]]` (Mode A) atau `["item_code","in",[...]]` (Mode B).

### 4.11 Contoh filter yang butuh perbandingan/rumus di frontend (F2–F6, F8)

**Langkah 1 — ambil P1 + R:**

```bash
# R: level min/max untuk gudang terpilih
curl -X POST "https://site-anda.com/api/method/frappe.client.get_list" \
  -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" \
  -d '{ "doctype": "Item",
        "fields": ["name", {"reorder_levels": ["warehouse","warehouse_reorder_level","max_stock_level"]}],
        "filters": [["reorder_levels.warehouse","=","Finished Goods - APSD"]],
        "limit_page_length": 0 }'
```

Hasil uji: **11 item** di gudang itu punya baris reorder, mis. `HRM-AM-284503` → min `5`, max `50`;
`PAK-S` → min `0`, max `0` (**dianggap tidak dikonfigurasi → dikecualikan**).

**Langkah 2 — hitung cohort (frontend).** Contoh nyata dengan `P1` (Bin) + `R`:

| Item | `Bin.actual_qty` | min | max | F2 (`qty <= min`) | F3 (`qty >= max`) |
|---|---|---|---|---|---|
| `HRM-AM-284503` | 7 | 5 | 50 | ❌ (7 > 5) | ❌ (7 < 50) |
| `PAK-S` | – | 0 | 0 | dikecualikan | dikecualikan |

**F4 / F8 — pakai rumus `opening`** (P1 + T):

```text
opening = Bin.actual_qty − Σ(SLE.actual_qty hari ini, is_cancelled = 0)
F4: Bin.actual_qty = 0  AND opening > 0
F8: Bin.actual_qty < 0  AND opening >= 0
```

Contoh nyata pada tanggal simulasi `2020-12-01`:

| Item | `Bin.actual_qty` | Σ `actual_qty` hari ini | `opening` | F4 | F8 |
|---|---|---|---|---|---|
| `SKU004` | 0 | 0 | 0 | ❌ (sudah 0 sebelum hari ini) | ❌ |
| `SKU001` | −25 | 0 | −25 | ❌ | ❌ (sudah minus sebelumnya) |
| `SKU010` | 65 | −13 | 78 | ❌ | ❌ |

> Pada data dev tidak ada item yang memenuhi F4/F8 pada tanggal simulasi (riwayatnya tidak memuat
> transisi tersebut) — **rumus & panggilannya** yang diverifikasi, bukan angkanya.

**F5 / F6 — “menyentuh hari ini” = masuk zona hari ini dan berakhir di zona** (P1 + T + R):

```text
opening = Bin.actual_qty − qty_today          (qty_today dari T)
F5: opening > min  AND  Bin.actual_qty <= min
F6: opening < max  AND  Bin.actual_qty >= max
```

| Skenario | min / max | `opening` (awal hari) | `Bin.actual_qty` (akhir hari) | Hasil |
|---|---|---|---|---|
| min `5`; minggu lalu `10`; **hari ini** terjual → `3` | min 5 | 10 | 3 | **F5 ✅** |
| min `5`; hari ini `10 → 3` lalu di-restock → `8` | min 5 | 10 | 8 | **F5 ❌** (berakhir di luar zona) |
| min `5`; hari ini `10 → 3` lalu ditambah → `4` | min 5 | 10 | 4 | **F5 ✅** (masuk & berakhir di zona) |
| max `20`; sebelumnya `10`; **dua hari lalu** beli → `30`; hari ini tetap `30` | max 20 | 30 | 30 | **F6 ❌** (awal hari sudah ≥ max) |
| max `20`; `opening` 10; **hari ini** beli → `30` | max 20 | 10 | 30 | **F6 ✅** |
| min `5`; sudah `3` sejak kemarin; hari ini turun → `2` | min 5 | 3 | 2 | **F5 ❌** — tetapi **F2 ✅** |

> `opening` = saldo **awal hari** (bukan "kemarin"), dihitung dari identitas `Bin = Σ SLE`.
> "Akhir hari" memakai `Bin.actual_qty`; nilainya sama dengan `qty_after_transaction` baris SLE
> terakhir hari ini selama tidak ada dokumen bertanggal setelah hari ini. Bila UI perlu **waktu
> kejadian**, ambil versi rinci **T** (§2.13).

---

## 5. Catatan teknis (tanpa kode baru) & batasan

### 5.1 Apa yang dijalankan server

| Fitur yang dipakai | Perilaku (terverifikasi pada frappe v16 / ERPNext 16.32.3) |
|---|---|
| `fields` bertitik `item_group.item_group_name` | `validate_fields` melewatkannya → query builder menambah `LEFT JOIN \`tabItem Group\`` dan menamai kolom `item_group_name` |
| `fields` bertipe dict (`{"attributes":[...]}`) | child query terpisah (`parent IN (...) ORDER BY idx`) → array bersarang, baris induk tidak terduplikasi |
| `order_by` `link_field.field` | diterima: `SELECT ... ORDER BY \`tabItem Group\`.\`item_group_name\`` |
| `order_by` `child_table.field` / tabel lain | `tabBin.actual_qty` → `ValidationError: Invalid field format in Order By ...` |
| `filters` ke child (`attributes.attribute_value`) | `LEFT JOIN` child → **dapat** menduplikasi baris induk (§2.7) |
| `filters` dengan nilai berupa **nama kolom** | dikirim sebagai **parameter string**, bukan referensi kolom: `actual_qty <= %(param)s` / `IFNULL(child_level,'') < %(param)s` → tidak ada perbandingan kolom (§2.13 note 1) |
| 2 kondisi pada child yang sama | satu join yang sama (`WHERE attribute=? AND attribute_value=?`) |
| `group_by: "name"` + `order_by: "name asc"` | 1 baris per item, paginasi stabil |
| `get_count` | `SELECT COUNT(*) FROM (SELECT name FROM tabItem WHERE <filters>)` — menghitung baris join |
| `fields` dict `{"SUM":"actual_qty","as":"x"}` | menjadi `SUM(\`actual_qty\`) AS x` |
| `limit_page_length: 0` | tanpa klausa `LIMIT` → seluruh baris |

### 5.2 Hasil verifikasi di site dev (2026-10-14, 2026-10-22 & 2026-10-23)

| Uji | Hasil |
|---|---|
| Filter wajib §2.1 (`Item.get_count`) | **68** item |
| `has_variants=0` (tanpa filter lain) | **87** item → **36** varian + **13** template terbuang |
| Filter `attribute_value in ["Blue","Big"]` (tanpa pasangan) | **12** baris, **9** item unik, `get_count` **12** → duplikat |
| `attribute = "Color"` + `attribute_value in ["Green","Blue"]` | **4** baris = **4** item unik = `get_count` **4** |
| `group_by: "name"` (daftar) + `group_by` di body `get_count` | 12 → **9** (pengaman, §5.3 no. 1) |
| Paginasi `group_by` + `order_by name asc` (3 halaman × 4) | tanpa duplikat/overlap |
| P1 gudang `Finished Goods - APSD` | **9 baris**, **627 byte** |
| P3 dengan `name in [9 kode]` | **9** (= len P1) |
| Sortir `item_name asc` pada 9 item itu | `Book`, `Camera`, `Coffee Mug` |
| Sortir `item_group.item_group_name asc` | diterima & berjalan |
| Filter `item_name like "%a%"` → P4a | **59 kode** |
| P4b (gudang tsb ∩ 59 kode) | **4 item**, halaman: `SKU010 65`, `HRM-AM-476125 7`, `SKU004 0` |
| P4b `order_by "actual_qty desc, item_code asc"` | urut global, hal. 1 `65/7/2`, hal. 2 `0/−15/−25` |
| Filter `["actual_qty",">",0]` di `Bin` | diterima → hanya saldo positif |
| `Bin` + filter Item (`is_stock_item`) | **ditolak** → `DataError: Field not permitted in query: is_stock_item` |
| `Item` + `order_by "actual_qty desc"` | **ditolak** → `PermissionError: ... Item.actual_qty` |
| `order_by "tabBin.actual_qty desc"` | **ditolak** → `ValidationError: Invalid field format in Order By` |
| `Bin` `fields`/`order_by` `item_name` | ditolak (`DataError` / `PermissionError`) |
| Index `tabBin` | `unique_item_warehouse` **unique** (item_code, warehouse) |
| Mode C (9 item) | P1 **627 B** + P6 **1.204 B**; top3 stok `65 / 7 / 2`; total 9 |
| Field tidak dikenal di `fields`/`filters` | `417 DataError: Field not permitted in query: <field>` |
| Nested dict ke **link field** (`{"item_group":["item_group_name"]}`) | `AttributeError: 'ChildQuery' object has no attribute 'fieldname'` (500) |
| Agregat bentuk string (`"sum(actual_qty) as x"`) | `417 ValidationError ... Use dict syntax like {'COUNT': '*'}` |
| Group induk (`item_group_name = "All Item Groups"`) | **0** baris (perlu resolve leaf) |
| **Identitas `Bin` vs SLE** (2026-10-23) | untuk **semua** baris `Bin` gudang uji: `Bin.actual_qty − Σ(SLE.actual_qty, is_cancelled=0)` = **0** |
| **F1 stok kosong** (`Bin.actual_qty = 0`) | **1** item (`SKU004`) di gudang uji |
| **F7 stok minus** (`< 0`) | **5** item (`SKU001/002/003/006/007`) |
| **F9 stok berubah** (`SLE` `group_by item_code`) | seluruh riwayat = **8 item**; per hari (`2020-12-01`) = **2 item** (`AABB +2`, `SKU010 −13`) |
| **R level reorder per gudang** | **11 item** punya baris `Item Reorder` untuk `Finished Goods - APSD` (mis. min 5/max 50; ada pula baris min 0/max 0) |
| **Filter nilai = nama kolom** (`["actual_qty","<=","warehouse_reorder_level"]`) | **tidak error** → SQL `actual_qty <= %(p)s` dengan `p` string; pada child jadi `IFNULL(level,'') < 'actual_qty'` → **hasil salah** |
| **Simulasi F2/F3/F4/F5/F6/F8** (2020-12-01) | semua **0** item — data dev tidak memuat kasusnya; rumus & panggilan yang diverifikasi (`SKU004` opening 0, `SKU001` opening −25, `SKU010` opening 78). Contoh transisi F5/F6 di §4.11 dihitung manual dari data contoh. |

### 5.3 Batasan & jebakan (baca sebelum implementasi UI)

1. **Duplikat baris & total membengkak bila filter atribut tidak berpasangan.** Kontrak §2.7 wajib.
   Pengaman opsional: `"group_by": "name"` di `get_list` **dan** `get_count`.
   ⚠️ `group_by` pada `get_count` **tidak ada di signature resminya**; ia bekerja karena parameter
   body bocor ke `frappe.form_dict` (pola sama dengan `or_filters`, lihat
   [`prd_item.md` §2.1 no. 9](./prd_item.md)) — jangan diandalkan tanpa uji regresi setelah upgrade.
2. **Tidak ada `JOIN`** → tidak mungkin satu query memfilter/sortir lintas `Item` + `Bin`. Karena itu
   ada dua sumber paginasi (§2.10). **Jangan** mengirim `order_by` stok ke P2, atau filter item ke P4b.
3. **Mode B butuh P4a tanpa paginasi.** Biayanya sebanding jumlah item yang lolos filter (bukan
   seluruh katalog, tetapi bisa ribuan) → `IN (...)` besar; gunakan Mode C/custom API bila berat.
4. **`reorder_levels` tidak terfilter per gudang** → filter di frontend. `get_list` ke doctype
   `Item Reorder` **bukan** solusi (child doctype tanpa DocPerm → hanya `name`).
5. **Dotted field hanya aman untuk `item_group`** (§2.11 no. 8).
6. **`warehouse` group → daftar kosong** (P1 tidak punya baris). Validasi leaf di UI
   (`prd_warehouse.md` §5).
7. **Item tanpa baris reorder** → tidak ada entri untuk gudang tsb (jangan tampilkan `0`).
8. **Tanpa tiebreaker**, paginasi tidak stabil saat ada nilai kembar (mis. beberapa item bersaldo 0)
   → selalu `name asc` / `item_code asc`.
9. **`get_count` menghitung baris join**, bukan item unik — konsekuensi no. 1.
10. **Populasi = `Bin`** → item yang belum terdaftar di gudang **tidak tampil** (keputusan
    2026-10-22). Item bersaldo 0 yang punya baris `Bin` tetap tampil. Bila suatu saat ingin
    menampilkannya lagi, populasi harus kembali ke master `Item` (lihat riwayat revisi) — dan
    konsekuensinya sortir stok tidak bisa di server.
11. **`limit_page_length: 0`** memuat semua baris → gunakan hanya untuk P1/P4a/P5/P6 (bukan untuk
    daftar yang dipaginasi).
12. **Mode C** memuat seluruh dataset gudang (2 panggilan) → batasi ke katalog kecil.
13. **Tidak ada perbandingan kolom-dengan-kolom.** Filter kondisi stok F2/F3/F5/F6 **tidak bisa**
    dihitung server; mengirim nama kolom sebagai nilai filter akan **diam-diam salah** (§2.13 note 1).
14. **`Stock Ledger Entry` punya DocPerm sendiri.** User aplikasi perlu role `Stock User` atau
    `Accounts Manager`; tanpa itu F4/F5/F6/F8/F9 → `403` (bukan daftar kosong).
15. **F5/F6 = masuk zona hari ini DAN berakhir di zona** (bukan "pernah menyentuh", bukan kondisi
    sekarang). Item yang sudah di zona sejak kemarin lalu turun lagi → **F2 ya, F5 tidak**; item yang
    melewati maksimum dua hari lalu dan masih di atasnya → **F3 ya, F6 tidak**; item yang hari ini
    `10 → 3 → 8` (min `5`) → **F5 tidak** karena berakhir di luar zona.
16. **Nilai 0 pada min/max dianggap "tidak dikonfigurasi"** → item dikecualikan dari F2/F3/F5/F6.
    Tanpa aturan ini, `max = 0` membuat **semua** item bersaldo ≥ 0 terhitung "menyentuh maksimum".
17. **Ukuran cohort bisa besar** (mis. F9 untuk gudang sibuk = ribuan item) → `["name","in",[...]]`
    menjadi panjang. Pertimbangkan Mode C/custom API, atau pindahkan filter F1/F7 ke query `Bin`
    langsung (Mode B) agar tidak perlu `IN` besar.

### 5.4 Keputusan desain (ringkas & alasan)

| Keputusan | Alasan |
|---|---|
| Populasi baris = `Bin` (item × 1 gudang) | Satu-satunya sumber yang benar-benar menyatakan "item terdaftar di gudang"; sesuai keputusan user 2026-10-22 (item belum terdaftar tidak ditampilkan) |
| **Dua sumber paginasi** (Item vs Bin) | `get_list` tidak bisa `JOIN`; `order_by` hanya menerima kolom doctype sendiri / link / child → agar **baik** kolom `Item` **maupun** stok bisa jadi kunci sortir |
| Mode A: `name in [kode Bin]` | Menyatukan populasi gudang ke dalam query `Item` tanpa `JOIN`, tetap bisa filter atribut/group & paginasi |
| Mode B: `item_code in [kode hasil filter]` | Agar `ORDER BY actual_qty` sah (dilakukan di `Bin`), sementara filter item sudah "dibekukan" menjadi daftar kode |
| Mode C: muat semua (opsional) | Paling sederhana & bebas kolom sortir untuk katalog kecil; menghilangkan kompleksitas dua mode |
| Filter atribut **wajib berpasangan** | Satu-satunya bentuk bebas duplikat tanpa `group_by` (efek samping) |
| Total dari `get_count` (P3/P5) | `get_list` tidak mengembalikan total; total harus mencakup seluruh hasil filter |
| Tiebreaker `name asc` / `item_code asc` | Paginasi stabil meski ada nilai kembar |
| Semua parameter di **body JSON** | Struktur `filters`/`fields` kompleks + `IN (...)` panjang |
| Filter wajib `disabled=0` | Kebutuhan "hanya item aktif"; bisa dilonggarkan nanti tanpa mengubah pola |
| Filter kondisi stok = **2 tahap** (cohort → tampilkan) | Filter stok sebagian butuh data dari `Bin`/SLE yang tidak bisa di-`JOIN` ke `Item`; menghitung cohort lebih dulu membuat sortir & paginasi tetap bisa di server |
| F1/F7/F9 di **server**, F2/F3/F5/F6 di **frontend**, F4/F8 lewat **rumus** | Hanya kolom satu tabel yang bisa difilter server (`actual_qty`); perbandingan dua kolom mustahil di DSL. F5/F6 = masuk zona hari ini **dan berakhir di zona** (`opening` vs min/max + `Bin.actual_qty`) |
| `opening = Bin.actual_qty − Σ(SLE hari ini)` | Identitas `Bin = Σ SLE` terbukti (diff 0) → tidak perlu menarik riwayat SLE sebelum hari ini (payload jauh lebih kecil) |
| Min/max `0` = tidak dikonfigurasi | Mencegah `max = 0` membuat semua item bersaldo ≥ 0 dianggap "menyentuh maksimum" |

### 5.5 Endpoint terkait (dropdown filter)

| Kebutuhan | Dokumen |
|---|---|
| Dropdown Warehouse (leaf, per company, hormati User Permission) | [`prd_warehouse.md` §5](../setup/prd_warehouse.md) |
| Dropdown Item Group (leaf) | [`prd_item_group.md`](./prd_item_group.md), pola group induk → leaf: [`prd_item.md` §6.7](./prd_item.md) |
| Dropdown Item Attribute + nilainya | [`prd_item.md`](./prd_item.md) (master `Item Attribute`) |
| Daftar item + varian (hierarki template → varian) | [`prd_item.md` §4.5.1](./prd_item.md) |

### 5.6 Kapan perlu pindah ke custom API (`baseapp`)

| Aspek | Dokumen ini | Custom API (mis. `prd_stock_balance.md`) |
|---|---|---|
| Kode backend baru | tidak ada | ya (method whitelisted) |
| Sortir kolom `Item` + stok dalam satu query | tidak bisa (dua sumber paginasi) | `LEFT JOIN tabBin`, `ORDER BY` bebas |
| Panggilan saat muat awal (Mode A) | 3 (P1, P2, P3) | 1 (+1 total) |
| Panggilan per halaman (Mode B) | 3 (P4b, P4c + P5 sekali) | 1 |
| Panggilan "resolve kode" | ada (P4a) | tidak perlu |
| Permission | otomatis (DocPerm + User Permission) | harus diperiksa manual |
| Cocok untuk | semua ukuran katalog (Mode C untuk katalog kecil) | katalog besar + butuh sortir/filter campuran dalam 1 panggilan |

---

## 6. Penanganan error umum

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | User tidak punya hak baca `Item`/`Bin` | `{"exc_type": "PermissionError", "message": "Not permitted"}` |
| 417 | Field tidak dikenal di `fields` **atau** `filters` (typo, `item_group_name` polos, filter Item di doctype `Bin`) | `{"exc_type": "DataError", "message": "Field not permitted in query: <field>"}` |
| 417 | `order_by` memakai kolom yang bukan milik doctype tsb (mis. `actual_qty` pada `Item`) | `{"exc_type": "PermissionError", "message": "You do not have permission to access field: Item.actual_qty"}` |
| 417 | `order_by` memakai nama tabel eksplisit (`tabBin.actual_qty`) | `{"exc_type": "ValidationError", "message": "Invalid field format in Order By: tabBin.actual_qty. Use 'field', 'link_field.field', or 'child_table.field'."}` |
| 417 | `fields`/`order_by` di doctype `Bin` memakai kolom `Item` (mis. `item_name`) | `DataError` / `PermissionError` (kolom tidak ada di `Bin`) |
| 417 | Fungsi agregat ditulis sebagai **string** | `{"exc_type": "ValidationError", "message": "SQL functions are not allowed as strings in SELECT: sum(actual_qty) as total. Use dict syntax like {'COUNT': '*'} instead."}` |
| 417 | `filters` bukan list/dict valid (JSON rusak) | `{"exc_type": "ValidationError", ...}` |
| 500 | Nested dict untuk **link field** (`{"item_group": ["item_group_name"]}`) | `{"exc_type": "AttributeError", "message": "'ChildQuery' object has no attribute 'fieldname'"}` |
| 500 | Galat tak terduga lain | `{"exc_type": "...", "exception": "...", "message": "..."}` |
| — | Tidak ada item cocok / P1 kosong | **Bukan error** → `{"message": []}` (list) / `{"message": 0}` (count) |
| — | `warehouse` **group** (`is_group=1`) | **Bukan error** → P1 kosong → daftar kosong |
| — | Halaman melebihi total (`limit_start` terlalu besar) | **Bukan error** → `{"message": []}` |
| — | User Permission membatasi gudang | **Bukan error** → P1 terbatas/kosong |
| 403 | User tidak punya role baca `Stock Ledger Entry` (`Stock User` / `Accounts Manager`) → filter F4/F5/F6/F8/F9 gagal | `{"exc_type": "PermissionError", "message": "Not permitted"}` |

> **Catatan:**
> - Body error berbentuk `{ "exc_type": ..., "exception": ..., "message": ... }` (standar
>   `/api/method/...`); pesan di atas adalah pesan **asli** Frappe/ERPNext.
> - Kesalahan frontend tersering: (1) mengirim `order_by` stok ke P2 (Mode A) — harus pindah ke
>   Mode B; (2) tidak mengirim tiebreaker sehingga paginasi tidak stabil; (3) lupa pasangan filter
>   atribut; (4) menaruh filter Item di panggilan `Bin`; (5) memakai `["item_group_name","=",...]`
>   alih-alih `["item_group.item_group_name","=",...]`; (6) mengirim `IN []` saat daftar kode kosong.
> - **Bukan error, tetapi salah:** mengirim nama kolom sebagai **nilai** filter
>   (mis. `["actual_qty","<=","warehouse_reorder_level"]`) diterima server (HTTP 200) karena nilai
>   itu diperlakukan sebagai **string** → hasil tidak benar. Perbandingan min/max **wajib** di
>   frontend (§2.13).

---

## 7. Rencana koleksi Postman

Folder **`21. Stock Report`** sudah ditambahkan ke koleksi `docs/postman/postman_erpnext_api.json`
(Collection v2.1) — **22 request**, sudah di-*smoke test* terhadap site dev:

| Request | Isi | § |
|---|---|---|
| `21.1` | **P1** — `Bin` gudang (`limit_page_length: 0`), simpan `item_code` + stok | §4.1 |
| `21.2` | **Mode A** — `Item` halaman, `order_by "item_name asc, name asc"` | §4.2 |
| `21.3` | **Mode A** — total (`Item.get_count`, filter identik `21.2`) | §2.4 |
| `21.4` | **Mode A** — sortir `item_group.item_group_name asc` | §4.3 |
| `21.5` | **Mode A** — halaman 2 (`limit_start`) + cek `total_count` | §4.7 |
| `21.6` | **Mode B** — P4a resolve kode (`fields:["name"]`, `limit 0`) | §4.4 |
| `21.7` | **Mode B** — P4b halaman terurut `actual_qty desc, item_code asc` | §4.4 |
| `21.8` | **Mode B** — P5 total (`Bin.get_count`, filter identik `21.7`) | §2.6 |
| `21.9` | **Mode B** — "hanya ada stok" (`actual_qty > 0`) + sortir `asc` | §4.5 |
| `21.10` | **Mode C** — P1 + P6 (muat semua) untuk katalog kecil | §4.8 |
| `21.11` | Filter pencarian + atribut **berpasangan** + item group (dipakai di Mode A & B) | §4.6 |
| `21.12` | **Kasus negatif**: `order_by` stok di `Item` → `417 PermissionError` | §6 |
| `21.13` | **Kasus negatif**: `order_by "tabBin.actual_qty"` → `417 ValidationError` | §6 |
| `21.14` | **Kasus negatif**: filter Item (`is_stock_item`) di doctype `Bin` → `417 DataError` | §6 |
| `21.15` | **Kasus negatif**: filter `attribute_value` tanpa pasangan → duplikat (12 vs 9) | §2.7 / §5.3 no. 1 |
| `21.16` | **Kasus negatif**: agregat bentuk string → `417 ValidationError` | §6 |
| `21.17` | **F1 stok kosong** — `Bin` + `["actual_qty","=",0]` | §4.10 |
| `21.18` | **F7 stok minus** — `Bin` + `["actual_qty","<",0]` | §4.10 |
| `21.19` | **F9 stok berubah** — SLE hari ini, `group_by item_code` (+ test script → `sr_changed_codes`) | §4.10 |
| `21.20` | **R** — level reorder per gudang (nested `reorder_levels`) | §2.13 / §4.11 |
| `21.21` | **T (rinci)** — SLE hari ini dengan `qty_after_transaction` (opsional, untuk **waktu kejadian**); perhitungan F4/F5/F6/F8/F9 cukup `21.19` | §2.13 / §4.11 |
| `21.22` | **Kasus negatif**: filter nilai = nama kolom (`["actual_qty","<=","warehouse_reorder_level"]`) → HTTP 200 tetapi hasil salah | §2.13 note 1 |

**Variabel yang perlu diisi (Collection Variables):**

- `sr_company` — company (mis. `PT Maju Jaya`) — hanya untuk dokumentasi/label
- `sr_warehouse` — warehouse **leaf** (mis. `Toko Cikarang - PTMJ`)
- `sr_item_name_search` — kata kunci pencarian nama (mis. `kemeja`)
- `sr_attribute` / `sr_attribute_value` — pasangan atribut (mis. `Color` / `["Merah"]`)
- `sr_item_group_name` — label Item Group leaf (mis. `Pakaian`)
- `sr_page_length` — jumlah baris per halaman (disarankan `50`)
- `sr_limit_start` — offset halaman (mis. `0`, `50`)
- `sr_sort_item` — kolom sortir Mode A (mis. `item_name asc, name asc`)
- `sr_sort_stock` — kolom sortir Mode B (mis. `actual_qty desc, item_code asc`)
- `sr_today` — tanggal untuk filter kondisi stok berbasis hari (`F4`/`F5`/`F6`/`F8`/`F9`); di site dev
  isi dengan tanggal yang punya data SLE (mis. `2020-12-01`), di produksi = hari ini
- `sr_bin_codes`, `sr_bin_codes_50`, `sr_filtered_codes`, `sr_page_codes`, `sr_changed_codes` —
  **diisi otomatis oleh test script** request `21.1`, `21.6`, `21.7`, `21.19` (jangan diisi manual).
  Nilainya berupa **string JSON array** (mis. `["A","B"]`) dan dipakai **tanpa tanda kutip** di body,
  mis. `["name","in",{{sr_bin_codes}}]`.

> Request `21.1`–`21.22` memerlukan app **`baseapp`** terpasang (kolom
> `is_product_bundle`/`is_dynamic_product_bundle`/`max_stock_level`); sisanya murni ERPNext.
> Request `21.17`–`21.21` tambahan memerlukan role baca `Stock Ledger Entry`.

> **Dokumen terkait:** **[prd_item.md](./prd_item.md)** — master item (varian, atribut, reorder level,
> item group, hashtag), **[prd_item_group.md](./prd_item_group.md)** — Item Group (leaf/tree),
> **[prd_warehouse.md](../setup/prd_warehouse.md)** — Warehouse (leaf, User Permission),
> **[prd_stock_balance.md](./prd_stock_balance.md)** — laporan saldo periode (custom API `baseapp`),
> **[prd_stock_ledger.md](./prd_stock_ledger.md)** — kartu stok/mutasi & batch,
> **[prd_stock_entry.md](./prd_stock_entry.md)** / **[prd_stock_reconciliation.md](./prd_stock_reconciliation.md)**
> — dokumen pengubah stok, **[prd_baseapp.md](../baseapp/prd_baseapp.md)** — custom field `baseapp`,
> **[prd_oauth.md](../prd_oauth.md)** — autentikasi.
