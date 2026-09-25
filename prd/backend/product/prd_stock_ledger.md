# PRD — REST API Doctype Stock Ledger Entry (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Stock Ledger Entry** (SLE) di ERPNext (Frappe),
> diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Stock (ERPNext) — path `erpnext.stock.doctype.stock_ledger_entry`
- **Doctype:** `Stock Ledger Entry` — **read-only**. Bukan master data dan bukan transaksi yang
  dibuat user; setiap baris SLE **ditulis otomatis oleh sistem** (bukan lewat `insert`/`save`)
- **Versi API:** `/api/method/...` (API v1) — method whitelisted `frappe.client.*`, `name` & filter
  dikirim di **body**
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

> Stock Ledger Entry adalah **bukti (jurnal) pergerakan stok** — catatan tiap perubahan qty & nilai
> per item+warehouse pada tanggal/jam posting tertentu (deskripsi doctype: *"Stock Ledger Entry is
> created whenever a stock transaction takes place"*). SLE **dibuat sistem** saat dokumen transaksi
> stok di-*submit* — mis. Purchase Receipt / Purchase Invoice, Delivery Note / Sales Invoice, Stock
> Entry, **Stock Reconciliation** (lihat §2.4) — dan **dibalik/ditandai** saat transaksi tersebut
> di-*cancel*.
>
> Karena SLE hanya *jurnal hasil posting* (tidak pernah dibuat user), perilakunya **berbeda dari
> semua PRD lain**:
> - **Tidak ada CREATE/UPDATE/DELETE/SUBMIT/CANCEL.** SLE tidak bisa dibuat, diubah, apalagi
>   dicancel satuan lewat API (backend menolak cancel baris SLE: *"Individual Stock Ledger Entry
>   cannot be cancelled."*). Satu-satunya operasi yang relevan: **READ** (`get_list`, `get`,
>   `get_value`, `get_last_doc`, `get_count`) — inilah fokus dokumen ini.
> - **Pembatalan transaksi sumber ditandai `is_cancelled`**, bukan `docstatus` seperti transaksi
>   lain. Saat dokumen stok di-cancel, ERPNext menandai SLE lama `is_cancelled=1` dan menulis SLE
>   pembalik baru (qty berlawanan) ber-`is_cancelled=0`. Untuk melihat riwayat "efektif", frontend
>   **wajib memfilter `is_cancelled = 0`** (§2.1 no. 3, §4.8).
> - Untuk **saldo stok terkini** item+warehouse, frontend **membaca** `qty_after_transaction` /
>   `stock_value` dari **SLE terakhir yang aktif** pada kartu stok — tidak menghitung manual dari
>   seluruh baris (§4.2, §4.6).
> - Role baca default (v16) lebih longgar daripada Stock Reconciliation: **`Stock User`** atau
>   **`Accounts Manager`** (§2.1 no. 5).

---

## 1. Ruang lingkup

| # | Endpoint (POST `/api/method/...`) | Operasi | `name` di |
|---|---|---|---|
| 1 | `frappe.client.get_list` | Daftar SLE transaksi stok pada rentang tanggal + company (§4.1) | body (filters) |
| 2 | `frappe.client.get_list` | **Kartu stok / mutasi** 1 item+warehouse, urut kronologis (running balance, §4.2) | body (filters) |
| 3 | `frappe.client.get_list` | Verifikasi posting — SLE milik 1 voucher (`voucher_type`+`voucher_no`, §4.3) | body (filters) |
| 4 | `frappe.client.get` | Detail 1 baris SLE (READ single, §4.4) | body |
| 5 | `frappe.client.get_value` | Ambil nilai 1/beberapa field SLE (cek cepat, §4.5) | body |
| 6 | `frappe.client.get_list` / `get_last_doc` | Baris SLE **terakhir** item+warehouse → saldo terkini (§4.6) | body |
| 7 | `frappe.client.get_count` | Total baris SLE sesuai filter — pagination (§4.7) | body (filters) |
| 8 | `frappe.client.get_list` | Dropdown pendukung filter: Item / Warehouse / Company (§5) | body (filters) |
| 9 | `frappe.client.get_list` (Item, lalu SLE) | Menyaring SLE utk produk stok per Item Group — `is_stock_item` + `item_group` (§4.9) | body (filters) |

> **Konvensi pemanggilan (penting):** seluruh operasi memakai method whitelisted **`frappe.client.*`**
> dengan `name` (dan filter) dikirim lewat **body JSON**, bukan di URL path (alasan sama dengan
> Warehouse — konsistensi body-based, lihat catatan `prd_warehouse.md`). **Format respons:** method
> `/api/method/...` membungkus hasil di **`"message"`** (bukan `"data"` seperti `/api/resource/...`).
> Untuk list, `message` berupa **array** baris SLE (kosong `[]` bila tidak ada data — bukan error).

> **Perbedaan utama dengan PRD lain (master / transaksi):**
> - Item Group / Warehouse / Item (master) & Stock Reconciliation (transaksi submittable) punya alur
>   CREATE–UPDATE–submit/cancel. SLE **tidak punya alur tulis** sama sekali.
> - `name` SLE dibuat sistem via naming series **`MAT-SLE-.YYYY.-.#####`** (mis. `MAT-SLE-00001-2026`).
>   Saat insert cepat, ERPNext sempat memakai `name` hash 10 karakter lalu me-*rename* otomatis ke
>   seri tsb di belakang layar (scheduled job) — jadi `name` SLE **tidak dipakai ulang** sebagai
>   input operasi apa pun; identifikasi baris yang stabil memakai kombinasi
>   `voucher_type`+`voucher_no`+`voucher_detail_no` (atau `item_code`+`warehouse`+`posting_datetime`).
> - Stock Reconciliation v16 hanya bisa diakses role `Stock Manager`; SLE justru bisa **dibaca** oleh
>   role yang lebih rendah (`Stock User`/`Accounts Manager`) karena SLE memang disediakan utk
>   kebutuhan lihat-riwayat stok.

---

## 2. Ringkasan field & data

Semua field SLE bersifat **`read_only`** (ditulis sistem) — tidak ada field yang "wajib dikirim".
Klasifikasi di bawah memakai makna: **🔴 = filter utama** yang sebaiknya selalu dipakai frontend,
**🟠 = filter lanjutan** (kondisional), **⚪ = hanya output** (boleh difilter, tapi umumnya cukup
dibaca).

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **Filter utama** | `is_cancelled` | Check | `0` = SLE aktif (efektif), `1` = SLE yang dibatalkan (dari transaksi sumber yang di-cancel / entry pembalik lama). **Selalu filter `= 0`** untuk riwayat efektif (§2.1 no. 3). |
| 🔴 **Filter utama** | `item_code` | Link → Item | Produk yang stoknya bergerak. Filter di sini = lihat riwayat 1 produk. |
| 🔴 **Filter utama** | `warehouse` | Link → Warehouse | Gudang tempat stok bergerak. Filter di sini = lihat riwayat 1 gudang (leaf, bukan group). |
| 🔴 **Filter utama** | `posting_date` (atau `posting_datetime`) | Date / Datetime | Tanggal (dan jam) efektif transaksi. Untuk rentang, filter `posting_date between [from, to]` atau `posting_datetime between [...]`. |
| 🟠 **Filter lanjutan** | `company` | Link → Company | Perusahaan. Berguna memisahkan data multi-company (nilai SLE dalam mata uang company ini). |
| 🟠 **Filter lanjutan** | `voucher_type` | Link → DocType | Jenis transaksi sumber (mis. `Purchase Receipt`, `Delivery Note`, `Stock Entry`, `Stock Reconciliation`). Dipakai utk verifikasi posting 1 dokumen. |
| 🟠 **Filter lanjutan** | `voucher_no` | Dynamic Link (ke `voucher_type`) | Nomor dokumen sumber. Kombinasi `voucher_type`+`voucher_no` = bukti posting 1 dokumen (§4.3). |
| 🟠 **Filter lanjutan** | `batch_no` / `serial_no` | Data / Long Text | Hanya utk item ber-*batch*/ber-*serial* (lihat juga `serial_and_batch_bundle`). |
| ⚪ **Output** | `name` | Data | Dibuat sistem (`MAT-SLE-.YYYY.-.#####`). Tidak dipakai ulang sebagai input (§1). |
| ⚪ **Output** | `actual_qty` | Float | **Perubahan qty** transaksi ini (+ = masuk, − = keluar). Label desk: *Qty Change*. |
| ⚪ **Output** | `qty_after_transaction` | Float | **Qty berjalan** item+warehouse **setelah** transaksi ini — inilah "saldo" yang tampil di kartu stok. |
| ⚪ **Output** | `incoming_rate` / `outgoing_rate` | Currency | Rate masuk/keluar dari transaksi ini (sebelum penyesuaian). |
| ⚪ **Output** | `valuation_rate` | Currency | Rate rata-rata stok berjalan setelah transaksi (label desk: *Average Rate*). |
| ⚪ **Output** | `stock_value` | Currency | **Nilai stok berjalan** (qty_after_transaction × valuation_rate) setelah transaksi. Label desk: *Balance Stock Value*. |
| ⚪ **Output** | `stock_value_difference` | Currency | **Perubahan nilai** yang diposting transaksi ini. |
| ⚪ **Output** | `stock_uom` | Link → UOM | Satuan stok item. |
| ⚪ **Output** | `posting_datetime` | Datetime | Gabungan `posting_date`+`posting_time` — urutan kronologis sebenarnya. |
| ⚪ **Output** | `voucher_detail_no` | Data | `name` baris pada dokumen sumber (mis. nama baris item di Stock Reconciliation / Stock Entry) — pencocokan granular. |
| ⚪ **Output** | `creation` / `modified` | Datetime | Stempel waktu pembuatan baris — dipakai sebagai *tiebreaker* urutan bila `posting_datetime` kembar. |
| ⚪ **Output (serial/batch)** | `serial_and_batch_bundle`, `serial_no`, `batch_no`, `has_serial_no`, `has_batch_no` | — | Untuk item serial/batch v16, rincian serial tersimpan di **Serial and Batch Bundle** (`serial_and_batch_bundle`) — SLE lama menyimpan teks `serial_no`. |
| ⚪ **Output (internal)** | `fiscal_year`, `project`, `is_adjustment_entry`, `recalculate_rate`, `to_rename`, `stock_queue`, `dependant_sle_voucher_detail_no` | — | Umumnya tidak perlu ditampilkan. `is_adjustment_entry=1` menandai SLE teknis (penyesuaian) yang bukan transaksi user. |
| 🟠 **Filter (via master Item)** | `item_group` | (field master `Item`) | Grup produk utk menyaring SLE. **Bukan kolom SLE** — SLE tidak menyimpan group; pemakaiannya: *resolve* `item_code` dari master Item lalu `["item_code","in",[...]]` saat baca SLE (§4.9). |
| 🟠 **Filter (via master Item)** | `is_stock_item` | (field master `Item`) | Hanya produk stok (`1`). **Bukan kolom SLE** — dipakai utk membatasi *resolve* Item / dropdown Item (hanya produk stok yang punya SLE) sebelum baca SLE (§4.9, §5.1). |

> Catatan: daftar di atas menonjolkan field yang berguna untuk **pengambilan data** (filter + output
> tampilan), bukan seluruh field doctype. Karena SLE read-only, tidak ada aturan "wajib kirim" dari
> sisi frontend — yang wajib adalah **memilih `fields` yang diinginkan** pada `get_list` (hindari
> meminta field yang tidak ada; lihat §6). Dua baris terakhir (`item_group`, `is_stock_item`) adalah
> **field master `Item`** yang dipakai sebagai *filter tidak langsung* terhadap SLE — SLE sendiri
> tidak menyimpan kedua kolom tsb, sehingga pemakaiannya lewat **2 langkah**: resolve `item_code`
> dari Item, lalu baca SLE dengan `["item_code","in",[...]]` (contoh lengkap §4.9).

### 2.1 Catatan penting

1. **SLE read-only — jangan coba tulis.** SLE dibuat otomatis oleh transaksi stok. `insert`/`save`
   manual pada doctype ini bukan alur yang sah untuk aplikasi (dokumen hanya boleh dibuat lewat
   alur transaksi yang sudah didokumentasikan di PRD masing-masing). `frappe.client.cancel` pada SLE
   juga ditolak backend.
2. **Bukan `docstatus` 0/1/2 — tapi `is_cancelled`.** SLE selalu ter-*posting* (tidak ada status
   Draft). Pembatalan ditandai field `is_cancelled` (0 aktif / 1 dibatalkan) + SLE pembalik baru.
   Jangan memfilter `docstatus` pada SLE seperti pada transaksi biasa.
3. **Selalu filter `is_cancelled = 0` untuk data efektif.** Bila transaksi sumber di-cancel (atau
   ada reposting), SLE lama ditandai `is_cancelled = 1`. Tanpa filter ini, kartu stok akan
   menampilkan baris yang sudah "dibatalkan" dan saldo jadi salah. Untuk audit *semua* baris
   (termasuk yang dibatalkan), hilangkan filter ini.
4. **`qty_after_transaction`/`stock_value` = saldo berjalan.** Nilainya **per kombinasi
   item+warehouse** (dan per batch/serial bila ada) pada urutan posting, bukan saldo global antar
   warehouse. Untuk item non serial/batch, baris terakhir (urut `posting_datetime`) = saldo terkini
   item di warehouse itu.
5. **Role baca (v16): `Stock User` atau `Accounts Manager`.** Permission doctype Stock Ledger Entry
   memberikan hak **read/export/report** ke role `Stock User` dan `Accounts Manager` — lebih rendah
   dari `Stock Manager` (yang dibutuhkan Stock Reconciliation). Jadi user operasional gudang yang
   hanya ber-role `Stock User` pun bisa membaca SLE. Semua operasi di dokumen ini cukup butuh
   **read**.
6. **Data bisa sangat besar — selalu batasi.** `tabStock Ledger Entry` adalah tabel yang tumbuh
   cepat (1+ baris per baris transaksi stok). Selalu sertakan filter item/warehouse/rentang tanggal
   dan pakai pagination (`limit_start`/`limit_page_length`), jangan `limit_page_length = 0` tanpa
   filter ketat.

### 2.2 Arti kolom qty & nilai (cara membaca 1 baris SLE)

Satu baris SLE menjawab: *"pada `posting_datetime`, berapa qty & nilai item di warehouse berubah,
dan berapa saldonya setelahnya?"*

- `actual_qty` = perubahan (mis. Delivery Note keluar 10 → `-10`; Purchase Receipt masuk 10 →
  `+10`; Stock Reconciliation naikkan fisik 20 → `+20`).
- `qty_after_transaction` = saldo qty setelah baris ini (angka berjalan — ini yang biasanya
  ditampilkan UI sebagai kolom "Balance Qty").
- `incoming_rate`/`outgoing_rate` = rate yang dipakai transaksi ini; `valuation_rate` = rata-rata
  berjalan setelahnya.
- `stock_value_difference` = nilai yang diposting baris ini; `stock_value` = nilai saldo berjalan
  setelahnya.
- Konsistensi (bila dipakai utk verifikasi): `qty_after_transaction` baris ke-n = `qty_after_transaction`
  baris ke-(n−1) + `actual_qty` baris ke-n (untuk kombinasi item+warehouse yang sama, diurut
  `posting_datetime`+`creation`).

### 2.3 Mengapa ada SLE "ganda" saat transaksi di-cancel

Saat dokumen stok di-cancel, ERPNext **tidak menghapus** SLE lama. Yang terjadi:
- SLE lama di-set `is_cancelled = 1` (tetap tersimpan utk audit).
- Dibuat SLE **pembalik** baru dengan `actual_qty` berlawanan, `docstatus=1`, `is_cancelled = 0`,
  `voucher_no` tetap dokumen yang sama — inilah yang membuat saldo kembali ke kondisi sebelum
  transaksi.
- Bila ada transaksi stok lain setelahnya, sistem memicu **reposting** (Repost Item Valuation) agar
  SLE sesudahnya ikut dihitung ulang.

> Implikasi UI: daftar/kartu stok yang menampilkan riwayat "efektif" cukup memakai filter
> `is_cancelled = 0`. Untuk menampilkan *audit trail* (apa yang dibatalkan), tampilkan juga baris
> `is_cancelled = 1` dan/atau beri label.

### 2.4 Tabel-tabel terkait (dari sisi Stock Ledger Entry)

| Tabel (`tab...`) | Doctype | Peran |
|---|---|---|
| `tabStock Ledger Entry` | `Stock Ledger Entry` | **Jurnal pergerakan stok** (dokumen ini). Satu baris = satu perubahan qty/nilai item+warehouse dari 1 voucher. `name` via seri `MAT-SLE-.YYYY.-.#####`; kolom kunci: `item_code`, `warehouse`, `posting_datetime`, `actual_qty`, `qty_after_transaction`, `valuation_rate`, `stock_value`, `voucher_type`, `voucher_no`, `is_cancelled`. |
| `tabBin` | `Bin` | **Saldo stok ringkas per item+warehouse** (`actual_qty`, `stock_value`, `valuation_rate`, `reserved_qty`) — diperbarui mengikuti SLE. Berguna utk *current stock* cepat (bukan riwayat). Pengambilan datanya di luar cakupan dokumen ini (dokumentasi stok/PRD tersendiri). |
| `tabSerial and Batch Bundle` (+ `tabSerial and Batch Bundle Item`) | `Serial and Batch Bundle` | **Rincian serial/batch** utk SLE item serial/batch v16 (`serial_and_batch_bundle`). Bila UI perlu daftar serial per baris SLE, baca bundle ini. |
| `tabRepost Item Valuation` | `Repost Item Valuation` | Dicatat saat SLE di-*repost* (transaksi backdated / cancel) — indikator bahwa sebagian SLE dihitung ulang di background. Status reposting dapat memengaruhi konsistensi angka saat dibaca. |
| Voucher sumber (mis. `tabPurchase Receipt`, `tabDelivery Note`, `tabStock Entry`, `tabStock Reconciliation`, dll.) | Purchase Receipt / Delivery Note / Stock Entry / Stock Reconciliation / ... | **Dokumen yang memproduksi SLE**. `voucher_type`+`voucher_no` di SLE menunjuk ke sini. Detail masing-masing ada di PRD terkait (mis. Stock Reconciliation: `[prd_stock_reconciliation.md §2.4](./prd_stock_reconciliation.md)`). |

> Karena SLE hanya bisa dibaca, seluruh "penulisan" ke tabel di atas terjadi lewat submit/cancel
> transaksi sumber — bukan lewat API dokumen ini.

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Detail lengkap setup OAuth Client ada di **[`prd_oauth.md`](../prd_oauth.md)**.

---

## 4. Pengambilan data (READ)

> **Alur umum untuk UI:**
> 1. Baca riwayat/mutasi dengan `frappe.client.get_list` + filter ketat (`is_cancelled=0`,
>    item/warehouse/rentang tanggal) → tampilkan tabel/kartu stok (§4.1–§4.2).
> 2. Bila perlu bukti posting satu dokumen → `get_list` dengan filter `voucher_type`+`voucher_no`
>    (§4.3).
> 3. Bila perlu detail/satu baris → `get` (§4.4); cek cepat nilai → `get_value` (§4.5); saldo
>    terkini → baris terakhir (§4.6).
> 4. Pagination memakai `get_count` + `limit_start` (§4.7).
>
> Semua filter bersifat opsional — kombinasi bebas. `message` berisi **array** (list) atau **objek**
> (single); hasil kosong pada list = `[]` (bukan error).

### 4.1 Daftar SLE — transaksi stok pada rentang tanggal (READ list)

Skenario paling umum: tampilkan seluruh pergerakan stok di satu company dalam satu periode
(mis. halaman "Riwayat Stok" / laporan transaksi). Selalu sertakan `is_cancelled = 0`.

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Ledger Entry",
    "fields": [
      "name", "item_code", "warehouse", "posting_date", "posting_time",
      "actual_qty", "qty_after_transaction", "valuation_rate", "stock_value",
      "stock_value_difference", "voucher_type", "voucher_no", "is_cancelled"
    ],
    "filters": [
      ["is_cancelled", "=", 0],
      ["company", "=", "PT Maju Jaya"],
      ["posting_date", "between", ["2026-09-01", "2026-09-30"]]
    ],
    "order_by": "posting_date desc, posting_time desc, creation desc",
    "limit_start": 0,
    "limit_page_length": 50
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": [
    {
      "name": "MAT-SLE-00042-2026",
      "item_code": "Air Mineral 600ml",
      "warehouse": "Toko Cikarang - PTMJ",
      "posting_date": "2026-09-04",
      "posting_time": "10:30:00",
      "actual_qty": -10,
      "qty_after_transaction": 90,
      "valuation_rate": 4500,
      "stock_value": 405000,
      "stock_value_difference": -45000,
      "voucher_type": "Delivery Note",
      "voucher_no": "MAT-DN-00021-2026",
      "is_cancelled": 0
    }
  ]
}
```

> - Halaman berikutnya naikkan `limit_start` kelipatan `limit_page_length`; `limit_page_length = 0` =
>   ambil **semua** (hindari tanpa filter ketat — §2.1 no. 6).
> - `posting_date between [from, to]` memakai string tanggal; untuk presisi jam gunakan
>   `posting_datetime between ["2026-09-01 00:00:00", "2026-09-04 23:59:59"]`.
> - Field yang diminta di `fields` menentukan isi tiap objek — minta hanya yang akan ditampilkan.

### 4.2 Kartu stok / mutasi 1 item+warehouse (running balance)

Untuk halaman "Kartu Stok" (mutasi satu produk di satu gudang), urutkan **naik** berdasarkan
`posting_datetime` (lalu `creation` sebagai tiebreaker) agar `qty_after_transaction` membentuk
saldo berjalan yang benar:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Ledger Entry",
    "fields": ["name","posting_date","posting_time","actual_qty","qty_after_transaction","incoming_rate","outgoing_rate","valuation_rate","stock_value","stock_value_difference","voucher_type","voucher_no","voucher_detail_no"],
    "filters": [
      ["item_code", "=", "Air Mineral 600ml"],
      ["warehouse", "=", "Toko Cikarang - PTMJ"],
      ["is_cancelled", "=", 0],
      ["posting_date", "<=", "2026-09-04"]
    ],
    "order_by": "posting_datetime asc, creation asc",
    "limit_page_length": 100
  }'
```

**Contoh respons sukses (HTTP 200):** `message` = array baris berurutan; perhatikan kolom saldo
berjalan:

```json
{
  "message": [
    { "name": "MAT-SLE-00011-2026", "posting_date": "2026-09-02", "posting_time": "08:00:00", "actual_qty": 100, "qty_after_transaction": 100, "valuation_rate": 4500, "stock_value": 450000, "voucher_type": "Purchase Receipt", "voucher_no": "MAT-PRE-00015-2026" },
    { "name": "MAT-SLE-00031-2026", "posting_date": "2026-09-04", "posting_time": "09:15:00", "actual_qty": 20,  "qty_after_transaction": 120, "valuation_rate": 4500, "stock_value": 540000, "voucher_type": "Stock Reconciliation", "voucher_no": "MAT-RECO-00001-2026" },
    { "name": "MAT-SLE-00042-2026", "posting_date": "2026-09-04", "posting_time": "10:30:00", "actual_qty": -10, "qty_after_transaction": 110, "valuation_rate": 4500, "stock_value": 495000, "voucher_type": "Delivery Note", "voucher_no": "MAT-DN-00021-2026" }
  ]
}
```

> - `qty_after_transaction` baris terakhir = **saldo item di warehouse tersebut** hingga tanggal tsb.
> - Bila item memakai batch/serial, filter juga `batch_no`/`serial_no` (atau `serial_and_batch_bundle`)
>   — karena saldo berjalan dihitung per batch/serial. Untuk item non serial/batch, cukup kombinasi
>   item+warehouse.

### 4.3 Verifikasi posting — SLE milik 1 voucher (voucher_type + voucher_no)

Saat UI menampilkan "bukti posting" sebuah dokumen transaksi stok (mis. tombol lihat Stock Ledger
dari Stock Reconciliation / Stock Entry), baca semua SLE dengan `voucher_type` = doctype dokumen dan
`voucher_no` = `name` dokumen:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Ledger Entry",
    "fields": ["name","item_code","warehouse","posting_date","posting_time","actual_qty","qty_after_transaction","valuation_rate","stock_value_difference","voucher_detail_no"],
    "filters": [
      ["voucher_type", "=", "Stock Reconciliation"],
      ["voucher_no", "=", "MAT-RECO-00001-2026"],
      ["is_cancelled", "=", 0]
    ],
    "order_by": "creation asc",
    "limit_page_length": 0
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": [
    { "name": "MAT-SLE-00031-2026", "item_code": "Air Mineral 600ml", "warehouse": "Toko Cikarang - PTMJ", "posting_date": "2026-09-04", "posting_time": "09:15:00", "actual_qty": 20, "qty_after_transaction": 120, "valuation_rate": 4500, "stock_value_difference": 90000, "voucher_detail_no": "abc123" }
  ]
}
```

> Pola yang sama berlaku utk voucher stok lain — isi `voucher_type` (mis. `Purchase Receipt`,
> `Delivery Note`, `Sales Invoice`, `Stock Entry`) dengan nama doctype dokumen, dan `voucher_no`
> dengan `name` dokumen tsb. `voucher_detail_no` mengaitkan baris SLE ke baris item di dokumen
> sumber. Bila dokumen sudah di-cancel, SLE pembaliknya muncul dengan `actual_qty` berlawanan dan
> `is_cancelled = 0`; SLE lama ber-`is_cancelled = 1` (lihat §4.8).

### 4.4 Detail 1 baris SLE (READ single) — `frappe.client.get`

Untuk melihat seluruh field satu baris SLE (mis. saat user membuka detail sebuah entri di kartu
stok), gunakan `get` dengan `name` baris:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Ledger Entry",
    "name": "MAT-SLE-00031-2026"
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": {
    "name": "MAT-SLE-00031-2026",
    "owner": "Administrator",
    "creation": "2026-09-04 09:15:00.000000",
    "docstatus": 1,
    "item_code": "Air Mineral 600ml",
    "warehouse": "Toko Cikarang - PTMJ",
    "posting_date": "2026-09-04",
    "posting_time": "09:15:00",
    "posting_datetime": "2026-09-04 09:15:00",
    "actual_qty": 20,
    "qty_after_transaction": 120,
    "incoming_rate": 4500,
    "valuation_rate": 4500,
    "stock_value": 540000,
    "stock_value_difference": 90000,
    "stock_uom": "Nos",
    "voucher_type": "Stock Reconciliation",
    "voucher_no": "MAT-RECO-00001-2026",
    "voucher_detail_no": "abc123",
    "company": "PT Maju Jaya",
    "fiscal_year": "2026-2027",
    "is_cancelled": 0
  }
}
```

> `docstatus` di SLE selalu `1` (sudah ter-posting); jangan salah tafsir sebagai status
> Draft/Submitted seperti transaksi biasa — pembatalan ditandai `is_cancelled`, bukan `docstatus`
> (§2.1 no. 2).

### 4.5 Ambil nilai 1/beberapa field — `frappe.client.get_value` (cek cepat)

Untuk cek cepat (mis. hanya perlu `qty_after_transaction` & `valuation_rate` baris terakhir suatu
item+warehouse tanpa mengambil seluruh baris):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Ledger Entry",
    "filters": [
      ["item_code", "=", "Air Mineral 600ml"],
      ["warehouse", "=", "Toko Cikarang - PTMJ"],
      ["is_cancelled", "=", 0]
    ],
    "fieldname": ["qty_after_transaction", "valuation_rate", "posting_datetime"]
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": { "qty_after_transaction": 110, "valuation_rate": 4500, "posting_datetime": "2026-09-04 10:30:00" }
}
```

> - `filters` memakai baris yang cocok pertama (default urutan `creation desc`) — utk mendapatkan
>   **nilai terakhir** secara deterministik lebih baik pakai pola §4.6 (`get_list` + `order_by`
>   `posting_datetime desc` + `limit_page_length 1`).
> - Bila filter tidak cocok, respons `message` berupa objek dengan field `None` (bukan error).

### 4.6 Baris terakhir item+warehouse → saldo terkini

Cara paling andal mengambil **saldo stok terkini** sebuah item di sebuah warehouse adalah membaca
baris SLE **terakhir** (urut `posting_datetime` + `creation` menurun), karena baris tsb membawa
`qty_after_transaction` & `stock_value` final:

```bash
# Opsi A — get_list + order_by desc + limit 1 (direkomendasikan)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Ledger Entry",
    "fields": ["posting_datetime","actual_qty","qty_after_transaction","valuation_rate","stock_value"],
    "filters": [
      ["item_code", "=", "Air Mineral 600ml"],
      ["warehouse", "=", "Toko Cikarang - PTMJ"],
      ["is_cancelled", "=", 0]
    ],
    "order_by": "posting_datetime desc, creation desc",
    "limit_page_length": 1
  }'
```

```bash
# Opsi B — get_last_doc (mengembalikan dokumen utuh baris terakhir sesuai filter)
curl -X POST https://site-anda.com/api/method/frappe.client.get_last_doc \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Ledger Entry",
    "filters": [
      ["item_code", "=", "Air Mineral 600ml"],
      ["warehouse", "=", "Toko Cikarang - PTMJ"],
      ["is_cancelled", "=", 0]
    ]
  }'
```

**Contoh respons (HTTP 200):** Opsi A → `message` array berisi 1 objek; Opsi B → `message` objek
dokumen. Keduanya memuat nilai yang sama:

```json
{ "posting_datetime": "2026-09-04 10:30:00", "actual_qty": -10, "qty_after_transaction": 110, "valuation_rate": 4500, "stock_value": 495000 }
```

> - Filter `is_cancelled = 0` penting agar tidak "menangkap" baris yang sudah dibatalkan.
> - `get_last_doc` memilih baris terakhir berdasarkan `creation`; Opsi A lebih disarankan karena
>   mengurutkan berdasarkan `posting_datetime` (urutan bisnis stok).
> - Bila item tidak pernah punya SLE (belum ada transaksi) → `message` kosong/`None`; UI dapat
>   menampilkan saldo 0.

### 4.7 Pagination & total count — `frappe.client.get_count`

Untuk menghitung total baris SLE yang cocok dengan filter (lazy loading / indikator jumlah):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_count \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Ledger Entry",
    "filters": [
      ["is_cancelled", "=", 0],
      ["company", "=", "PT Maju Jaya"],
      ["posting_date", "between", ["2026-09-01", "2026-09-30"]]
    ]
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": 1240
}
```

> Gabungkan dengan `get_list` ber-`limit_start`/`limit_page_length` (§4.1) untuk halaman berikutnya.
> Ingat: jumlah baris SLE bisa besar — batasi scope filter.

### 4.8 Membaca efek cancel / repost (is_cancelled & entry pembalik)

Ketika sebuah transaksi stok di-cancel, riwayat SLE berubah. Agar UI tidak salah menampilkan:

- **Lihat riwayat efektif** → filter `is_cancelled = 0` (seluruh contoh §4.1–§4.7 sudah memakai ini).
- **Lihat audit trail** → tampilkan juga baris `is_cancelled = 1` (beri label "Dibatalkan"). SLE
  pembalik (aktif) memakai `voucher_type`+`voucher_no` yang sama dengan transaksi yang dibatalkan,
  dengan `actual_qty` berlawanan.

```bash
# Contoh: semua SLE (aktif + yang dibatalkan) milik satu dokumen yang sudah di-cancel
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Ledger Entry",
    "fields": ["name","item_code","warehouse","posting_datetime","actual_qty","qty_after_transaction","stock_value_difference","is_cancelled","creation"],
    "filters": [
      ["voucher_type", "=", "Stock Reconciliation"],
      ["voucher_no", "=", "MAT-RECO-00001-2026"]
    ],
    "order_by": "creation asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "name": "MAT-SLE-00031-2026", "posting_datetime": "2026-09-04 09:15:00", "actual_qty": 20,  "qty_after_transaction": 120, "stock_value_difference": 90000, "is_cancelled": 1 },
    { "name": "MAT-SLE-00058-2026", "posting_datetime": "2026-09-05 08:00:00", "actual_qty": -20, "qty_after_transaction": 100, "stock_value_difference": -90000, "is_cancelled": 0 }
  ]
}
```

> Baris pertama (`is_cancelled=1`) = SLE asli yang dibatalkan; baris kedua = SLE pembalik. Bila
> setelah cancel ada transaksi stok lain, sistem menjalankan **reposting** (Repost Item Valuation,
> umumnya background) — beberapa SLE sesudahnya bisa ikut berubah nilainya. UI boleh menampilkan
> status reposting dan me-refresh setelah selesai bila diperlukan.

### 4.9 Menyaring SLE berdasarkan Item Group / `is_stock_item` (via master Item)

SLE **tidak punya kolom** `item_group`/`is_stock_item` — keduanya field master `Item` (§2). Untuk
membaca SLE **hanya** untuk produk stok dari grup tertentu, lakukan 2 langkah:

**Langkah 1 — resolve `item_code`** (daftar Item stok pada leaf group):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name"],
    "filters": [
      ["item_group", "=", "Minuman"],
      ["is_stock_item", "=", 1],
      ["disabled", "=", 0]
    ],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):** `message` = array objek → kumpulkan `name` sebagai daftar `item_code`:

```json
{ "message": [ { "name": "Air Mineral 600ml" }, { "name": "Air Mineral 1500ml" } ] }
```

> Bila filter memakai **group induk** (ingin menyertakan sub-group): resolve dulu seluruh leaf
> keturunannya (rentang `lft`/`rgt` pada Item Group, atau `frappe.desk.treeview.get_children`),
> lalu gunakan `["item_group","in",[...]]` pada Item — pola lengkap:
> **[`prd_item.md` §6.7](./prd_item.md)** & [`prd_item_group.md` §4/§5.1](./prd_item_group.md).

**Langkah 2 — baca SLE utk item tsb** (`item_code in [...]` + filter SLE biasa):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Ledger Entry",
    "fields": ["name","item_code","warehouse","posting_date","actual_qty","qty_after_transaction","valuation_rate","stock_value","voucher_type","voucher_no"],
    "filters": [
      ["item_code", "in", ["Air Mineral 600ml", "Air Mineral 1500ml"]],
      ["is_cancelled", "=", 0],
      ["posting_date", "between", ["2026-09-01", "2026-09-30"]]
    ],
    "order_by": "posting_date desc, posting_time desc, creation desc",
    "limit_start": 0,
    "limit_page_length": 50
  }'
```

> - Gabungkan dengan `get_count` (§4.7) utk pagination. Daftar `item_code` bisa panjang — bila besar,
>   pecah menjadi beberapa request `in` atau persempit dgn warehouse/rentang tanggal (§2.1 no. 6).
> - `is_stock_item=1` praktis selalu benar utk SLE (hanya produk stok yang punya SLE) — filter ini
>   berguna terutama saat me-*resolve* daftar Item di Langkah 1.

---

## 5. GET pendukung UI (dropdown filter)

Dropdown berikut membantu user mengisi filter pada halaman riwayat/kartu stok. Semuanya
`frappe.client.get_list` (body-based).

### 5.1 Dropdown Item (is_stock_item)

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name", "item_name"],
    "filters": [["is_stock_item", "=", 1], ["disabled", "=", 0]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

> Dropdown membatasi pilihan ke produk stok aktif (`is_stock_item=1`, `disabled=0`) — hanya produk
> stok yang punya SLE. Untuk **menyaring daftar SLE** berdasarkan Item Group / `is_stock_item`
> (bukan filter langsung di dropdown, karena keduanya bukan kolom SLE), lihat **§4.9**.
> Detail Item: **[`prd_item.md`](./prd_item.md)**.

### 5.2 Dropdown Warehouse (per company, leaf)

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "fields": ["name"],
    "filters": [["company", "=", "PT Maju Jaya"], ["disabled", "=", 0], ["is_group", "=", 0]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

> Detail Warehouse: **[`prd_warehouse.md`](../setup/prd_warehouse.md)**.

### 5.3 Dropdown Company

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Company",
    "fields": ["name"],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

> Catatan: filter SLE tidak wajib per company (SLE punya `company` sendiri); dropdown ini hanya
> membantu mempersempit tampilan bila aplikasi multi-company.

---

## 6. Penanganan error umum

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | User tidak punya role baca (`Stock User` / `Accounts Manager`) | `{"message": "Not permitted"}` / `PermissionError` |
| 404 | Baris SLE tidak ditemukan (mis. `get` dengan `name` salah) | `{"exc_type":"DoesNotExistError","message":"Resource Not Found"}` |
| 417 | `fields`/`filters` memakai field yang tidak ada di doctype | `{"exc_type":"ValidationError","message":"..."}` / `Invalid filter` |
| 417 | Format tanggal/filter salah (mis. `between` bukan array 2 elemen) | `{"exc_type":"ValidationError","message":"..."}` |
| 417 | Filter memakai operator/tipe tidak cocok (mis. string utk field Float) | `{"exc_type":"ValidationError","message":"..."}` |
| — | List kosong (tidak ada data cocok) | Bukan error — `message: []` (untuk `get_value`/`get_last_doc` tanpa hasil: field `None`/`message` kosong) |

> **Catatan:**
> - Karena seluruh pemanggilan memakai `/api/method/...`, hasil sukses dibungkus di `message`; body
>   error berbentuk `{ "exc_type": ..., "exception": ..., "message": ... }`.
> - SLE read-only — tidak ada error terkait submit/cancel/frozen (operasi tersebut tidak dipakai di
>   dokumen ini). Yang paling sering salah di sisi frontend: **lupa filter `is_cancelled = 0`** dan
>   **meminta `fields` yang tidak ada** (mis. `item_name`/`amount` yang bukan field SLE).
> - Kinerja: tabel SLE besar — tanpa filter, request `limit_page_length = 0` bisa lambat/timeout;
>   selalu batasi rentang tanggal/scope dan paginasi (§2.1 no. 6).

---

## 7. Koleksi Postman

Seluruh pemanggilan di atas sudah tersedia dalam **satu koleksi Postman** yang siap import:
`docs/postman/postman_erpnext_api.json` (Collection v2.1). Koleksi ini berisi seluruh modul API
ERPNext (OAuth 2.0 + Supplier + Customer + Contact + Address + Lead + Employee + User + Warehouse
+ POS Profile + Item Group + Stock Reconciliation + **Stock Ledger**).

Folder **`13. Stock Ledger`** berisi **11 request** yang mencakup:
- Daftar SLE periode + company — `13.1`
- Kartu stok 1 item+warehouse (running balance, ascending) — `13.2`
- Verifikasi per voucher (`voucher_type`+`voucher_no`) — `13.3`
- Detail baris SLE (`get`) & cek cepat (`get_value`) — `13.4`–`13.5`
- Saldo terkini — baris terakhir (`get_list desc` / `get_last_doc`) — `13.6`
- Total count (`get_count`) — `13.7`
- Menyaring SLE per Item Group / `is_stock_item` — 2 langkah (§4.9): `13.8b` (Langkah 1: resolve Item per Item Group) lalu `13.1` / `13.7` (Langkah 2: SLE + count)
- Dropdown pendukung lain: Item (`13.8`), Warehouse (`13.9`), Company (`13.10`)

**Variabel yang perlu diisi** (Collection Variables):
- `sle_company` — company (mis. `PT Maju Jaya`)
- `sle_item` — `item_code` (mis. `Air Mineral 600ml`)
- `sle_warehouse` — warehouse (mis. `Toko Cikarang - PTMJ`)
- `sle_voucher_type` / `sle_voucher_no` — dokumen sumber utk verifikasi (mis. `Stock Reconciliation`
  / `MAT-RECO-00001-2026`)
- `sle_date_from` / `sle_date_to` — rentang tanggal filter
- `sle_item_group` — leaf Item Group utk menyaring dropdown Item (`13.8b`, mis. `Minuman`)
- `sle_row_name` — `name` baris SLE (terisi otomatis dari hasil `13.1`/`13.2` lewat test script)

Cara pakai sama dengan folder lain: isi variabel di atas, jalankan folder `0. OAuth 2.0` (atau Get
New Access Token), lalu jalankan request pada folder `13. Stock Ledger`. Request `13.1`/`13.2`
otomatis menyimpan `name` baris SLE pertama ke variabel `sle_row_name` lewat test script.

> **Dokumen terkait:** **[prd_stock_entry.md](./prd_stock_entry.md)** — dokumen sumber SLE pada
> alur transfer/pemakaian stok (transfer antar gudang, Material Issue/Receipt, transfer 2 tahap
> lewat gudang Transit), termasuk contoh verifikasi SLE per voucher di §6.4.
> **[prd_stock_balance.md](./prd_stock_balance.md)** — saldo stok per gudang/periode.
