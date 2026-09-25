# PRD — Custom API Laporan Stock Balance (ERPNext / Frappe — custom app `baseapp`)

> Dokumen spesifikasi pemanggilan **custom API laporan Stock Balance** yang dibuat di custom app
> **`baseapp`**, diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Base App (`baseapp`) — path publik `baseapp.api`
  (implementasi `baseapp/baseapp/api/stock_balance.py`; re-export `baseapp/baseapp/api/__init__.py`)
- **Sumber logika:** report bawaan ERPNext **Stock Balance** —
  `erpnext.stock.report.stock_balance.stock_balance` (API ini **meng-*reuse*** logika report tsb,
  lihat §5)
- **Doctype yang dibaca:** `Stock Ledger Entry` (SLE), `Bin`, `Item`, `Item Group`, `Warehouse`,
  `Price List`, `Item Price` (+ saldo `Stock Closing Balance` bila site memakai Stock Closing Entry)
- **Versi API:** `/api/method/...` (API v1) — parameter dikirim di **query string**; nama method
  adalah fungsi whitelisted milik custom app (**bukan** `frappe.client.*`)
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format respons:** JSON, dibungkus `"message"`
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

> **Prasyarat:** custom app **`baseapp`** harus terpasang di site
> (`bench --site <site> install-app baseapp`). Konteks app & module *Base App*:
> [`prd_item_dynamic_product_bundle.md`](./prd_item_dynamic_product_bundle.md).

> **Sifat dokumen ini sama dengan [`prd_stock_ledger.md`](./prd_stock_ledger.md): read-only.**
> Tidak ada CREATE/UPDATE/DELETE/SUBMIT/CANCEL — hanya **2 endpoint GET** (§1). Berbeda dari PRD
> lain, endpoint di sini **bukan** `frappe.client.*`, melainkan custom API yang **dibuat di app
> `baseapp`** (§5). Karena itu dokumen ini memuat dua hal: **kontrak API untuk tim UI** (§1–§4, §6)
> dan **spesifikasi implementasi** (§5).

> **Singkatnya:** pemeriksaan stok per **item** pada **1 warehouse** untuk **1 periode** —
> kolom saldo awal/in/out/saldo akhir (qty & nilai, mengikuti report Stock Balance) **plus**
> harga jual opsional dari Price List terpilih, lengkap dengan paginasi dan endpoint total.

---

## 1. Ruang lingkup

| # | Endpoint (GET `/api/method/...`) | Operasi | § |
|---|---|---|---|
| 1 | `baseapp.api.get_stock_balance_paginated` | Daftar baris laporan Stock Balance (1 baris = 1 item) untuk **1 company + 1 warehouse** + rentang tanggal, **ber-paginasi**, opsional menyertakan **Selling Price** dari `price_list` terpilih | §4.1–§4.4 |
| 2 | `baseapp.api.get_stock_balance_totals` | **Total** `bal_qty`, `bal_val`, dan Selling Price dari **seluruh** baris sesuai filter yang sama — **tidak dibatasi paginasi** | §4.5 |

> **Konvensi pemanggilan (penting):**
> - Parameter dikirim sebagai **query string** (`?company=...&warehouse=...`), sesuai pola custom API
>   yang diminta. Method whitelisted Frappe juga menerima **POST dengan body JSON** (karena
>   `frappe.form_dict` diisi dari body) — frontend bebas memilih, isi parameternya identik.
> - **Format respons:** `/api/method/...` membungkus hasil di **`"message"`** (bukan `"data"` seperti
>   `/api/resource/...`). Di dalam `message` ada amplop milik API ini: `data` + metadata (§2.2).
> - Endpoint **tidak menyediakan** CREATE/UPDATE/DELETE/export maupun **dropdown pendukung** (Item,
>   Warehouse, Company, Price List) — pemanggilan dropdown memakai PRD masing-masing
>   (`prd_item.md`, `../setup/prd_warehouse.md`, `prd_item_price.md`).
> - Data yang dibaca **hanya** yang boleh dilihat user — API memeriksa permission (§5.4).

### 1.1 Perbedaan perilaku vs report bawaan "Stock Balance"

| Aspek | Report bawaan (Desk) | Custom API (dokumen ini) |
|---|---|---|
| Akses | query report di Desk | REST `/api/method/...` (mobile/web/API client) |
| Paginasi | seluruh baris sekaligus | `page`/`page_length` + metadata (`total_count`, `total_pages`) |
| Warehouse | multi-warehouse / opsional | **wajib diisi, tepat 1 warehouse** (menjaga ukuran hasil & kinerja) |
| Baris saldo 0 | disembunyikan (kecuali filter `include_zero_stock_items`) | **ditampilkan** — termasuk item yang ada di `Bin` walau saldo 0 (§2.3 no. 3) |
| Harga jual | tidak ada | kolom `selling_price` + `selling_value` bila `price_list` dikirim |
| Item non-stok | disaring lewat `is_stock_item` pada dropdown item | **selalu** dikecualikan (`is_stock_item=1`) + `disabled=0` |
| Total | baris "total" di UI | endpoint terpisah `get_stock_balance_totals` (`bal_qty`, `bal_val`, Selling Price) |

> Perbedaan lain yang penting: report bawaan memakai **warehouse turunan** bila user memilih node
> group (lihat `apply_warehouse_filter`). Custom API **tidak** menerima group — `warehouse` harus
> **leaf** (`is_group=0`), bila tidak → `417` (§6).

---

## 2. Parameter & field

### 2.1 Parameter / filter

Simbol: **🔴 = wajib**, **🟠 = opsional**.

| Status | Parameter | Tipe / Link | Default | Keterangan |
|---|---|---|---|---|
| 🔴 **Wajib** | `company` | Link → Company | — | Company yang dibaca. Kosong / tidak ada → `417`. Nilai company menentukan **mata uang** hasil (`Company.default_currency`). |
| 🔴 **Wajib** | `warehouse` | Link → Warehouse | — | **Tepat satu** warehouse, harus **leaf** (`is_group=0`) & milik `company` tsb. Dikirim lebih dari satu nilai / berupa group → `417` (§6). |
| 🟠 Opsional | `from_date` | Date (`YYYY-MM-DD`) | **hari ini** | Awal periode (inklusif). Saldo **sebelum** `from_date` masuk kolom `opening_*`. |
| 🟠 Opsional | `to_date` | Date (`YYYY-MM-DD`) | **hari ini** | Akhir periode (inklusif). `from_date > to_date` → `417`. |
| 🟠 Opsional | `item_code` | Item (boleh banyak) | semua item stok | Daftar `item_code` yang hanya boleh muncul: `item_code=ITEM-A,ITEM-B` **atau** parameter diulang (`item_code=ITEM-A&item_code=ITEM-B`). Kosong = semua item stok yang punya baris di `Bin` warehouse tsb. |
| 🟠 Opsional | `item_group` | Item Group (boleh banyak) | semua group | Bila diisi, **termasuk seluruh sub-group di bawahnya** (descendants, pola `lft`/`rgt` — sama seperti report bawaan). Boleh dikombinasikan dengan `item_code` (hasil = irisan). |
| 🟠 Opsional | `price_list` | Link → Price List | — | Bila diisi → setiap baris mendapat **`selling_price`** & **`selling_value`** (§2.5). Bila kosong → kedua field bernilai `null`. Price List yang tidak ada → `417`. |
| 🟠 Opsional | `page` | Int ≥ 1 | `1` | Halaman ke-`page` (endpoint a). `page` melebihi jumlah halaman → `data: []` (**bukan** error). |
| 🟠 Opsional | `page_length` | Int `1..500` | `50` | Jumlah baris per halaman (endpoint a). Di luar rentang → `417`. Tidak ada mode "ambil semua" — gunakan `page_length` maksimum + paginasi. |
| 🟠 Opsional | `order_by` | Select | `item_code` | Urutan baris (endpoint a). Pilihan: `item_code`, `item_name`, `bal_qty`, `bal_val`, `selling_value`. Nilai lain → `417`. Selalu diakhiri *tiebreaker* `item_code asc` agar paginasi stabil. |
| 🟠 Opsional | `order` | `asc` / `desc` | `asc` | Arah urutan untuk `order_by`. |

> **Tidak berlaku di endpoint b:** `page`, `page_length`, `order_by`, `order` — endpoint b selalu
> menghitung **seluruh** baris yang cocok filter (tidak "dilock" paginasi). Mengirim parameter tsb
> ke endpoint b akan **diabaikan** (bukan error).

### 2.2 Bentuk respons

**Endpoint a — `get_stock_balance_paginated`:**

```jsonc
{
  "message": {
    "company": "PT Maju Jaya",
    "warehouse": "Toko Cikarang - PTMJ",
    "from_date": "2026-09-01",
    "to_date": "2026-09-30",
    "currency": "IDR",                    // Company.default_currency
    "price_list": "Harga Jual Retail",     // null bila parameter price_list tidak dikirim
    "price_list_currency": "IDR",          // Price List.currency — null bila price_list kosong
    "page": 1,
    "page_length": 50,
    "total_count": 128,                    // jumlah SELURUH baris (tanpa paginasi)
    "total_pages": 3,
    "has_next_page": true,
    "data": [ /* array baris — §2.3 */ ]
  }
}
```

**Endpoint b — `get_stock_balance_totals`:**

```jsonc
{
  "message": {
    "company": "PT Maju Jaya",
    "warehouse": "Toko Cikarang - PTMJ",
    "from_date": "2026-09-01",
    "to_date": "2026-09-30",
    "currency": "IDR",
    "price_list": "Harga Jual Retail",
    "price_list_currency": "IDR",
    "total_count": 128,                    // jumlah baris yang disisir (info, bukan paginasi)
    "total_bal_qty": 134.0,
    "total_bal_val": 639000.0,
    "total_selling_price": 840000.0        // null bila price_list tidak dikirim
  }
}
```

> Metadata `company`/`warehouse`/`from_date`/`to_date`/`currency`/`price_list` dikirim balik agar
> UI dapat memastikan **parameter efektif** yang dipakai server (khususnya `from_date`/`to_date`
> yang default-nya "hari ini" bila tidak dikirim).

### 2.3 Field per baris (`data[]`)

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| ⚪ Output | `item_code` | Link → Item | Kode item (kunci baris). |
| ⚪ Output | `item_name` | Data | Nama item (dari master `Item`). |
| ⚪ Output | `item_group` | Link → Item Group | Item Group **leaf** dari item tsb. |
| ⚪ Output | `stock_uom` | Link → UOM | Satuan stok item — **semua kolom qty memakai satuan ini**. |
| ⚪ Output | `warehouse` | Link → Warehouse | Selalu = parameter `warehouse` (dikirim per baris agar UI/ekspor tidak perlu menempelkan konteks). |
| ⚪ Output | `opening_qty` | Float | Saldo qty **sebelum** `from_date`. Label desk: *Opening Qty*. |
| ⚪ Output | `opening_val` | Currency | Nilai saldo sebelum `from_date`. Label desk: *Opening Value*. |
| ⚪ Output | `in_qty` | Float | Total qty **masuk** pada periode (`actual_qty` ≥ 0). |
| ⚪ Output | `in_val` | Currency | Total nilai masuk pada periode. |
| ⚪ Output | `out_qty` | Float | Total qty **keluar** pada periode (nilai absolut). |
| ⚪ Output | `out_val` | Currency | Total nilai keluar pada periode (nilai absolut). |
| ⚪ Output | `bal_qty` | Float | **Saldo qty akhir** (= `opening_qty + in_qty − out_qty`). Label desk: *Balance Qty*. |
| ⚪ Output | `bal_val` | Currency | **Nilai saldo akhir** (= `opening_val + in_val − out_val`). Label desk: *Balance Value*. |
| ⚪ Output | `val_rate` | Currency | *Valuation Rate* terakhir pada/ sebelum `to_date` (0 bila tidak ada transaksi). |
| ⚪ Output | `selling_price` | Currency / `null` | **Harga jual satuan** dari `price_list` terpilih (§2.5) — `null` bila `price_list` tidak dikirim **atau** item tidak punya harga di price list tsb. Label: **Selling Price**. |
| ⚪ Output | `selling_value` | Currency / `null` | **Nilai jual baris** = `selling_price × bal_qty` (§2.5) — `null` bila `selling_price` `null`. |
| ⚪ Output | `currency` | Link → Currency | `Company.default_currency` (mata uang `opening_val`/`in_val`/`out_val`/`bal_val`/`val_rate`). |

> **Catatan angka:** API **tidak membulatkan** qty/nilai (dikirim sesuai presisi yang tersimpan).
> Pembulatan tampilan (mis. 2 desimal untuk currency) adalah tanggung jawab UI.
> `selling_price`/`selling_value` memakai mata uang **Price List** (`price_list_currency`), yang
> **bisa berbeda** dari `currency` — **tanpa konversi kurs** (§2.5 no. 5).

### 2.4 Catatan penting

1. **Hanya SLE efektif yang dihitung.** Sumber angka adalah `Stock Ledger Entry` dengan filter
   `is_cancelled = 0` & `docstatus < 2` (pola wajib yang sama dengan
   [`prd_stock_ledger.md` §2.1 no. 3](./prd_stock_ledger.md) — transaksi yang di-cancel ditandai
   `is_cancelled=1` + SLE pembalik). **Tidak ada** opsi menampilkan baris yang dibatalkan.
2. **Baris = pasangan item × warehouse, bukan per transaksi.** Untuk 1 item di 1 warehouse, seluruh
   SLE-nya dirangkum menjadi satu baris (opening/in/out/bal). Ini adalah bentuk laporan
   *stock balance*, bukan kartu stok per transaksi (kartu stok: `prd_stock_ledger.md` §4.2).
3. **Baris bersaldo 0 tetap tampil.** Sumber daftar baris adalah `tabBin` (item stok yang
   **pernah** punya catatan stok di warehouse tsb), lalu nilainya diambil dari SLE. Item yang ada
   di `Bin` tapi tidak punya SLE pada/ sebelum `to_date` dikembalikan dengan seluruh nilai `0`
   (termasuk `opening_*`). Item non-stok (`is_stock_item=0`) dan item `disabled=1` **tidak**
   dikembalikan.
4. **Periode & opening.** `opening_*` = akumulasi SLE **sebelum** `from_date`. Bila site memakai
   *Stock Closing Entry*, saldo pembuka diambil dari **Stock Closing Balance** terakhir (persis
   seperti report bawaan) sehingga tidak perlu memindai seluruh riwayat SLE. Konsekuensi praktis:
   **`from_date` terlalu jauh ke belakang** membuat query lebih berat (SLE besar) — kirim rentang
   seperlunya.
5. **`warehouse` wajib leaf.** Nilai group (`is_group=1`) tidak diterima; UI harus memakai dropdown
   warehouse **leaf** per company (`prd_warehouse.md` §5) dan mengirim salah satunya. Dengan begitu
   hasil selalu terikat pada satu gudang fisik/leaf, bukan agregat subtree.
6. **Penjumlahan `total_bal_qty` lintas item tidak selalu bermakna bisnis.** Baris dapat berisi item
   dengan `stock_uom` berbeda (mis. `Nos` + `Kg`), sehingga `total_bal_qty` hanya berguna bila
   filter item/UOM membatasi satu satuan yang sama. `total_bal_val` & `total_selling_price`
   (keduanya nilai uang) aman dijumlahkan.
7. **Harga jual hanya dari `Item Price` umum.** Karena laporan ini bukan transaksi, tidak ada
   `customer`/`supplier` → hanya `Item Price` **tanpa** customer/supplier yang dipakai
   (deterministik), lihat §2.5.

### 2.5 Arti & aturan "Selling Price"

Bila parameter `price_list` dikirim, API menambahkan **dua** field pada setiap baris
(lihat contoh angka di §4.2):

| Field | Arti | Rumus |
|---|---|---|
| `selling_price` | **harga jual satuan** item (label *Selling Price*), diambil dari `tabItemPrice.price_list_rate` untuk `price_list` terpilih, dikonversi ke `stock_uom` | `price_list_rate` (hasil resolusi §2.5 A–C) |
| `selling_value` | **nilai stok bila dijual** pada harga tsb | `selling_price × bal_qty` |

dan pada endpoint b: `total_selling_price` = **Σ `selling_value`** seluruh baris.

**Contoh kasus** — item `Air Mineral 600ml` (`stock_uom` = `Nos`), Price List `Harga Jual Retail`
dengan `price_list_rate` = **Rp 6.000 / Nos**; baris laporan: `bal_qty` = 40 Nos,
`valuation_rate` = Rp 4.500 → `bal_val` = Rp 180.000:

| Kolom | Nilai | Arti |
|---|---|---|
| `bal_qty` | 40 | saldo stok (Nos) |
| `bal_val` | 180.000 | nilai stok pada **harga pokok** (valuation) |
| `selling_price` | 6.000 | harga jual **per satuan** |
| `selling_value` | 240.000 | nilai stok pada **harga jual** (40 × 6.000) |

- `240.000 − 180.000 = 60.000` → indikasi potensi margin kotor **bila** seluruh saldo terjual.
- Karena `total_selling_price` = Σ `selling_value`, footer **sama** dengan penjumlahan kolom
  `selling_value` yang tampil di UI (tidak membingungkan seperti menjumlahkan harga satuan).
- Bila `bal_qty` = 0 → `selling_value` = 0 **tetapi `selling_price` tetap terisi** (harga satuannya
  masih bisa ditampilkan UI). Bila `bal_qty` negatif (stok minus) → `selling_value` ikut negatif.

**A–C. Resolusi harga (urutan yang dipakai):**

**A. Harga per `stock_uom`** — pakai helper ERPNext
`erpnext.stock.get_item_details.get_item_price(pctx, item_code)` dengan
`pctx = {"price_list": <price_list>, "uom": <item.stock_uom>, "transaction_date": <to_date>}`.
Helper ini (terverifikasi pada source v16) hanya mengembalikan `Item Price` yang:

- `price_list` = parameter, dan `uom` = `stock_uom` **atau** `uom` kosong,
- **tanpa** `customer` dan **tanpa** `supplier` (karena tidak ada parti pada konteks laporan),
- `valid_from ≤ to_date ≤ valid_upto` (bila `valid_from`/`valid_upto` kosong → dianggap tak
  terbatas, memakai default `2000-01-01` / `2500-12-31`),
- diurutkan `valid_from desc` → `batch_no desc` → `uom desc`, diambil **1 baris**.

**B. Fallback harga UOM lain (konversi ke `stock_uom`)** — bila langkah A tidak menemukan harga,
cari `Item Price` (kriteria `price_list`/item/validitas/parti sama) yang `uom`-nya salah satu UOM
**alternatif** item (dari child `Item.uoms` = `UOM Conversion Detail`, bukan `stock_uom`), lalu:

```text
selling_price = price_list_rate ÷ conversion_factor
```

karena `conversion_factor` pada Item = **jumlah `stock_uom` per 1 UOM tersebut** (mis. `Dus = 12`
→ `Rp 60.000 / Dus` = `Rp 5.000 / Nos`). Bila UOM tsb tidak punya `conversion_factor` (> 0) →
harga dianggap **tidak ada**.

> ⚠️ Langkah B adalah **tambahan** di atas perilaku helper bawaan (helper `get_price_list_rate_for`
> hanya mengonversi pada arah sebaliknya, yaitu dari UOM transaksi ke UOM price list). Karena
> laporan ini selalu memakai `stock_uom`, konversi di atas **wajib** diimplementasikan eksplisit —
> lihat §5.5.

**C. Bila tetap tidak ditemukan** → `selling_price = null` **dan** `selling_value = null` pada baris
tsb (bukan `0` — UI dapat membedakan "tidak ada harga" dari "nilai nol"). Untuk total,
baris `null` disumbangkan sebagai **0**; bila `price_list` tidak dikirim sama sekali →
`total_selling_price = null`.

**Catatan tambahan harga:**

1. **Tidak ada konversi kurs.** Bila `Price List.currency` ≠ `Company.default_currency`, angka
   `selling_*` tetap dalam mata uang price list. UI dapat menampilkan `price_list_currency` (§2.2)
   sebagai satuan/label.
2. **`packing_unit` diabaikan.** Item Price ber-`packing_unit` tetap dipakai nilai `price_list_rate`,
   tanpa validasi kelipatan qty (`check_packing_list` tidak relevan untuk baris saldo agregat).
3. **Item template/varian.** Pencarian harga dilakukan per `item_code` baris (tanpa *fallback*
   varian → template). Bila item memakai varian dan harga hanya ada di template, baris varian akan
   bernilai `null` — sementara report bawaan Item Price tetap menyimpan harga per item varian.
4. **Dibaca ulang setiap request** (tanpa cache) agar perubahan harga langsung terlihat.

### 2.6 Tabel-tabel terkait

| Tabel (`tab...`) | Doctype | Peran di API ini |
|---|---|---|
| `tabStock Ledger Entry` | Stock Ledger Entry | **Sumber angka** qty & nilai (`actual_qty`, `qty_after_transaction`, `stock_value`, `valuation_rate`, `stock_value_difference`) — selalu difilter `is_cancelled=0`. Detail & kolom: [`prd_stock_ledger.md`](./prd_stock_ledger.md). |
| `tabBin` | Bin | **Sumber daftar baris**: pasangan `item_code` + `warehouse` yang punya catatan stok (termasuk saldo 0). Juga sumber `valuation_rate` cadangan bila tidak ada SLE pada/ sebelum `to_date`. |
| `tabItem` (+ `tabUOM Conversion Detail`) | Item | Nama & group item (`item_name`, `item_group`, `stock_uom`), filter `is_stock_item=1`/`disabled=0`, dan UOM alternatif untuk konversi harga (§2.5 B). Detail: [`prd_item.md`](./prd_item.md). |
| `tabItem Group` | Item Group | Resolusi sub-group bila `item_group` dikirim (descendants `lft`/`rgt`). Detail: [`prd_item_group.md`](./prd_item_group.md). |
| `tabWarehouse` | Warehouse | Validasi `warehouse` (ada, `is_group=0`, milik `company`). Detail: [`prd_warehouse.md`](../setup/prd_warehouse.md). |
| `tabItem Price` / `tabPrice List` | Item Price / Price List | Sumber **Selling Price** (`price_list_rate`) & mata uangnya (`Price List.currency`). Detail: [`prd_item_price.md`](./prd_item_price.md). |
| `tabStock Closing Entry` (+ `tabStock Closing Balance`) | Stock Closing Entry | Bila ada, dipakai sebagai **saldo pembuka** periode (mempercepat query & menyamakan hasil dengan report bawaan) — dibaca read-only lewat logika report. |

> API ini **tidak menulis** ke tabel mana pun (read-only), dan tidak membuat doctype/custom field
> baru. Yang ditambahkan hanyalah 2 method whitelisted di app `baseapp` (§5).

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Alternatif (mis. untuk integrasi server-to-server) dapat memakai API key/secret user
(`Authorization: token <api_key>:<api_secret>`) — pola pembuatannya ada di PRD user. Namun
**cara yang dianjurkan** untuk aplikasi klien tetap **OAuth 2.0**; detail lengkap setup OAuth
Client ada di **[`prd_oauth.md`](../prd_oauth.md)**.

> Karena ini custom method (bukan `frappe.client.*`), **permission tidak diperiksa otomatis** oleh
> endpoint — API memeriksa sendiri (role baca SLE & akses warehouse user), lihat §5.4 & §6.

---

## 4. Contoh pemakaian

> **Alur umum untuk UI:**
> 1. User memilih `company` + `warehouse` (leaf) + rentang tanggal → panggil endpoint a (§4.1).
> 2. Bila user memilih Price List → sertakan `price_list` (§4.2) dan tampilkan kolom
>    **Selling Price** + **Nilai Jual**.
> 3. Persempit dengan `item_code`/`item_group` bila perlu (§4.3); pindah halaman via `page`
>    memakai metadata (§4.4).
> 4. Tampilkan footer/total memakai endpoint b (§4.5) — total **selalu** mencakup seluruh baris,
>    bukan hanya halaman yang tampil.

### 4.1 Daftar saldo stok — dasar (tanpa harga jual)

```bash
curl -X GET "https://site-anda.com/api/method/baseapp.api.get_stock_balance_paginated?company=PT%20Maju%20Jaya&warehouse=Toko%20Cikarang%20-%20PTMJ&from_date=2026-09-01&to_date=2026-09-30&page=1&page_length=50" \
  -H "Authorization: Bearer <access_token>"
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": {
    "company": "PT Maju Jaya",
    "warehouse": "Toko Cikarang - PTMJ",
    "from_date": "2026-09-01",
    "to_date": "2026-09-30",
    "currency": "IDR",
    "price_list": null,
    "price_list_currency": null,
    "page": 1,
    "page_length": 50,
    "total_count": 2,
    "total_pages": 1,
    "has_next_page": false,
    "data": [
      {
        "item_code": "Air Mineral 600ml",
        "item_name": "Air Mineral 600ml",
        "item_group": "Minuman",
        "stock_uom": "Nos",
        "warehouse": "Toko Cikarang - PTMJ",
        "opening_qty": 100.0,
        "opening_val": 450000.0,
        "in_qty": 20.0,
        "in_val": 90000.0,
        "out_qty": 10.0,
        "out_val": 45000.0,
        "bal_qty": 110.0,
        "bal_val": 495000.0,
        "val_rate": 4500.0,
        "selling_price": null,
        "selling_value": null,
        "currency": "IDR"
      },
      {
        "item_code": "Air Mineral 1500ml",
        "item_name": "Air Mineral 1500ml",
        "item_group": "Minuman",
        "stock_uom": "Nos",
        "warehouse": "Toko Cikarang - PTMJ",
        "opening_qty": 0.0,
        "opening_val": 0.0,
        "in_qty": 24.0,
        "in_val": 144000.0,
        "out_qty": 0.0,
        "out_val": 0.0,
        "bal_qty": 24.0,
        "bal_val": 144000.0,
        "val_rate": 6000.0,
        "selling_price": null,
        "selling_value": null,
        "currency": "IDR"
      }
    ]
  }
}
```

> `from_date`/`to_date` boleh tidak dikirim → server memakai **tanggal hari ini** untuk keduanya dan
> mengembalikan nilai efektifnya di metadata (`from_date`/`to_date` pada respons).

### 4.2 Dengan harga jual (`price_list`)

Price List `Harga Jual Retail` berisi: `Air Mineral 600ml` = Rp 6.000/Nos, `Air Mineral 1500ml` =
Rp 7.500/Nos. Tambahkan parameter `price_list`:

```bash
curl -X GET "https://site-anda.com/api/method/baseapp.api.get_stock_balance_paginated?company=PT%20Maju%20Jaya&warehouse=Toko%20Cikarang%20-%20PTMJ&from_date=2026-09-01&to_date=2026-09-30&price_list=Harga%20Jual%20Retail&page=1&page_length=50" \
  -H "Authorization: Bearer <access_token>"
```

**Contoh respons sukses (HTTP 200)** — hanya bagian yang berubah yang ditampilkan:

```json
{
  "message": {
    "price_list": "Harga Jual Retail",
    "price_list_currency": "IDR",
    "data": [
      {
        "item_code": "Air Mineral 600ml",
        "bal_qty": 110.0,
        "bal_val": 495000.0,
        "val_rate": 4500.0,
        "selling_price": 6000.0,
        "selling_value": 660000.0,
        "currency": "IDR"
      },
      {
        "item_code": "Air Mineral 1500ml",
        "bal_qty": 24.0,
        "bal_val": 144000.0,
        "val_rate": 6000.0,
        "selling_price": 7500.0,
        "selling_value": 180000.0,
        "currency": "IDR"
      }
    ]
  }
}
```

> `selling_value` = `selling_price × bal_qty` → `6.000 × 110 = 660.000`, `7.500 × 24 = 180.000`.
> Item yang tidak punya harga di price list tsb → `selling_price: null`, `selling_value: null`
> (baris tetap muncul).
>
> **Jika Item Price hanya tersedia untuk UOM lain**, mis. `Air Mineral 1500ml` dihargai
> `Rp 90.000 / Karton` dengan faktor konversi `1 Karton = 12 Nos`, maka
> `selling_price = 90.000 ÷ 12 = 7.500` (§2.5 B) — nilai yang sama seperti contoh di atas.

### 4.3 Menyaring item (`item_code` digabung sub-group)

```bash
# (a) item tertentu saja — boleh koma atau parameter diulang
curl -X GET "https://site-anda.com/api/method/baseapp.api.get_stock_balance_paginated?company=PT%20Maju%20Jaya&warehouse=Toko%20Cikarang%20-%20PTMJ&from_date=2026-09-01&to_date=2026-09-30&item_code=Air%20Mineral%20600ml,Air%20Mineral%201500ml" \
  -H "Authorization: Bearer <access_token>"

# (b) per Item Group "Minuman" — otomatis termasuk sub-group di bawahnya
curl -X GET "https://site-anda.com/api/method/baseapp.api.get_stock_balance_paginated?company=PT%20Maju%20Jaya&warehouse=Toko%20Cikarang%20-%20PTMJ&from_date=2026-09-01&to_date=2026-09-30&item_group=Minuman&order_by=bal_val&order=desc" \
  -H "Authorization: Bearer <access_token>"
```

> Lapisan kedua penyaringan tetap berlaku: `is_stock_item=1`, `disabled=0`, dan hanya item yang
> punya baris di `Bin` **warehouse yang dikirim** — jadi group besar tidak otomatis melahirkan
> baris untuk gudang lain atau item non-stok.
> Catatan: `item_group` menunjuk **leaf** group; untuk group induk, kirim group induk **atau**
> resolve leaf-nya di frontend (pola [`prd_item.md` §6.7](./prd_item.md)) — server sudah menghitung
> seluruh turunannya bila group induk yang dikirim.

### 4.4 Paginasi & metadata

```bash
# halaman 3, 50 baris per halaman, urut nilai saldo terbesar
curl -X GET "https://site-anda.com/api/method/baseapp.api.get_stock_balance_paginated?company=PT%20Maju%20Jaya&warehouse=Toko%20Cikarang%20-%20PTMJ&from_date=2026-09-01&to_date=2026-09-30&order_by=bal_val&order=desc&page=3&page_length=50" \
  -H "Authorization: Bearer <access_token>"
```

```json
{
  "message": {
    "page": 3,
    "page_length": 50,
    "total_count": 128,
    "total_pages": 3,
    "has_next_page": false,
    "data": [ /* 28 baris terakhir */ ]
  }
}
```

> - `total_pages = ceil(total_count / page_length)`; `has_next_page = page < total_pages`.
> - `page` melebihi `total_pages` → `data: []` dengan `total_count`/`total_pages` tetap terisi
>   (**bukan** error) — UI boleh menampilkan "tidak ada data".
> - Bila `total_count = 0` (tidak ada item yang cocok / tidak ada `Bin` di gudang tsb) →
>   `total_pages = 0`, `has_next_page = false`, `data: []`.
> - `page_length` **maksimum 500** (di atas itu → `417`). Untuk menampilkan seluruh baris, UI
>   melakukan perulangan halaman atau menampilkan ringkasan lewat endpoint b.

### 4.5 Total keseluruhan (tanpa paginasi) — endpoint b

```bash
curl -X GET "https://site-anda.com/api/method/baseapp.api.get_stock_balance_totals?company=PT%20Maju%20Jaya&warehouse=Toko%20Cikarang%20-%20PTMJ&from_date=2026-09-01&to_date=2026-09-30&price_list=Harga%20Jual%20Retail" \
  -H "Authorization: Bearer <access_token>"
```

**Contoh respons sukses (HTTP 200)** (sesuai data §4.2: 110 + 24 Nos;

`bal_val` 495.000 + 144.000; `selling_value` 660.000 + 180.000):

```json
{
  "message": {
    "company": "PT Maju Jaya",
    "warehouse": "Toko Cikarang - PTMJ",
    "from_date": "2026-09-01",
    "to_date": "2026-09-30",
    "currency": "IDR",
    "price_list": "Harga Jual Retail",
    "price_list_currency": "IDR",
    "total_count": 2,
    "total_bal_qty": 134.0,
    "total_bal_val": 639000.0,
    "total_selling_price": 840000.0
  }
}
```

> - Endpoint b **mengabaikan** `page`/`page_length`/`order_by`/`order`; angka total selalu mencakup
>   seluruh baris sesuai filter (`company`, `warehouse`, `from_date`, `to_date`, `item_code`,
>   `item_group`, `price_list`).
> - `price_list` tidak dikirim → `total_selling_price: null` (dan `price_list: null`).
> - `total_bal_qty` = Σ `bal_qty` (perhatikan §2.4 no. 6 — hanya bermakna bila satu `stock_uom`).
> - `total_count` = jumlah baris yang disisir (membantu UI menampilkan "dari N item").

### 4.6 Membaca & memverifikasi angka

Untuk setiap baris berlaku identitas berikut (konsisten dengan report bawaan):

```text
bal_qty = opening_qty + in_qty − out_qty
bal_val = opening_val + in_val − out_val
```

- `opening_*` = hasil seluruh SLE **sebelum** `from_date` (dari Stock Closing Balance bila ada).
- `in_*` = penjumlahan perubahan positif pada periode; `out_*` = penjumlahan **nilai absolut**
  perubahan negatif pada periode.
- `val_rate` = `valuation_rate` SLE **terakhir** pada/ sebelum `to_date` (0 bila belum ada
  transaksi) — bandingkan dengan `bal_val ÷ bal_qty` untuk sanity check (selisih kecil normal
  karena pembulatan).
- Untuk menelusuri satu item secara transaksi-per-transaksi, gunakan kartu stok di
  [`prd_stock_ledger.md` §4.2](./prd_stock_ledger.md).

---

## 5. Implementasi di app `baseapp`

> Bagian ini **tidak** untuk tim UI — hanya acuan pengembang backend app `baseapp`.

### 5.1 Berkas & struktur

```text
apps/baseapp/baseapp/
└── api/
    ├── __init__.py          # re-export 2 fungsi (§5.2)
    └── stock_balance.py     # implementasi 2 endpoint
```

`baseapp/baseapp/api/__init__.py`:

```python
from baseapp.api.stock_balance import get_stock_balance_paginated, get_stock_balance_totals

__all__ = ["get_stock_balance_paginated", "get_stock_balance_totals"]
```

> Pola re-export ini menjaga **path publik tetap pendek** (`baseapp.api.<method>`), sehingga bila
> nanti ada endpoint lain (mis. `stock_ledger.py`, `sales.py`) cukup ditambah berkas baru di folder
> `api/` **tanpa** mengubah URL yang sudah dipakai frontend. `@frappe.whitelist()` menempel pada
> objek fungsi, sehingga re-export tetap dikenali `/api/method/...` — **wajib diverifikasi** setelah
> deploy dengan memanggil kedua endpoint (bila gagal, gejalanya `404 Not found` /
> `Function ... is not whitelisted`).

### 5.2 Kontrak fungsi

```python
import frappe

@frappe.whitelist()
def get_stock_balance_paginated(**kwargs): ...

@frappe.whitelist()
def get_stock_balance_totals(**kwargs): ...
```

- Kedua fungsi menerima parameter sebagai `kwargs` (dari query string **atau** body JSON), lalu
  **memvalidasi & menormalkan** ke satu dict filter internal (§5.3) — baik endpoint a maupun b
  memakai **fungsi inti yang sama**, hanya berbeda pada tahap akhir (slice halaman + metadata vs
  agregasi total). Ini menjamin angka di kedua endpoint **selalu identik**.
- Kedua fungsi **tidak boleh** dipakai lintas user (tidak ada argumen `user`/`impersonate`).

### 5.3 Alur inti (dipakai bersama)

1. **Validasi & normalisasi parameter** (§2.1) → dict `filters`:
   - `company` wajib & harus ada (`frappe.db.exists("Company", ...)`); `warehouse` wajib, tepat 1,
     ada, `is_group = 0`, dan `company` cocok;
   - `from_date`/`to_date` default `frappe.utils.today()`; `from_date ≤ to_date`;
   - `item_code`: terima string koma **dan** parameter diulang (pakai
     `frappe.form_dict.getlist("item_code")` bila perlu), buang duplikat;
   - `item_group`: multi nilai → **resolve turunan** (`get_descendants_of("Item Group", ...)` atau
     bandingkan `lft`/`rgt`) — sama seperti `apply_items_filters` pada report bawaan;
   - `price_list`, `page`, `page_length` (1..500), `order_by` (whitelist), `order` (`asc`/`desc`);
   - setiap pelanggaran → `frappe.throw(..., frappe.ValidationError)` (HTTP `417`, §6).
2. **Permission** (§5.4).
3. **Ambil baris dasar** — **reuse** report bawaan:
   ```python
   from erpnext.stock.report.stock_balance.stock_balance import StockBalanceReport

   report = StockBalanceReport(frappe._dict({
       "company": company, "from_date": from_date, "to_date": to_date,
       "warehouse": [warehouse], "item_code": item_codes, "item_group": item_group,
       "include_zero_stock_items": 1,   # baris saldo 0 tetap tampil (§2.4 no. 3)
   }))
   columns, rows = report.run()
   ```
   Memakai class report apa adanya membuat hasil **identik** dengan Desk (termasuk aturan
   Stock Reconciliation, Stock Closing Balance, `is_cancelled=0`, `docstatus<2`) — jauh lebih aman
   daripada menulis ulang query. *(Bila versi ERPNext site tidak mengekspos class ini secara stabil,
   alternatifnya menyalin logika query-nya ke `stock_balance.py`; risiko duplikasi logika dicatat
   di sini.)*
   > ⚠️ **Wajib `frappe._dict`**, bukan `dict` biasa: class report mengakses sebagian filter sebagai
   > atribut (`self.filters.item_code`, `self.filters.ignore_closing_balance`,
   > `self.filters.valuation_field_type`) → `dict` polos akan `AttributeError`.
   > Karena API ini **tidak** mengirim *Inventory Dimension* dan tidak menyalakan
   > `show_dimension_wise_stock`, kunci pengelompokan tetap `(item_code, warehouse)` → **1 baris =
   > 1 item** (dimensi stok tidak memecah baris).
4. **Tambahkan baris yang hanya ada di `Bin`** — ambil `item_code` dari `tabBin`
   (`warehouse = <param>`, item lolos filter `is_stock_item=1`, `disabled=0`) yang **belum** ada di
   hasil langkah 3, lalu sisipkan baris dengan seluruh `opening_*`/`in_*`/`out_*`/`bal_*` = `0` dan
   `val_rate` dari `Bin.valuation_rate`. Dengan begitu janji "saldo 0 tetap tampil" terpenuhi walau
   tidak ada SLE pada/ sebelum `to_date`.
5. **Sisipkan Selling Price** (§5.5) bila `price_list` dikirim.
6. **Rapikan & urutkan**: ambil field §2.3 saja, urutkan sesuai `order_by`+`order` dengan
   *tiebreaker* `item_code asc` (stabil → paginasi konsisten), lalu:
   - **endpoint a**: `total_count = len(rows)`; slice `rows[(page-1)*page_length : page*page_length]`;
     susun metadata (`page`, `page_length`, `total_pages`, `has_next_page`);
   - **endpoint b**: jumlahkan `bal_qty`, `bal_val`, `selling_value` (baris `null` → 0;
     `total_selling_price = None` bila `price_list` kosong).
7. **Kembalikan** dict (Frappe membungkusnya sebagai `"message"`).

### 5.4 Permission (wajib, karena custom method)

Custom whitelisted method **tidak** melewati pemeriksaan DocPerm. API harus:

1. Menolak bila user tidak berhak membaca data stok:
   `frappe.has_permission("Stock Ledger Entry", "read")` → gagal = `403`. Role baca v16:
   `Stock User` atau `Accounts Manager` (lihat
   [`prd_stock_ledger.md` §2.1 no. 5](./prd_stock_ledger.md)).
2. Menghormati **User Permission** (mis. `allow = Warehouse`, lihat
   [`prd_warehouse.md` §4.1](../setup/prd_warehouse.md)): bila user dibatasi ke warehouse tertentu,
   permintaan atas warehouse di luar izinnya → `403`. Cara paling sederhana: panggil
   `frappe.has_permission("Warehouse", "read", doc=warehouse)` (dan/atau periksa
   `frappe.permissions.get_user_permissions(user).Warehouse`) sebelum query dijalankan.
3. **Tidak** memakai sumber daya `ignore_permissions=True` pada query milik API ini (kecuali untuk
   data referensi non-sensitif seperti `Item Group` tree — sejalan dengan report bawaan).

### 5.5 Resolusi Selling Price (implementasi §2.5)

```python
from erpnext.stock.get_item_details import get_item_price

def resolve_selling_price(item_code, stock_uom, price_list, to_date):
    pctx = {"price_list": price_list, "uom": stock_uom, "transaction_date": to_date}

    # A. Item Price pada UOM stok (atau UOM kosong), tanpa customer/supplier
    if rows := get_item_price(pctx, item_code):
        return flt(rows[0].price_list_rate)

    # B. fallback: Item Price pada UOM alternatif → konversi ke stock_uom
    for uom, conversion_factor in get_alternative_uoms(item_code, stock_uom):
        pctx["uom"] = uom
        if rows := get_item_price(pctx, item_code):
            if flt(conversion_factor) > 0:
                return flt(rows[0].price_list_rate) / flt(conversion_factor)

    return None  # C. tidak ada harga
```

- `get_alternative_uoms` membaca child **`UOM Conversion Detail`** item (`parenttype = "Item"`,
  `uom != stock_uom`), urut `idx` — pola yang sama dipakai report bawaan untuk kolom UOM alternatif.
- **Hindari N+1**: kumpulkan seluruh `item_code` hasil langkah §5.3 no. 4 lebih dulu, ambil
  `Item Price` kandidat **sekali query massal** (filter `price_list` + `item_code in [...]` +
  validitas), baru fallback per item yang belum dapat harga. Ini penting karena maksimum 500 baris
  per halaman.
- Cache **per request** (mis. `frappe.local` dict atau `functools.lru_cache` yang dibersihkan tiap
  request) bila harga item yang sama dipanggil berulang.

### 5.6 Deploy & verifikasi

1. Tambah `baseapp/baseapp/api/__init__.py` + `stock_balance.py` (tidak ada hook baru, **tidak ada**
   doctype/custom field, jadi **tidak perlu** `bench migrate`).
2. Muat ulang proses: di devcontainer cukup simpan berkas (auto-reload), atau `bench restart`.
3. Verifikasi cepat (site dev):
   `GET http://erpnext.localhost:8000/api/method/baseapp.api.get_stock_balance_paginated?company=...&warehouse=...`
   → pastikan `message.data` terisi, `total_count` benar, dan bandingkan 2–3 baris dengan report
   **Stock Balance** di Desk (company + warehouse + rentang tanggal yang sama) — angka harus sama.
4. Uji kasus tepi: tanpa `price_list` (`selling_*` = `null`), `price_list` berisi item tanpa harga
   (`null`), baris saldo 0 (`opening_*`/`bal_*` = 0), `item_group` induk (sub-group ikut muncul),
   halaman melebihi `total_pages` (`data: []`), `warehouse` group / kosong (`417`), user tanpa akses
   warehouse (`403`).

### 5.7 Keputusan desain (ringkas & alasan)

| Keputusan | Alasan |
|---|---|
| `warehouse` wajib & tepat 1 | Menjaga hasil tetap terkait gudang fisik, membatasi jumlah baris & beban query (SLE besar), dan menyederhanakan perhitungan total. |
| Reuse `StockBalanceReport` | Hasil identik dengan Desk (termasuk kasus Stock Reconciliation/Stock Closing Balance) & otomatis ikut perbaikan versi ERPNext. |
| Baris dari `Bin` (bukan SLE) | Agar item ber-saldo 0 tetap tampil sesuai permintaan; `Bin` = daftar pasangan item×warehouse yang pernah bergerak. |
| Dua endpoint terpisah (a & b) | Nilai total harus mencakup **seluruh** baris (tidak terpengaruh paginasi), sementara UI tetap butuh data per halaman. |
| `selling_price` **dan** `selling_value` | Harga satuan tetap terlihat walau `bal_qty` 0, sementara footer total tetap sama dengan Σ kolom nilai (tidak menyesatkan). |
| `page_length` maks 500 | Melindungi server; tetap cukup untuk tampilan tabel + ekspor bertahap. |

---

## 6. Penanganan error umum

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | User tidak punya hak baca data stok (`Stock User`/`Accounts Manager`) **atau** mencoba warehouse di luar User Permission-nya | `{"exc_type": "PermissionError", "message": "Not permitted"}` |
| 417 | `company` tidak dikirim / kosong / tidak ditemukan | `{"exc_type": "ValidationError", "message": "Parameter 'company' wajib diisi."}` |
| 417 | `warehouse` tidak dikirim / kosong | `{"exc_type": "ValidationError", "message": "Parameter 'warehouse' wajib diisi (tepat 1 warehouse)."}` |
| 417 | `warehouse` dikirim lebih dari satu nilai | `{"exc_type": "ValidationError", "message": "Hanya 1 warehouse yang didukung per permintaan."}` |
| 417 | `warehouse` berupa group (`is_group=1`) atau bukan milik `company` | `{"exc_type": "ValidationError", "message": "Warehouse harus warehouse leaf milik company terpilih."}` |
| 417 | `from_date` > `to_date`, atau format tanggal tidak valid (`YYYY-MM-DD`) | `{"exc_type": "ValidationError", "message": "Format tanggal tidak valid..."}` |
| 417 | `page` < 1, atau `page_length` di luar `1..500` | `{"exc_type": "ValidationError", "message": "page_length harus antara 1 dan 500."}` |
| 417 | `order_by` di luar daftar yang diizinkan, atau `order` bukan `asc`/`desc` | `{"exc_type": "ValidationError", "message": "order_by tidak dikenal..."}` |
| 417 | `price_list` tidak ditemukan | `{"exc_type": "ValidationError", "message": "Price List ... tidak ditemukan."}` |
| 417 | `item_group` tidak ditemukan | `{"exc_type": "ValidationError", "message": "Item Group ... tidak ditemukan."}` |
| 500 | Galat tak terduga di server (mis. query report gagal) | `{"exc_type": "..."}` + traceback di log |
| — | Tidak ada baris cocok | **Bukan error** → HTTP 200 dengan `data: []`, `total_count: 0`, `total_pages: 0` |
| — | `page` melebihi `total_pages` | **Bukan error** → HTTP 200 dengan `data: []` (+ metadata terisi) |

> **Catatan:**
> - Body error berbentuk `{ "exc_type": ..., "exception": ..., "message": ... }` (standar
>   `/api/method/...`). Pesan validasi berbahasa Indonesia agar mudah ditampilkan UI — daftar di
>   atas adalah kontrak, implementasi **wajib** memakai pesan yang sama.
> - Yang paling sering salah di sisi frontend: mengirim `warehouse` **group** (bukan leaf),
>   mengirim **lebih dari satu** warehouse, lupa bahwa `from_date`/`to_date` **default hari ini**
>   (bukan awal bulan), dan mengira `total_selling_price` bernilai `0` padahal `price_list` belum
>   dikirim (`null`).
> - Kinerja: dengan `company` + **satu** warehouse wajib, hasil biasanya terbatas (ratusan baris).
>   Namun rentang tanggal yang sangat lebar (mis. `from_date` 5 tahun ke belakang) dan tanpa
>   *Stock Closing Entry* membuat pemindaian SLE besar — pertimbangkan batas maksimum rentang di
>   masa depan bila ditemukan keluhan kinerja.

---

## 7. Rencana koleksi Postman

Endpoint di dokumen ini akan ditambahkan ke koleksi `docs/postman/postman_erpnext_api.json`
(Collection v2.1) sebagai folder **`18. Stock Balance`** (nomor 17 sudah dipakai *Product Bundle*),
berisi rencana request berikut:

| Request | Isi |
|---|---|
| `18.1` | Saldo stok dasar (company + warehouse + rentang tanggal, tanpa `price_list`) — §4.1 |
| `18.2` | Saldo stok + kolom Selling Price (`price_list`) — §4.2 |
| `18.3` | Filter `item_code` (beberapa item) — §4.3 (a) |
| `18.4` | Filter `item_group` (termasuk sub-group) + `order_by`/`order` — §4.3 (b) |
| `18.5` | Halaman berikutnya (`page`) + cek metadata `total_count`/`total_pages` — §4.4 |
| `18.6` | Endpoint total (`get_stock_balance_totals`, tanpa paginasi) — §4.5 |
| `18.7` | Kasus tanpa harga: `price_list` berisi item yang belum punya Item Price → `selling_*: null` — §2.5 C |
| `18.8` | Kasus validasi: `warehouse` kosong / group / lebih dari satu → `417` — §6 |

**Variabel yang perlu diisi** (Collection Variables):

- `sb_company` — company (mis. `PT Maju Jaya`)
- `sb_warehouse` — warehouse **leaf** (mis. `Toko Cikarang - PTMJ`)
- `sb_from_date` / `sb_to_date` — rentang tanggal (mis. `2026-09-01` / `2026-09-30`)
- `sb_price_list` — Price List untuk kolom Selling Price (mis. `Harga Jual Retail`)
- `sb_item_group` — Item Group untuk request `18.4` (mis. `Minuman`)
- `sb_page_length` — jumlah baris per halaman (default `50`)

> Request `18.1`–`18.8` hanya berfungsi bila app **`baseapp`** sudah terpasang di site yang
> digunakan (berbeda dari folder lain yang hanya butuh ERPNext).

> **Dokumen terkait:** **[prd_stock_entry.md](./prd_stock_entry.md)** — dokumen yang mengubah stok
> (transfer antar gudang, Material Issue/Receipt; alur 3 tahap pengajuan → pengeluaran →
> penerimaan), **[prd_stock_ledger.md](./prd_stock_ledger.md)** — kartu stok per dokumen,
> **[prd_stock_reconciliation.md](./prd_stock_reconciliation.md)** — opname/koreksi saldo.
