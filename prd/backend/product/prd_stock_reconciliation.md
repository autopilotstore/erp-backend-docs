# PRD — REST API Doctype Stock Reconciliation (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Stock Reconciliation** di ERPNext (Frappe),
> diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Stock (ERPNext) — path `erpnext.stock.doctype.stock_reconciliation`
- **Doctype:** `Stock Reconciliation` — doctype **submittable** (`is_submittable: 1`), artinya punya
  siklus hidup `docstatus` (Draft → Submitted → Cancelled), lihat §2.1 & §4.6.
- **Versi API:** `/api/method/...` (API v1) — method whitelisted `frappe.client.*`, `name` dikirim di **body**
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

> Stock Reconciliation adalah **transaksi stok** (bukan master data seperti Item Group / Warehouse /
> Item). Fungsinya **mencocokkan / memperbaiki** qty (jumlah fisik) **dan/atau** nilai (`valuation_rate`)
> stok di sistem agar sesuai dengan kondisi nyata di gudang pada tanggal posting tertentu
> (deskripsi doctype: *"update or fix the quantity and valuation of stock ... typically used to
> synchronise the system values and what actually exists in your warehouses"*).
>
> Karena ini transaksi (bukan master), perilakunya berbeda dari PRD master lain:
> - Ada **posting** ke Stock Ledger Entry (SLE) + General Ledger (GL) saat di-*submit*.
> - Dokumen **draft** bisa diubah/dihapus; dokumen **submitted** tidak bisa diedit langsung — koreksi
>   dilakukan lewat **Cancel** (atau Amend) / dokumen baru.
> - **`docstatus` dikelola sistem** (0 = Draft, 1 = Submitted, 2 = Cancelled) — *jangan dikirim manual
>   di body*; transisi dilakukan lewat endpoint submit/cancel (§2.1, §4.2, §4.6).
> - Berbeda dengan Item/Warehouse, Stock Reconciliation v16 **hanya** memberikan akses ke role
>   **`Stock Manager`** (baca & tulis), lihat §2.1 no. 6.

---

## 1. Ruang lingkup

| # | Endpoint (POST `/api/method/...`) | Operasi | `name` di |
|---|---|---|---|
| 1 | `frappe.client.insert` | Buat Stock Reconciliation **draft** (CREATE, §4.1) | body (`doc`) |
| 2 | `frappe.client.submit_doc` | **Submit** draft → `docstatus=1` (posting SLE + GL, §4.2) | body |
| 3 | `frappe.client.get` | Ambil detail 1 Stock Reconciliation (READ, §4.3) | body |
| 4 | `frappe.client.get_list` | Daftar Stock Reconciliation (READ list, §4.4) | body (filters) |
| 5 | `frappe.client.get_count` | Total record Stock Reconciliation sesuai filter — pagination (§4.3) | body |
| 6 | `frappe.client.save` | Ubah **draft** (UPDATE, §4.5) | body (`doc`) |
| 7 | `frappe.client.set_value` | Ubah field tunggal pada **draft** — `posting_date`, `expense_account`, dst. (§4.5) | body |
| 8 | `frappe.client.cancel` | **Cancel** submitted → `docstatus=2` (membalik posting, §4.6) | body |
| 9 | `frappe.client.delete` | Hapus **draft** (`docstatus=0`, §4.7) | body |
| 10 | `erpnext.stock.doctype.stock_reconciliation.stock_reconciliation.get_items` | Prefill tabel `items` dari warehouse (UI, §5.1) | body |
| 11 | `erpnext.stock.doctype.stock_reconciliation.stock_reconciliation.get_stock_balance_for` | Ambil qty & rate berjalan utk 1 item+warehouse (UI, §5.2) | body |
| 12 | `erpnext.stock.doctype.stock_reconciliation.stock_reconciliation.get_difference_account` | Ambil akun selisih default (UI, §5.3) | body |
| 13 | `frappe.client.get_list` | Daftar Item / Warehouse / Account / Cost Center (dropdown pendukung, §5.4–§5.6) | body (filters) |
| 14 | `frappe.client.get_list` | Baca Stock Ledger Entry utk verifikasi (opsional, §5.7) | body (filters) |

> **Konvensi pemanggilan (penting):** seluruh operasi memakai method whitelisted **`frappe.client.*`**
> (kecuali method ERPNext khusus) dengan `name` (dan filter) dikirim lewat **body JSON**, bukan di
> URL path. Nama Stock Reconciliation dibuat sistem lewat **naming series** (`MAT-RECO-.YYYY.-`) —
> baca `name` dari respons `message` hasil CREATE, lalu kirim `name` itu di body untuk operasi
> selanjutnya (submit / get / save / cancel / delete).
> **Format respons:** method `/api/method/...` membungkus hasil di **`"message"`** (bukan `"data"`
> seperti `/api/resource/...`).

> **Perbedaan utama dengan master data lain (Item Group / Warehouse / Item):**
> - Item Group/Warehouse/Item adalah master: `name` = nilai unik yang dipilih user, tidak di-*submit*,
>   dan (untuk Warehouse) punya pola non-aktif.
> - Stock Reconciliation adalah **transaksi**: `name` dibuat sistem via naming series, dan dokumennya
>   melalui **submit** (`docstatus` 0 → 1) serta bisa di-**cancel** (`docstatus` 1 → 2). Tidak ada
>   field `disabled`; "menonaktifkan" efek stok dilakukan lewat cancel (§4.6).
> - Stock Reconciliation v16 **tidak** punya purpose "Repacking" (sudah dihapus sejak versi lama —
>   repacking kini via **Stock Entry**, purpose `Repack`). Purpose yang tersedia hanya `Stock
>   Reconciliation` dan `Opening Stock` (§2.1 no. 4).

> Dokumen ini hanya membahas **satu dokumen Stock Reconciliation** (stock opname / penyesuaian qty &
> nilai, plus catatan ringkas Opening Stock). Import/export masal (CSV upload) tidak dipakai di
> aplikasi ini (tidak didokumentasikan). Detail master **Item** ada di
> **[`prd_item.md`](./prd_item.md)** dan **Warehouse** di **[`prd_warehouse.md`](../setup/prd_warehouse.md)**.

---

## 2. Ringkasan field & data

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `company` | Link → Company | Perusahaan. `reqd: 1`. Menentukan mata uang `valuation_rate` dan akun selisih default. |
| 🔴 **WAJIB** | `purpose` | Select | `reqd: 1`. Opsi (v16): **`Stock Reconciliation`** (stock opname — alur utama dokumen ini) dan **`Opening Stock`** (stok awal, §4.8). Default UI: `Stock Reconciliation`. |
| 🔴 **WAJIB** | `posting_date` | Date | Tanggal efektif penyesuaian (default "Today"). Menentukan qty/rate "sebelum" yang dipakai untuk menghitung selisih. |
| 🔴 **WAJIB** | `posting_time` | Time | Jam efektif (default "Now"). Bersama `posting_date` menjadi acuan urutan di Stock Ledger. |
| 🔴 **WAJIB** | `items` | Table → Stock Reconciliation Item | Baris penyesuaian — **minimal 1 baris** yang punya perubahan qty/rate (§2.2). |
| 🟠 **DISARANKAN** | `expense_account` | Link → Account | **Akun selisih / Difference Account**. Bila kosong saat `validate`, sistem **otomatis mengisi** dari `Company → Stock Adjustment Account`. Saat perusahaan memakai **perpetual inventory**, akun ini wajib ada saat submit. Untuk `purpose = Opening Stock`, akunnya wajib tipe **Asset/Liability (Temporary Opening)**, bukan P&L (§4.8). |
| 🟠 **DISARANKAN** | `cost_center` | Link → Cost Center | Cost center untuk jurnal selisih. Bila kosong, sistem otomatis memakai `Company → Cost Center` default. |
| 🟠 | `set_posting_time` | Check | `1` = gunakan `posting_date`/`posting_time` yang dikirim (bisa backdated). `0` = sistem memakai tanggal/jam saat ini. **Disarankan `1`** agar tanggal posting deterministik dari frontend. |
| 🟠 | `set_warehouse` | Link → Warehouse | **Default Warehouse** (konvenien UI): dipakai untuk mengisi `warehouse` pada baris baru. Tidak wajib — setiap baris `items` tetap membawa `warehouse` sendiri. |
| ⚪ **Otomatis — jangan dikirim** | `name` | — | Dibuat sistem via naming series `MAT-RECO-.YYYY.-` (mis. `MAT-RECO-00001-2026`). Baca dari respons CREATE. |
| ⚪ **Dikelola sistem — jangan dikirim** | `docstatus` | — | `0` Draft, `1` Submitted, `2` Cancelled. Berubah lewat `submit_doc` / `cancel`, **bukan** dikirim manual di body (§2.1 no. 2). |
| ⚪ **Read-only — jangan dikirim** | `difference_amount` | Currency | Total selisih nilai (jumlah `amount_difference` seluruh baris), dihitung sistem. |
| ⚪ **Khusus UI scan (jangan dipakai API)** | `scan_mode`, `scan_barcode`, `last_scanned_warehouse` | — | Mode scan barcode. `scan_mode=1` **menonaktifkan auto-fetch** qty berjalan (untuk input manual). Tidak relevan untuk alur API normal. |
| ✖️ **Tidak ada di v16** | `disabled`, purpose `Repacking` | — | Bukan transaksi soft-delete (pakai cancel); "Repacking" sudah tidak ada — repacking via Stock Entry purpose `Repack`. |

> Catatan: daftar di atas adalah **data yang wajib/diperlukan** menurut kebutuhan aplikasi, bukan
> seluruh field doctype. Field `reqd` sebenarnya oleh doctype: `company`, `purpose`, `posting_date`,
> `posting_time`, dan `items` (dengan `item_code` + `warehouse` `reqd` di tiap baris) — sisanya
> opsional / terisi default di sisi backend, namun frontend tetap disarankan mengirim sesuai tabel
> agar deterministik.

### 2.1 Catatan penting

1. **Ini transaksi submittable — beda dengan master data.** `name` dibuat sistem (naming series
   `MAT-RECO-.YYYY.-`), bukan diisi user. Tidak ada cek duplikat nama seperti Item Group/Item.
   Siklus hidup: **Draft → (Submit) → Submitted → (Cancel) → Cancelled**. Submitted/Cancelled
   **tidak bisa diedit** — koreksi lewat Cancel lalu Amend, atau buat dokumen baru.
2. **`docstatus` dikelola sistem (0/1/2).** Frontend **tidak mengirim `docstatus`** pada `insert`/
   `save`. Transisi status hanya lewat endpoint: `frappe.client.submit_doc` (0 → 1) dan
   `frappe.client.cancel` (1 → 2). Keterangan lengkap tiap status + efek cancel setelah ada transaksi
   lanjutan ada di **§4.6**.
3. **Setiap dokumen butuh minimal satu baris yang benar-benar berubah.** Saat `save`/`submit`,
   sistem membandingkan `qty`/`valuation_rate` tiap baris dengan kondisi ledger saat ini
   (`current_qty`/`current_valuation_rate`); baris yang **tidak berubah akan dibuang otomatis**, dan
   bila **semua** baris tidak berubah → ditolak: `EmptyStockReconciliationItemsError` (*"None of the
   items have any change in quantity or value."*). Artinya CREATE/SAVE sebuah draft yang "kosong"
   (semua baris = kondisi sekarang) akan gagal — baris yang dikirim harus memang menyesuaikan qty,
   rate, atau keduanya.
4. **`purpose` (v16) hanya dua opsi: `Stock Reconciliation` & `Opening Stock`.**
   - `Stock Reconciliation` — **stock opname** (alur utama §4). Memperbaiki qty fisik dan/atau
     `valuation_rate`. Akun selisih umumnya `Company → Stock Adjustment Account` (P&L).
   - `Opening Stock` — **stok awal** (mis. perusahaan baru, §4.8). Akun selisih wajib **Asset /
     Liability (Temporary Opening)**, karena entry pembuka tidak boleh lewat P&L
     (`OpeningEntryAccountError` bila P&L dipilih). Biasanya dibuat sistem via tombol "Opening Stock"
     pada Item.
   - **Tidak ada `Repacking`** di v16 — repacking memakai **Stock Entry** (purpose `Repack`),
     didokumentasikan di PRD Stock Entry (terpisah).
5. **`expense_account` & `cost_center` terisi otomatis bila kosong.** Saat `validate`, bila
   `expense_account` kosong sistem memakai `Company.stock_adjustment_account`; bila `cost_center`
   kosong memakai `Company.cost_center`. Wajib terisi (tidak boleh kosong) saat submit bila company
   mengaktifkan **perpetual inventory** — pastikan Company sudah punya default tsb, atau kirim
   eksplisit di body. Untuk `purpose = Opening Stock`, akun selisih **tidak boleh** akun P&L.
6. **Role yang dibutuhkan (v16).** Permission doctype Stock Reconciliation di v16 **hanya** untuk
   role **`Stock Manager`** (create/read/write/submit/cancel/amend/delete). Jadi semua operasi di
   dokumen ini — termasuk **membaca** — butuh user ber-role `Stock Manager`. Bila tim UI perlu akses
   baca untuk role lain (mis. `Stock User`), admin harus menambahkan permission tsb secara manual
   (customization) — **tidak tersedia by default**.
7. **Tidak bisa diubah setelah submit.** Untuk memperbaiki SR yang sudah submitted: **Cancel** dulu
   (membalik posting), lalu **Amend** (salinan baru `docstatus=0` dengan `amended_from`) untuk
   memasukkan nilai yang benar, atau buat dokumen baru. Jangan mengedit langsung (ditolak backend).

### 2.2 `items` — child table `Stock Reconciliation Item` (baris penyesuaian)

- Satu baris mewakili **satu kombinasi `item_code` + `warehouse`** (+ opsional batch/serial).
- `qty` = **qty fisik baru** hasil opname; `valuation_rate` = **nilai per unit baru**. Sistem
  membandingkan keduanya dengan nilai berjalan (`current_qty`, `current_valuation_rate`) lalu
  memposting selisihnya.
- Boleh mengubah **hanya qty** (nilai ikut dihitung ulang dgn rate), **hanya valuation_rate**
  (revaluasi nilai tanpa ubah qty), atau **keduanya**.

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `item_code` | Link → Item | Produk. `reqd: 1`. Harus `is_stock_item=1` (bukan jasa). |
| 🔴 **WAJIB** | `warehouse` | Link → Warehouse | Gudang. `reqd: 1`. Harus leaf (`is_group=0`) milik `company` yang sama. |
| 🟠 **DISARANKAN** | `qty` | Float | Qty fisik baru. Boleh dikosongkan bila hanya ingin mengubah `valuation_rate`. **Tidak boleh negatif**. Bila diisi dan `valuation_rate` kosong → sistem mencoba mengambil rate dari sistem (§2.3). |
| 🟠 **DISARANKAN** | `valuation_rate` | Currency | Nilai per unit baru. Boleh dikosongkan bila hanya ingin mengubah qty (§2.3). **Tidak boleh negatif**. Mata uang mengikuti `company`. |
| 🟠 | `allow_zero_valuation_rate` | Check | `1` = izinkan `valuation_rate = 0` (mis. stok awal bernilai 0 / opening tanpa nilai). |
| 🟠 | `batch_no` / `serial_no` / `use_serial_batch_fields` | — | Untuk produk **berbatch / berseri** (lanjutan). Mengubah qty/rate item serial/batch memakai mekanisme `Serial and Batch Bundle`. Umumnya tidak dipakai untuk alur opname sederhana — abaikan bila item tidak serial/batch. |
| ⚪ **Read-only (dihitung sistem)** | `current_qty`, `current_valuation_rate`, `current_amount`, `stock_uom` | — | Nilai stok **sebelum** penyesuaian (dari ledger pada `posting_date/time`). `current_*` terisi otomatis saat baris diisi (`item_code`+`warehouse`); jangan dikirim. |
| ⚪ **Read-only (dihitung sistem)** | `amount` (= `qty * valuation_rate`), `quantity_difference` (= `qty - current_qty`), `amount_difference` (= selisih nilai) | — | Hasil hitung; jangan dikirim. |

### 2.3 `valuation_rate` — perilaku saat dikosongkan (penting)

Pada baris `items`, `qty` dan `valuation_rate` saling independen — boleh mengisi salah satu atau
keduanya:

- **`qty` diisi, `valuation_rate` dikosongkan** → ERPNext **mencoba mengambil valuation rate terakhir
  dari sistem** (dari Stock Ledger item+warehouse tsb pada/paling lambat tanggal posting — nilai yang
  sama dengan `current_valuation_rate`), lalu memposting qty baru dengan rate itu. Urutan fallback
  backend bila stok sebelumnya tidak ada/nol:
  1. rate dari saldo stok terakhir di ledger (`current_valuation_rate`),
  2. bila tidak ada → harga beli (`Item Price`, buying, mata uang default company),
  3. bila masih tidak ada → field `valuation_rate` pada master `Item`.
- **Bila tidak ada rate yang bisa diambil** (tidak ada riwayat stok, tidak ada Item Price, dan
  `valuation_rate` Item kosong) **dan** `qty > 0`, submit **ditolak** backend:
  *"Valuation Rate required for Item {item_code} at row {idx}"* — kecuali
  `allow_zero_valuation_rate = 1`.
- **`qty` dikosongkan, `valuation_rate` diisi** → **revaluasi nilai** (mengubah nilai stok tanpa
  mengubah qty). Berguna utk menyesuaikan nilai persediaan (mis. setelah ditemukan selisih nilai).
- **Keduanya dikosongkan** pada baris yang punya stok berjalan → baris dianggap "tidak berubah" dan
  dibuang (lihat §2.1 no. 3).

> Implikasi UI: untuk skenario umum stock opname, frontend cukup mengirim `qty` hasil hitung fisik;
> biarkan `valuation_rate` kosong agar sistem memakai rate berjalan — **kecuali** memang ingin
> mengoreksi nilainya (kirim `valuation_rate` baru) atau item tidak punya riwayat nilai sama sekali
> (harus kirim rate, atau set `allow_zero_valuation_rate`).

### 2.4 Tabel-tabel terkait saat Stock Reconciliation diposting

Stock Reconciliation adalah transaksi yang **diposting** — saat `docstatus` berubah dari `0`
(draft) ke `1` (submitted), sistem menulis/memperbarui beberapa tabel; saat di-cancel (`2`) sebagian
ditulis ulang/ dibalik. Tabel berikut yang terlibat:

| Tabel (`tab...`) | Doctype | Peran saat Stock Reconciliation diposting |
|---|---|---|
| `tabStock Reconciliation` | `Stock Reconciliation` | **Dokumen transaksi** (header). `name` via naming series `MAT-RECO-.YYYY.-`; kolom `docstatus` (0/1/2), `purpose`, `posting_date/time`, `expense_account`, `cost_center`, `difference_amount`. |
| `tabStock Reconciliation Item` | `Stock Reconciliation Item` | **Child table `items`** — baris penyesuaian per `item_code`+`warehouse` (qty/rate baru + nilai read-only `current_*`, `amount_difference`). |
| `tabStock Ledger Entry` | `Stock Ledger Entry` | **Bukti posting stok.** Saat submit/cancel, dibuat SLE utk tiap baris yang berubah: `voucher_type = "Stock Reconciliation"`, `voucher_no = <name SR>`, `voucher_detail_no = <row name>`, `actual_qty`/`qty_after_transaction`/`valuation_rate`/`stock_value_difference`, tanda `is_cancelled` (0=aktif, 1=entry pembalik saat cancel). Di sinilah selisih qty/rate **menjadi riwayat** yang dipakai menghitung nilai item+warehouse setelahnya. |
| `tabGL Entry` | `GL Entry` | **Jurnal selisih nilai** (bila company memakai **perpetual inventory**): debit/kredit akun persediaan berhadapan dengan `expense_account` (Difference Account) + `cost_center`; `voucher_type = "Stock Reconciliation"`. Untuk `purpose = Opening Stock`, akun lawan = Temporary Opening (Asset/Liability). Saat cancel, GL di-*reverse*. |
| `tabBin` | `Bin` | **Saldo stok per item+warehouse** (`actual_qty`, `stock_value`, `valuation_rate`, `reserved_qty`) — diperbarui otomatis mengikuti SLE. Sumber nilai "sebelum" (`current_qty`/`current_valuation_rate`) dan dasar info stok di UI (`get_list` Bin). |
| `tabSerial and Batch Bundle` (+ `tabSerial and Batch Bundle Item`) | `Serial and Batch Bundle` | Hanya utk baris item **serial/batch** (bila memakai `serial_and_batch_bundle` / `use_serial_batch_fields`) — bundle stok lama & baru dibuat/dibalik saat submit/cancel (lanjutan, umumnya tidak dipakai di opname sederhana). |
| `tabItem Standard Cost` | `Item Standard Cost` | Khusus item yang memakai metode valuasi **Standard Cost**: saat submit, sistem bisa membuat/memperbarui Item Standard Cost dari `valuation_rate` baris (mis. utk opening/revaluation standard cost). Saat cancel, record terkait dibatalkan. |
| `tabRepost Item Valuation` | `Repost Item Valuation` | Dibuat & dijalankan (umumnya **background**) bila posting SR menyisakan **transaksi lanjutan** — mis. SR backdated / saat cancel — agar seluruh SLE & GL **setelah** tanggal posting SR dihitung ulang konsisten. Statusnya bisa dipantau lewat dokumen ini (lihat §4.6). |

> Ringkasan alur tulis saat **submit** (`docstatus` 0 → 1): hitung selisih tiap baris → buat
> **SLE** (`tabStock Ledger Entry`) → buat **GL** (`tabGL Entry`) bila perpetual inventory →
> perbarui **Bin** → (bila ada transaksi sesudahnya) antrekan **Repost Item Valuation** → (standard
> cost) kelola **Item Standard Cost**. Saat **cancel** (`1` → 2): SLE milik SR ditandai
> `is_cancelled`/dibalik, GL di-reverse, Bin dihitung ulang, lalu transaksi lanjutan di-reposting.
> Verifikasi dari sisi UI cukup membaca SLE/GL dengan filter `voucher_type`/`voucher_no` (§5.7), dan
> saldo terkini lewat Bin.

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Detail lengkap setup OAuth Client ada di **[`prd_oauth.md`](../prd_oauth.md)**.

---

## 4. CRUD — siklus hidup Stock Reconciliation

> **Alur umum (paling aman):**
> 1. `frappe.client.insert` → **draft** (`docstatus=0`), baca `name` dari `message`.
> 2. (opsional) `frappe.client.save` / `set_value` untuk menyempurnakan draft.
> 3. `frappe.client.submit_doc` → **submitted** (`docstatus=1`); saat inilah posting SLE + GL terjadi.
> 4. (bila salah / ingin koreksi) `frappe.client.cancel` → **cancelled** (`docstatus=2`) lalu Amend /
>    buat dokumen baru.

### 4.1 CREATE (draft) — `frappe.client.insert`

**Payload minimum (purpose `Stock Reconciliation`, sesuaikan qty fisik):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Stock Reconciliation",
      "company": "PT Maju Jaya",
      "purpose": "Stock Reconciliation",
      "posting_date": "2026-09-04",
      "posting_time": "09:15:00",
      "set_posting_time": 1,
      "items": [
        {
          "item_code": "Air Mineral 600ml",
          "warehouse": "Toko Cikarang - PTMJ",
          "qty": 120
        }
      ]
    }
  }'
```

> `valuation_rate` dikosongkan → sistem memakai rate terakhir dari ledger (§2.3). `expense_account`/
> `cost_center` boleh dikosongkan (terisi otomatis dari Company), atau dikirim eksplisit.

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": {
    "name": "MAT-RECO-00001-2026",
    "owner": "Administrator",
    "creation": "2026-09-04 09:10:00.000000",
    "modified": "2026-09-04 09:10:00.000000",
    "modified_by": "Administrator",
    "docstatus": 0,
    "idx": 0,
    "company": "PT Maju Jaya",
    "purpose": "Stock Reconciliation",
    "posting_date": "2026-09-04",
    "posting_time": "09:15:00",
    "set_posting_time": 1,
    "expense_account": "Stock Adjustment - PTMJ",
    "cost_center": "Main - PTMJ",
    "difference_amount": 0,
    "items": [
      {
        "name": "abc123",
        "item_code": "Air Mineral 600ml",
        "warehouse": "Toko Cikarang - PTMJ",
        "qty": 120,
        "stock_uom": "Nos",
        "current_qty": 100,
        "current_valuation_rate": 4500,
        "current_amount": 450000,
        "valuation_rate": 4500,
        "amount": 540000,
        "quantity_difference": 20,
        "amount_difference": 90000
      }
    ]
  }
}
```

> - `docstatus: 0` = masih draft (belum diposting).
> - `name` = `MAT-RECO-00001-2026` → simpan nilai ini untuk operasi berikutnya.
> - `valuation_rate` pada baris sudah terisi otomatis 4500 (rate terakhir dari sistem, §2.3).
> - Bila **semua** baris ternyata tidak berubah terhadap kondisi ledger → error
>   `EmptyStockReconciliationItemsError` (§6).

Variants berikut cukup dijadikan nilai **`doc`** pada `frappe.client.insert` di atas.

**Varian A — hanya koreksi nilai (revaluasi, tanpa ubah qty):**

```json
{
  "doctype": "Stock Reconciliation",
  "company": "PT Maju Jaya",
  "purpose": "Stock Reconciliation",
  "posting_date": "2026-09-04",
  "posting_time": "09:20:00",
  "set_posting_time": 1,
  "expense_account": "Stock Adjustment - PTMJ",
  "cost_center": "Main - PTMJ",
  "items": [
    {
      "item_code": "Air Mineral 1500ml",
      "warehouse": "Gudang Pusat - PTMJ",
      "valuation_rate": 7500
    }
  ]
}
```

> `qty` dikosongkan, `valuation_rate` diisi → stok di-*revalue* ke 7500/unit tanpa mengubah qty.

**Varian B — qty & rate dikoreksi sekaligus:**

```json
{
  "doctype": "Stock Reconciliation",
  "company": "PT Maju Jaya",
  "purpose": "Stock Reconciliation",
  "posting_date": "2026-09-04",
  "posting_time": "09:30:00",
  "set_posting_time": 1,
  "items": [
    {
      "item_code": "Air Mineral 600ml",
      "warehouse": "Toko Cikarang - PTMJ",
      "qty": 90,
      "valuation_rate": 4600
    }
  ]
}
```

### 4.2 SUBMIT — `frappe.client.submit_doc` (draft → submitted)

Setelah draft dibuat & diverifikasi, submit untuk mem-posting:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.submit_doc \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Reconciliation",
    "name": "MAT-RECO-00001-2026"
  }'
```

**Efek saat submit (`docstatus` 0 → 1):**
- Membuat **Stock Ledger Entry (SLE)** untuk setiap selisih (qty/rate) per item+warehouse pada
  `posting_date/time`.
- Bila company memakai **perpetual inventory**, membuat **General Ledger Entry** atas selisih nilai
  (`difference_amount`) — stok (balance sheet) di-*debit/credit* berhadapan dengan akun selisih
  (`expense_account`) + `cost_center`. Untuk `purpose = Opening Stock`, akun selisih harus akun
  Temporary (Asset/Liability).
- Baris yang tidak berubah dibuang; bila tidak ada SLE yang dibuat sama sekali → error
  *"No stock ledger entries were created ..."*.
- Bila ada item ber-*reserved stock* yang qty-nya berubah → submit ditolak (lihat §4.6).

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": {
    "name": "MAT-RECO-00001-2026",
    "docstatus": 1,
    "difference_amount": 90000,
    "items": [ { "name": "abc123", "item_code": "Air Mineral 600ml", "qty": 120, "current_qty": 100, "valuation_rate": 4500, "amount_difference": 90000 } ]
  }
}
```

> Setelah submit, dokumen **tidak bisa diedit** (`frappe.client.save`/`set_value` akan ditolak).
> Koreksi = Cancel lalu Amend / dokumen baru (§2.1 no. 7, §4.6).

### 4.3 READ (satu record) & total count

**Langkah 1 — Ambil detail Stock Reconciliation (`frappe.client.get`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Reconciliation",
    "name": "MAT-RECO-00001-2026"
  }'
```

Respons `message` berisi seluruh field (header + child table `items`, termasuk `docstatus` dan
`difference_amount`). Field read-only (`current_*`, `amount_difference`, dst.) ikut terbawa —
jangan dikirim ulang saat `save`.

**Total count — `frappe.client.get_count`** (pagination / lazy loading):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_count \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Reconciliation",
    "filters": [["docstatus","=",1],["company","=","PT Maju Jaya"]]
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": 7
}
```

### 4.4 READ (daftar) — `frappe.client.get_list`

```bash
# Daftar — halaman 1, hanya yang sudah submitted, per company
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Reconciliation",
    "fields": ["name","docstatus","posting_date","posting_time","purpose","company","difference_amount"],
    "filters": [["docstatus","=",1],["company","=","PT Maju Jaya"]],
    "order_by": "posting_date desc, name desc",
    "limit_start": 0,
    "limit_page_length": 50
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "name": "MAT-RECO-00002-2026", "docstatus": 1, "posting_date": "2026-09-04", "posting_time": "10:00:00", "purpose": "Stock Reconciliation", "company": "PT Maju Jaya", "difference_amount": -12000 }
  ]
}
```

> Halaman berikutnya naikkan `limit_start` kelipatan `50`; `limit_page_length=0` = ambil **semua**.
> Filter `docstatus` berguna untuk memisahkan daftar Draft / Submitted / Cancelled pada UI.

### 4.5 UPDATE (draft only) — `frappe.client.save` & `frappe.client.set_value`

Update memakai `save`: kirim dokumen lengkap (hasil `frappe.client.get` yang dimodifikasi);
`name` ada di dalam body. **Hanya berlaku saat `docstatus = 0`.**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Stock Reconciliation",
      "name": "MAT-RECO-00001-2026",
      "company": "PT Maju Jaya",
      "purpose": "Stock Reconciliation",
      "posting_date": "2026-09-04",
      "posting_time": "09:15:00",
      "set_posting_time": 1,
      "expense_account": "Stock Adjustment - PTMJ",
      "cost_center": "Main - PTMJ",
      "items": [
        { "item_code": "Air Mineral 600ml", "warehouse": "Toko Cikarang - PTMJ", "qty": 125 }
      ]
    }
  }'
```

> **Catatan:**
> - `save` membangun ulang dokumen dari dict — kirim dokumen konsisten/lengkap. Child table `items`
>   berlaku **replace-all** — kirim seluruh baris yang diinginkan.
> - `docstatus` pada body **harus tetap 0**. Mengirim dokumen ber-`docstatus=1` lewat `save` ditolak.
> - Mengubah `posting_date`/`posting_time` draft bebas; setelah submit tidak bisa (harus cancel).

**Perubahan kecil — `frappe.client.set_value`** (draft only):

```bash
# Ubah akun selisih pada draft
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Reconciliation",
    "name": "MAT-RECO-00001-2026",
    "fieldname": { "expense_account": "Stock Adjustment - PTMJ" }
  }'
```

### 4.6 CANCEL & `docstatus` — `frappe.client.cancel` (submitted → cancelled)

**Keterangan `docstatus`:**

| `docstatus` | Label | Keterangan |
|---|---|---|
| `0` | **Draft** | Baru dibuat, **belum diposting** — tidak ada efek ke Stock Ledger/GL. Bisa diedit (`save`/`set_value`) dan dihapus (`delete`). Status awal setelah `frappe.client.insert`. |
| `1` | **Submitted** | Sudah di-*submit* (`frappe.client.submit_doc`) → **diposting**: SLE (selisih qty/rate) + GL (selisih nilai ke akun selisih) dibuat pada `posting_date/time`. Dokumen terkunci — tidak bisa diedit langsung. |
| `2` | **Cancelled** | Sudah di-*cancel* (`frappe.client.cancel`) → **posting dibatalkan**: SLE & GL milik SR di-reverse, nilai item+warehouse yang terdampak dikembalikan ke kondisi sebelum SR (via reposting transaksi sesudahnya). Dokumen tetap tersimpan (tidak dihapus) tapi tidak aktif — tidak bisa diedit, bisa di-Amend untuk membuat draft koreksi baru. |

**Request cancel:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.cancel \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Reconciliation",
    "name": "MAT-RECO-00001-2026"
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": {
    "name": "MAT-RECO-00001-2026",
    "docstatus": 2,
    "difference_amount": 0,
    "items": [ { "name": "abc123", "item_code": "Air Mineral 600ml", "qty": 120, "current_qty": 120, "valuation_rate": 4500, "amount_difference": 0 } ]
  }
}
```

**Apa yang terjadi saat cancel (`docstatus` 1 → 2):**
- Sistem **membalik SLE** milik SR (entry pembalik, `is_cancelled`/reverse) dan **membatalkan GL**-nya
  (`make_gl_entries_on_cancel`).
- Untuk item+warehouse yang masih punya transaksi **setelah** tanggal posting SR, sistem memicu
  **reposting** (Repost Item Valuation, biasanya di background) sehingga seluruh SLE/GL sesudahnya
  dihitung ulang agar konsisten dengan kondisi "SR dibatalkan".

**Penting — cancel setelah ada transaksi lanjutan (konsekuensi):**
- Stock Reconciliation yang di-*submit* sering menjadi **dasar nilai/qty** bagi transaksi lanjutan
  (mis. Delivery Note / Stock Entry / transaksi yang memakai qty & valuation rate hasil SR untuk
  menghitung COGS atau stok keluar).
- **Cancel SR setelah transaksi lanjutan TIDAK membatalkan transaksi lanjutan itu sendiri** —
  transaksinya tetap `Submitted`, tetapi **nilainya di-reposting ulang** (stock value / valuation
  rate / COGS / selisih) sesuai kondisi stok sebelum SR. Akibatnya angka di dokumen lama (dan
  laporan/GL) bisa berubah secara retroaktif.
- **Batasan sebelum cancel berhasil:**
  - Bila item di SR kini punya **reserved stock** (mis. sudah ada Sales Order / reservasi stok yang
    belum terpenuhi) → cancel **ditolak** backend (`validate_reserved_stock`).
  - Bila periode posting sudah **ditutup** (Period Closing / frozen date / Accounting Period closed)
    → cancel **ditolak**.
  - Role wajib **`Stock Manager`** (§2.1 no. 6).
- **Rekomendasi UI:** tampilkan **konfirmasi** berisi peringatan bahwa cancel akan membatalkan
  posting SR dan menghitung ulang nilai transaksi stok sesudahnya. Bila SR sudah lama & sudah banyak
  transaksi lanjutan, umumnya **lebih aman membuat SR koreksi baru** (dokumen baru, bukan cancel),
  karena cancel memicu reposting yang bisa mengubah angka historis.

**Amend (koreksi setelah cancel):** untuk membuat salinan baru dari SR yang di-cancel, gunakan
alur: `cancel` (di atas) → `frappe.client.insert` dengan `doc.amended_from = "<nama SR yang
di-cancel>"` (nama baru tetap dibuat sistem) → `submit_doc`. Opsional; umumnya tim UI lebih suka
membuat dokumen baru dari nol.

### 4.7 Hapus draft — `frappe.client.delete`

Hanya draft (`docstatus=0`) yang boleh dihapus. Submitted/Cancelled **tidak bisa dihapus** (harus
dibiarkan / di-Amend):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.delete \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Reconciliation",
    "name": "MAT-RECO-00001-2026"
  }'
```

> Karena draft belum diposting, penghapusan tidak berdampak ke stok/akuntansi. Ini satu-satunya
> operasi "hapus" yang wajar — **jangan** mencoba `delete` pada dokumen `docstatus=1` (ditolak).

### 4.8 Opening Stock (stok awal) — ringkas

Untuk mengisi stok awal (mis. setup perusahaan baru / migrasi, sebelum ada transaksi stok lain),
pakai `purpose = "Opening Stock"` dengan payload yang sama seperti §4.1, perbedaan:

- `purpose` = **`Opening Stock`**.
- Akun selisih (`expense_account`) wajib akun tipe **Asset/Liability (Temporary Opening)** — bila
  memilih akun P&L, backend menolak: `OpeningEntryAccountError` (*"Difference Account must be an
  Asset/Liability type account, since this Stock Reconciliation is an Opening Entry"*). Contoh nama
  akun: `Temporary Opening - PTMJ`.
- `valuation_rate` bersifat wajib secara praktik (belum ada riwayat nilai) — bila sengaja 0, set
  `allow_zero_valuation_rate = 1`.

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Stock Reconciliation",
      "company": "PT Maju Jaya",
      "purpose": "Opening Stock",
      "posting_date": "2026-09-01",
      "posting_time": "00:00:01",
      "set_posting_time": 1,
      "expense_account": "Temporary Opening - PTMJ",
      "cost_center": "Main - PTMJ",
      "items": [
        { "item_code": "Air Mineral 600ml", "warehouse": "Toko Cikarang - PTMJ", "qty": 100, "valuation_rate": 4500 }
      ]
    }
  }'
```

> Lalu `frappe.client.submit_doc` untuk mem-posting. Skenario ini biasanya dibuat otomatis oleh
> ERPNext lewat tombol "Opening Stock" pada master Item (menggunakan akun Temporary yang diset di
> Company) — frontend cukup meniru pola di atas bila memang perlu via API. **Alur utama aplikasi
> tetaplah `purpose = "Stock Reconciliation"`** (stock opname berkala).

---

## 5. GET pendukung UI

### 5.1 GET prefill tabel `items` dari warehouse — `get_items`

Saat membuka halaman stock opname untuk sebuah warehouse, ambil daftar item ber-stok beserta qty &
rate berjalannya (agar frontend tinggal mengisi qty hasil opname):

```bash
curl -X POST https://site-anda.com/api/method/erpnext.stock.doctype.stock_reconciliation.stock_reconciliation.get_items \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "warehouse": "Toko Cikarang - PTMJ",
    "posting_date": "2026-09-04",
    "posting_time": "09:00:00",
    "company": "PT Maju Jaya",
    "ignore_empty_stock": 1
  }'
```

> `item_code` opsional (filter 1 item); `ignore_empty_stock=1` → hanya item yang punya stok.
> `posting_date`/`posting_time` menentukan qty/rate "sebelum" (posisi ledger saat itu).

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "item_code": "Air Mineral 600ml", "warehouse": "Toko Cikarang - PTMJ", "qty": 100, "valuation_rate": 4500, "stock_uom": "Nos", "item_name": "Air Mineral 600ml", "has_serial_no": 0, "has_batch_no": 0 },
    { "item_code": "Air Mineral 1500ml", "warehouse": "Toko Cikarang - PTMJ", "qty": 50, "valuation_rate": 7200, "stock_uom": "Nos", "item_name": "Air Mineral 1500ml", "has_serial_no": 0, "has_batch_no": 0 }
  ]
}
```

### 5.2 GET qty & rate berjalan utk 1 item+warehouse — `get_stock_balance_for`

Saat user menambah/ubah baris `items` (isi `item_code` + `warehouse`), ambil `current_qty` &
`current_valuation_rate` agar frontend bisa menampilkan kondisi sekarang & menghitung selisih:

```bash
curl -X POST https://site-anda.com/api/method/erpnext.stock.doctype.stock_reconciliation.stock_reconciliation.get_stock_balance_for \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "item_code": "Air Mineral 600ml",
    "warehouse": "Toko Cikarang - PTMJ",
    "posting_date": "2026-09-04",
    "posting_time": "09:00:00",
    "company": "PT Maju Jaya",
    "with_valuation_rate": true
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": { "qty": 100, "rate": 4500, "use_serial_batch_fields": 0 }
}
```

> Endpoint ini butuh permission **write** Stock Reconciliation (role Stock Manager). Nilai `qty`/
> `rate` inilah yang dipakai sebagai `current_qty`/`current_valuation_rate` pada baris (`get_stock_balance_for`
> adalah method yang sama dengan yang dipanggil form Desk saat baris diisi).

### 5.3 GET akun selisih default — `get_difference_account`

Untuk **prefill `expense_account`** sesuai `purpose` & company (bukan menyimpan, hanya membaca):

```bash
curl -X POST https://site-anda.com/api/method/erpnext.stock.doctype.stock_reconciliation.stock_reconciliation.get_difference_account \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "purpose": "Stock Reconciliation",
    "company": "PT Maju Jaya"
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": "Stock Adjustment - PTMJ"
}
```

> Untuk `purpose = "Stock Reconciliation"` mengembalikan `Company → Stock Adjustment Account`; untuk
> `purpose = "Opening Stock"` mengembalikan akun **Temporary** (`account_type = Temporary`, leaf)
> milik company.

### 5.4 GET dropdown Item (is_stock_item) — `frappe.client.get_list`

Untuk memilih `item_code` pada baris `items` (hanya produk stok yang boleh):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name"],
    "filters": [["is_stock_item","=",1],["disabled","=",0]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

### 5.5 GET dropdown Warehouse — `frappe.client.get_list`

Untuk field `warehouse` pada baris `items` / `set_warehouse` (dibatasi per company, hanya leaf):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "fields": ["name"],
    "filters": [["company","=","PT Maju Jaya"],["disabled","=",0],["is_group","=",0]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

> Detail Warehouse: **[prd_warehouse.md](../setup/prd_warehouse.md)**.

### 5.6 GET dropdown Account & Cost Center — `frappe.client.get_list`

Untuk field `expense_account` (akun selisih) dan `cost_center` (leaf per company):

```bash
# Account (pilih akun selisih; utk Opening Stock batasi ke akun Temporary/Asset-Liability)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Account",
    "fields": ["name"],
    "filters": [["company","=","PT Maju Jaya"],["is_group","=",0]],
    "limit_page_length": 0
  }'

# Cost Center (leaf, per company)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Cost Center",
    "fields": ["name"],
    "filters": [["company","=","PT Maju Jaya"],["is_group","=",0]],
    "limit_page_length": 0
  }'
```

> Catatan pemilihan akun: untuk `purpose = Stock Reconciliation`, akun selisih biasanya
> `Company → Stock Adjustment Account` (tipe P&L/expense); untuk `purpose = Opening Stock` **wajib**
> akun **Asset/Liability (Temporary Opening)** — backend menolak akun P&L pada Opening Stock (§4.8).

### 5.7 GET verifikasi via Stock Ledger Entry (opsional) — `frappe.client.get_list`

Setelah submit, frontend dapat menampilkan bukti posting dengan membaca SLE milik SR tsb:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Ledger Entry",
    "fields": ["name","item_code","warehouse","actual_qty","qty_after_transaction","valuation_rate","stock_value_difference","posting_datetime"],
    "filters": [["voucher_type","=","Stock Reconciliation"],["voucher_no","=","MAT-RECO-00001-2026"],["is_cancelled","=",0]],
    "order_by": "creation asc",
    "limit_page_length": 0
  }'
```

> Detail lengkap doctype Stock Ledger Entry & GL Entry (bila perlu menampilkan jurnal selisih)
> didokumentasikan di PRD Stok/Accounting (terpisah).

---

## 6. Penanganan error umum

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | Role bukan `Stock Manager` (baca/tulis/submit/cancel) | `{"message": "Not permitted"}` |
| 404 | Resource tidak ditemukan | `{"exc_type":"DoesNotExistError","message":"Resource Not Found"}` |
| 417 | Field wajib kosong (`reqd`) | `{"exc_type":"MandatoryError","message":"purpose is mandatory"}` (juga `company`, `posting_date`, `posting_time`, `items`; per baris: `item_code`, `warehouse`) |
| 417 | Tidak ada perubahan pada semua baris | `{"exc_type":"EmptyStockReconciliationItemsError","message":"None of the items have any change in quantity or value."}` |
| 417 | Qty diisi tapi `valuation_rate` kosong & tidak ada rate bisa diambil | `{"exc_type":"ValidationError","message":"Valuation Rate required for Item Air Mineral 600ml at row 1"}` |
| 417 | Qty / rate negatif | `{"exc_type":"ValidationError","message":"Negative Quantity is not allowed"}` / `"Negative Valuation Rate is not allowed"` |
| 417 | Item/warehouse punya **reserved stock** saat submit/cancel | `{"exc_type":"ValidationError","message":"..."}` berisi daftar item+warehouse yang ter-reservasi |
| 417 | Opening Stock memakai akun P&L | `{"exc_type":"OpeningEntryAccountError","message":"Difference Account must be an Asset/Liability type account, since this Stock Reconciliation is an Opening Entry"}` |
| 417 | Perpetual inventory aktif tapi akun selisih kosong | `{"exc_type":"ValidationError","message":"Please enter Expense Account"}` |
| 417 | Submit tanpa SLE yang dibuat | `{"exc_type":"ValidationError","message":"No stock ledger entries were created. Please set the quantity or valuation rate for the items properly and try again."}` |
| 417 | Posting di periode terkunci / ditutup | `{"exc_type":"FrozenAccountError","message":"..."}` / pesan Accounting Period / Period Closing |
| 417 | Submit/cancel pada status yang salah (mis. cancel draft, submit ulang submitted) | `{"exc_type":"ValidationError","message":"..."}` / `"Not allowed"` |

> **Catatan:** karena seluruh pemanggilan memakai `/api/method/...`, hasil sukses dibungkus di
> `message`. Body error berbentuk `{ "exc_type": ..., "exception": ..., "message": ... }`. Bila
> operasi submit/cancel memicu reposting massal, ERPNext bisa menjalankannya di **background**
> (progress "Reposting in progress" pada daftar/Report) — frontend dapat menampilkan status tersebut
> dan me-refresh setelah selesai.

---

## 7. Koleksi Postman

Seluruh pemanggilan di atas sudah tersedia dalam **satu koleksi Postman** yang siap import:
`docs/postman/postman_erpnext_api.json` (Collection v2.1). Koleksi ini berisi seluruh modul API
ERPNext (OAuth 2.0 + Supplier + Customer + Contact + Address + Lead + Employee + User + Warehouse
+ POS Profile + Item Group + **Stock Reconciliation**).

Folder **`12. Stock Reconciliation`** (nomor `11` di-reserve untuk folder `Item`) berisi
**20 request** yang mencakup:
- CREATE draft (opname, revaluasi, qty+rate) — `12.1`–`12.3`
- SUBMIT (`submit_doc`) — `12.4`
- READ single / list / count — `12.5`–`12.7`
- UPDATE draft (`save` / `set_value`) — `12.8`–`12.9`
- CANCEL & delete draft — `12.10`–`12.11`
- Opening Stock — `12.12`
- GET pendukung UI (`get_items`, `get_stock_balance_for`, `get_difference_account`) — `12.13`–`12.15`
- Dropdown pendukung (Item / Warehouse / Account / Cost Center) — `12.16`–`12.19`
- Verifikasi Stock Ledger Entry — `12.20`

**Variabel yang perlu diisi** (Collection Variables):
- `stock_reco_id` — `name` hasil CREATE (mis. `MAT-RECO-00001-2026`)
- `stock_reco_company` — company (mis. `PT Maju Jaya`)
- `stock_reco_warehouse` — warehouse (mis. `Toko Cikarang - PTMJ`)
- `stock_reco_item` — `item_code` (mis. `Air Mineral 600ml`)
- `stock_reco_posting_date` / `stock_reco_posting_time` — tanggal/jam posting
- `stock_adjustment_account` — akun selisih (mis. `Stock Adjustment - PTMJ`)

Cara pakai sama dengan folder lain: isi variabel di atas, jalankan folder `0. OAuth 2.0` (atau Get
New Access Token), lalu jalankan request pada folder `12. Stock Reconciliation`. Request CREATE
otomatis menyimpan `name` hasil submit ke variabel `stock_reco_id` lewat test script.
