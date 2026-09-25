# PRD — REST API Doctype Stock Entry (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Stock Entry** — dokumen pergerakan stok
> (transfer antar gudang, pengeluaran/pemakaian, penerimaan, dsb.) — beserta
> **Material Request** sebagai dokumen **pengajuan/permintaan transfer** — diperuntukkan bagi
> tim **UI/Frontend**.

- **Modul:** Stock — path `erpnext.stock.doctype.stock_entry` (+ `erpnext.stock.doctype.material_request`)
- **Doctype:** `Stock Entry` (submittable) + `Material Request` (submittable) + `Stock Entry Detail` (child)
- **Versi API:** `/api/method/...` (API v1) — method whitelisted `frappe.client.*` & method ERPNext,
  parameter dikirim di **body** JSON
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `http://erpnext.localhost:8000`)
- **Status verifikasi:** seluruh contoh, pesan error, dan angka pada dokumen ini **diuji langsung**
  pada site dev ERPNext v16 (`erpnext.localhost`) pada 2026-09-24. Tanda ✅ = terverifikasi.

> **Konvensi pemanggilan (penting):** seluruh operasi memakai method whitelisted dengan
> parameter di **body JSON** (`frappe.client.insert`, `frappe.client.submit`, `run_doc_method`,
> method ERPNext seperti `make_stock_entry`, dst.),
> **bukan** `/api/resource/{doctype}/{name}`. Alasannya sama seperti PRD lain (nama dokumen/warehouse
> bisa mengandung spasi & karakter khusus) dan karena Stock Entry memerlukan pemanggilan method
> ERPNext (`make_stock_entry`, `make_in_transit_stock_entry`, `make_stock_in_entry`).
> **Format respons:** method `/api/method/...` membungkus hasil di **`"message"`**.

> **Perbedaan utama dengan doctype master (Item / Item Group / Warehouse):**
> - Stock Entry adalah **dokumen transaksi** (submittable): ada siklus **draft → submit → cancel**.
>   Tidak ada field `disabled`; "batal" = **submit** (stok bergerak) atau **cancel** (stok kembali).
> - Stok **tidak berubah saat draft**. Perubahan stok terjadi **saat submit** (`Stock Ledger Entry`).
> - Nama dokumen memakai **naming series** `MAT-STE-.YYYY.-` (contoh: `MAT-STE-2026-00001`),
>   **bukan** nilai yang dikirim frontend.
> - Setiap dokumen **wajib** punya `stock_entry_type` + minimal 1 baris `items`, dan posting
>   memerlukan **Fiscal Year aktif** yang mencakup `posting_date`.

---

## 1. Ruang lingkup

| # | Endpoint (POST `/api/method/...`) | Operasi | Catatan |
|---|---|---|---|
| 1 | `frappe.client.insert` | Buat Stock Entry (CREATE, `docstatus=0`) | draft; stok belum bergerak |
| 2 | `frappe.client.submit` | **Submit** draft (`docstatus=1`) | stok bergerak + jurnal (GL) |
| 3 | `frappe.client.cancel` | **Cancel** dokumen submitted (`docstatus=2`) | stok dikembalikan |
| 4 | `frappe.client.get` | Ambil detail 1 dokumen (READ) | `name` di body |
| 5 | `frappe.client.get_list` | Daftar Stock Entry (READ list) | filter `purpose`, `docstatus`, `posting_date`, dst. |
| 6 | `frappe.client.get_count` | Total record sesuai filter | untuk pagination |
| 7 | `frappe.client.get_value` | Ambil 1–beberapa field | mis. `per_transferred`, `docstatus` |
| 8 | `frappe.client.save` | Ubah dokumen **draft** (UPDATE) | `items` replace-all |
| 9 | `frappe.client.set_value` | Ubah field tunggal **draft** | mis. `posting_date`, `remarks` |
| 10 | `frappe.client.delete` | Hapus **draft** | submitted wajib cancel dulu (ditolak) |
| 11 | `frappe.client.insert` + `submit` (doctype `Material Request`) | **Tahap 1 — pengajuan transfer** | §6.1 |
| 12 | `erpnext.stock.doctype.material_request.material_request.make_stock_entry` | MR → **draft** Stock Entry (transfer **langsung** sumber→tujuan) | §6.2 varian A |
| 13 | `erpnext.stock.doctype.material_request.material_request.make_in_transit_stock_entry` | MR → **draft** Stock Entry **keluar ke gudang Transit** (`add_to_transit=1`) | **Tahap 2** — §6.2 |
| 14 | `erpnext.stock.doctype.stock_entry.stock_entry.make_stock_in_entry` | SE keluar → **draft** Stock Entry **penerimaan** (Transit → tujuan) | **Tahap 3** — §6.3 |
| 15 | `run_doc_method` (`method: "set_items_for_stock_in"`) | Isi ulang baris penerimaan dari dokumen keluar (manual) | §5.6 |
| 16 | `run_doc_method` (`method: "get_item_details"`) | Autofill `uom`/`conversion_factor`/`expense_account` 1 baris | §5.7 |
| 17 | `frappe.client.get_list` / `get` | GET pendukung UI: Stock Entry Type, Warehouse, Item, Bin, Company, Cost Center, Account, Serial No, Batch, Stock Settings | §5 |

> **Alur 3 tahap (yang diminta):** **1) pengajuan** = `Material Request` (material_request_type
> `Material Transfer`) → **2) pengeluaran** = Stock Entry `Material Transfer` `add_to_transit=1`
> (sumber → **Transit**) → **3) penerimaan** = Stock Entry `Material Transfer` dengan
> `outgoing_stock_entry` (Transit → gudang tujuan). Studi kasus lengkap di **§6**.

> Dokumen ini hanya membahas **data wajib terisi + field penting** untuk alur transfer/pemakaian
> gudang. Purpose manufaktur (Manufacture, Repack, Subcontracting, Disassemble) tidak dibahas
> detail (butuh BOM/Work Order). Export/import masal tidak dipakai di aplikasi ini.

---

## 2. Ringkasan field & data

| Status | Field (parent `Stock Entry`) | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `stock_entry_type` | Link → Stock Entry Type (`reqd: 1`) | Jenis entri. `purpose` **ikut otomatis** dari type ini. Tersedia 13 type standar (§2.1 no. 2). |
| 🔴 **WAJIB** | `company` | Link → Company | Perusahaan; wajib sama dengan `company` gudang/akun. |
| 🔴 **WAJIB** | `items` | Table → Stock Entry Detail | Minimal 1 baris (lihat §2.2). |
| ⚪ Otomatis | `naming_series` | Select (`reqd`, `set_only_once`) | Satu opsi: `MAT-STE-.YYYY.-` → `name` dibuat sistem (`MAT-STE-2026-00001`). |
| ⚪ Otomatis | `purpose` | Select (`read_only`, `fetch_from: stock_entry_type.purpose`) | Terisi dari `stock_entry_type`; **jangan dikirim**. |
| ⚪ Otomatis | `amended_from` | Link → Stock Entry | Terisi bila dokumen hasil amend. |
| ⚪ Read-only | `total_outgoing_value`, `total_incoming_value`, `value_difference`, `total_amount`, `total_additional_costs` | Currency | Dihitung sistem dari baris. |
| ⚪ Read-only | `per_transferred` | Percent | Untuk entri transit: % qty yang sudah diterima di tahap 3. |
| 🟠 **DISARANKAN** | `posting_date` (+ `posting_time`) | Date/Time | Default **hari ini**. Untuk mengisi tanggal lampau **wajib** set `set_posting_time=1` (lihat §2.1 no. 3). |
| 🟠 **DISARANKAN** | `from_warehouse` | Link → Warehouse | Gudang **asal** default (header). Baris bisa menimpa lewat `s_warehouse`. |
| 🟠 **DISARANKAN** | `to_warehouse` | Link → Warehouse | Gudang **tujuan** default (header). Baris bisa menimpa lewat `t_warehouse`. |
| 🟠 | `add_to_transit` | Check | `1` = entri keluar **2 tahap** (barang berhenti di gudang bertipe **Transit**). Default 0. |
| 🟠 | `outgoing_stock_entry` | Link → Stock Entry (`read_only`) | Pada dokumen penerimaan: nama dokumen keluar. |
| 🟠 | `remarks` | Text | Keterangan internal (alasan transfer, no. surat jalan, dsb.). |
| 🟠 | `project`, `cost_center` | Link | Dimensi akuntansi (opsional; `cost_center` baris terisi otomatis). |
| 🟠 | `additional_costs` | Table → Landed Cost Taxes and Charges | Biaya tambahan (ongkos kirim) yang menambah nilai stok masuk. |
| 🟠 | `apply_putaway_rule` | Check | Isi otomatis `t_warehouse` baris dari Putaway Rule. |
| 🟠 | `supplier`, `supplier_address`, `purchase_order`, `sales_invoice_no`, `delivery_note_no`, `pick_list`, `work_order`, `bom_no`, `fg_completed_qty`, `job_card`, `source_stock_entry`, `subcontracting_order` | Link/Data/Float | Referensi dokumen sumber — hanya untuk purpose terkait (Subcontracting/Manufacture/Return). |
| ⚪ Read-only | `is_opening`, `is_return` | Select/Check | Penanda khusus (opening stock / retur). |
| ✖️ **Tidak ada** | `disabled` | — | Stock Entry tidak punya soft-delete. Batal = **cancel**. |

### 2.1 Catatan penting

1. **Stok hanya berubah saat SUBMIT.** Draft boleh dibuat/diubah/dihapus tanpa efek stok.
   Setelah submit, baris **tidak bisa** diubah bebas (mis. `UpdateAfterSubmitError` untuk
   `serial_no`), koreksi dilakukan via **cancel** (atau amend).
2. **`purpose` mengikuti `stock_entry_type`.** Cukup kirim `stock_entry_type`; ERPNext mengisi
   `purpose` (`set_purpose_for_stock_entry()`). Daftar purpose (13, urutan persis opsi doctype):
   `Material Issue`, `Material Receipt`, `Material Transfer`, `Material Transfer for Manufacture`,
   `Material Consumption for Manufacture`, `Manufacture`, `Repack`, `Send to Subcontractor`,
   `Disassemble`, `Receive from Customer`, `Return Raw Material to Customer`,
   `Subcontracting Delivery`, `Subcontracting Return`.
   Stock Entry Type standar (`is_standard=1`) yang relevan untuk aplikasi ini:
   `Material Issue`, `Material Receipt`, `Material Transfer` (semua dengan `add_to_transit=0`).
3. **Tanggal posting & Fiscal Year (WAJIB diperhatikan).**
   - `posting_date` diisi otomatis **hari ini** kecuali frontend mengirim `set_posting_time=1`
     **bersamaan** dengan `posting_date` (dan opsional `posting_time`).
   - Posting memerlukan **Fiscal Year aktif** yang mencakup `posting_date` + `company`; jika tidak:
     `417 FiscalYearError: Transaction Date 24-09-2026 is not in any active Fiscal Year for <company>` ✅.
   - Site dev saat ini hanya punya **Fiscal Year 2020** ⇒ semua contoh di dokumen ini memakai
     `set_posting_time: 1` + `posting_date: "2020-12-01"`. Di produksi pakai tanggal nyata
     (pastikan Fiscal Year tahun berjalan sudah dibuat).
4. **Wajib satu gudang asal / tujuan sesuai purpose** (`validate_warehouse`):

   | Perlu | Purpose |
   |---|---|
   | **Gudang asal** (`s_warehouse`/`from_warehouse`) | `Material Issue`, `Material Transfer`, `Material Transfer for Manufacture`, `Material Consumption for Manufacture`, `Send to Subcontractor`, `Return Raw Material to Customer`, `Subcontracting Delivery` |
   | **Gudang tujuan** (`t_warehouse`/`to_warehouse`) | `Material Receipt`, `Material Transfer`, `Material Transfer for Manufacture`, `Send to Subcontractor`, `Receive from Customer`, `Subcontracting Return` |

   Bila salah satu kosong → `417 ValidationError: Target warehouse is mandatory for row 1`
   / `Source warehouse is mandatory for row 1` ✅.
5. **Gudang asal = tujuan.** Untuk purpose `Material Transfer` hal ini **diizinkan** secara default
   (`Stock Settings.validate_material_transfer_warehouses = 0` di site dev ✅). Bila ingin diblokir,
   aktifkan setting tersebut → error: *"Row #1: Source and Target Warehouse cannot be the same for
   Material Transfer"*. Untuk purpose lain, gudang sama selalu ditolak
   (*"Source and target warehouse cannot be same for row 0"*).
6. **Role (DocPerm v16):** `Stock User`, `Stock Manager`, `Manufacturing User`, `Manufacturing Manager`
   punya read/write/create/submit/cancel/delete/amend; System Manager bypass. Untuk `Material Request`:
   `Stock User`, `Stock Manager`, `Purchase User`, `Purchase Manager`. Role tanpa hak → `403 Not permitted`.
7. **Item wajib `is_stock_item=1`** (barang non-stok seperti Product Bundle tidak boleh) →
   `417 ValidationError: <item> is not a stock Item`. Qty wajib > 0 →
   *"Row 1: The item X, quantity must be positive number"*. UOM tanpa faktor konversi →
   *"Row 1: UOM Conversion Factor is mandatory"*.
8. **Serial/Batch** dibahas terpisah di **§2.4** (perlu setting Stock Settings + contoh lengkap di §6.7).
9. **Transit warehouse** (`warehouse_type = "Transit"`) **tidak dibuat otomatis** oleh ERPNext.
   Untuk transfer 2 tahap, buat dulu: Warehouse Type `Transit` (sudah ada di site dev ✅) +
   Warehouse baru bertipe Transit (lihat §5.2). Di site dev: `Goods In Transit - APSD`.
10. **Cancel** mengembalikan stok, tetapi **diblokir** bila stok hasil transfer sudah terpakai
    (guard stok negatif) → `417 NegativeStockError: 5.0 units of Item SKU010 needed in Warehouse
    Store Semarang - APSD to complete this transaction.` ✅ (lihat §6.8).
11. **`frappe.client.submit`** harus dikirim dengan **dokumen lengkap** hasil CREATE/GET.
    Mengirim hanya `{"doctype": "...", "name": "..."}` → `417 TimestampMismatchError` ✅ (§4.2).

### 2.2 `items` — child table `Stock Entry Detail`

| Field | Tipe | Keterangan |
|---|---|---|
| `item_code` | Link → Item (`reqd`) | Barang yang dipindah. |
| `qty` | Float (`reqd`) | Jumlah dalam `uom` baris. |
| `uom` | Link → UOM | UOM transaksi; terisi otomatis dari `stock_uom` bila kosong. |
| `conversion_factor` | Float | Faktor konversi UOM → stock UOM (default 1). |
| `stock_uom`, `transfer_qty` | Link/Float (read-only) | Diisi sistem; `transfer_qty = qty × conversion_factor`. |
| `s_warehouse` | Link → Warehouse | Gudang asal baris (kosongkan bila purpose hanya masuk). |
| `t_warehouse` | Link → Warehouse | Gudang tujuan baris (kosongkan bila purpose hanya keluar). |
| `basic_rate` | Currency | Nilai per unit. **Kosongkan** agar diambil otomatis dari nilai stok asal (FIFO) — pada transfer nilainya dihitung sistem. Untuk `Material Receipt` isi rate beli. |
| `basic_amount`, `amount`, `valuation_rate` | Currency (read-only) | Dihitung sistem. |
| `expense_account` | Link → Account | **Material Issue**: akun beban/penyesuaian. Bila kosong, otomatis diisi (site dev: `5110.020 - Penyesuaian Stock - APSD`) ✅. |
| `cost_center` | Link → Cost Center | Terisi otomatis dari company/item (`Main - APSD`) ✅. |
| `batch_no` | Link → Batch | Batch (barang ber-`has_batch_no`) — batch **harus sudah ada** (§2.4). |
| `serial_no` | Small Text | Daftar Serial No **dipisah newline** (`"SN-1\nSN-2"`). |
| `serial_and_batch_bundle` | Link → Serial and Batch Bundle | Diisi otomatis sistem saat insert/submit. |
| `allow_zero_valuation_rate` | Check | Untuk barang tanpa nilai (mis. gratisan). |
| `material_request`, `material_request_item` | Link | Jejak dokumen pengajuan (terisi otomatis dari mapper MR). |
| `against_stock_entry`, `ste_detail` | Link | Pada dokumen penerimaan: dokumen keluar + barisnya. |
| `transferred_qty`, `per_transferred` | Float/Percent | Progres penerimaan (dihitung sistem). |
| `project`, `cost_center` | Link | Dimensi akuntansi per baris. |
| `is_finished_item`, `type`, `bom_no`, `sco_rm_detail`, `scio_detail`, `job_card_item` | — | Khusus Manufacture/Subcontracting (tidak dipakai alur transfer). |

> **Perilaku child table:** `frappe.client.save` memperlakukan `items` sebagai **replace-all** —
> kirim seluruh baris yang diinginkan. Untuk **submit**, kirim dokumen hasil CREATE/GET apa adanya
> (kecuali kasus serial/batch — lihat §2.4 no. 3).

### 2.3 `Material Request` — dokumen pengajuan transfer (dokumen pendamping)

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 | `material_request_type` | Select (`reqd`) | Opsi: `Purchase`, **`Material Transfer`**, `Material Issue`, `Manufacture`, `Subcontracting`, `Customer Provided`. |
| 🔴 | `company` | Link → Company (`reqd`) | Company transaksi. |
| 🔴 | `transaction_date` | Date (`reqd`, default **Today**) | Tanggal pengajuan. |
| 🔴 | `items` | Table → Material Request Item (`reqd`) | Minimal 1 baris. |
| ⚪ | `naming_series` | Select (`reqd`) | Satu opsi `MAT-MR-.YYYY.-` → `MAT-MR-2026-00001`. |
| 🟠 | `schedule_date` | Date | Target tanggal dibutuhkan. |
| 🟠 | `set_from_warehouse` / `set_warehouse` | Link → Warehouse | Gudang asal / tujuan **default**. ⚠️ Di-null-kan otomatis bila tiap baris sudah punya `from_warehouse`/`warehouse` ✅ (§6.1 catatan). |
| ⚪ Read-only | `status` | Select | `Draft` → `Pending` (setelah submit) → `Transferred` (setelah pengeluaran) / `Cancelled` / dst. |
| ⚪ Read-only | `transfer_status` | Select | `""`/`Not Started` → **`In Transit`** (setelah tahap 2) → **`Completed`** (setelah tahap 3). ✅ |
| ⚪ Read-only | `per_ordered`, `per_received` | Percent | `per_ordered` = 100 setelah Stock Entry dibuat & disubmit. ✅ |

**Material Request Item:** `item_code` (reqd), `qty` (reqd), `uom` (reqd), `conversion_factor` (reqd),
`stock_uom` (reqd), `schedule_date` (reqd), **`warehouse`** (Link) = gudang **tujuan**,
**`from_warehouse`** (Link) = gudang **asal** (khusus `Material Transfer`), `ordered_qty`/`stock_qty` (read-only).

> **Pemetaan MR → Stock Entry** (`make_stock_entry`): untuk `Material Transfer`, `s_warehouse` diambil
> dari `Material Request Item.from_warehouse` dan `t_warehouse` dari `Material Request Item.warehouse` ✅.

### 2.4 Batch & Serial No (dan setting yang dibutuhkan)

1. **Prasyarat site (Stock Settings):**

   | Setting | Fungsi |
   |---|---|
   | `enable_serial_and_batch_no_for_item` ("Activate Serial / Batch No for Item") | **Harus 1** agar field `serial_no`/`batch_no` bisa dipakai & sistem membuat *Serial and Batch Bundle*. Bila 0 → submit ditolak: `417 ValidationError: Please check the 'Activate Serial / Batch No for Item' checkbox in the Stock Settings ...` ✅ |
   | `use_serial_batch_fields` | `1` = frontend mengirim `serial_no`/`batch_no` (teks); `0` = hanya `serial_and_batch_bundle`. Site dev: **1** ✅ |

2. **Batch harus sudah ada.** Mengirim `batch_no` yang belum terdaftar → `417 LinkValidationError:
   Could not find Row #1: Batch No: BATCH-AABB-01` ✅. Buat dulu master Batch:
   `POST /api/method/frappe.client.insert` → `{"doc": {"doctype": "Batch", "batch_id": "BATCH-AABB-01",
   "item": "AABB"}}` (batch duplikat → `409 DuplicateEntryError` ✅). Item ber-`has_expiry_date=1`
   mendapat `expiry_date` otomatis dari shelf life.
3. **Serial** tidak perlu dibuat manual: kirim `serial_no` bertipe teks dipisah newline, sistem membuat
   dokumen `Serial No` + `Serial and Batch Bundle` saat insert/submit ✅.
   ⚠️ **Pitfall terverifikasi:** submit (via GET) dokumen yang barisnya **masih** berisi `serial_no`
   **dan** sudah punya `serial_and_batch_bundle` → `417 ValidationError: At row 1: Serial and Batch
   Bundle <hash> has already created. Please remove the values from the serial no or batch no fields.` ✅
   **Solusi (dua-duanya terbukti 200 ✅):** (a) submit memakai **respons CREATE apa adanya**; atau
   (b) bila dokumen diambil ulang via GET, **kosongkan `serial_no` + `batch_no`** pada baris sebelum submit.
4. **Barang ber-`has_batch_no=1` wajib isi batch** saat keluar/masuk:
   `417 ValidationError: At row 1: Batch No is mandatory for Item AABB` ✅.
5. **Jumlah & ketersediaan divalidasi:** qty > stok tersedia di gudang asal →
   `417 ValidationError: For the item AABB, the Available qty 2.0 is less than the Required Qty 5.0 in
   the warehouse ...` ✅.

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Detail lengkap setup OAuth Client ada di **[`prd_oauth.md`](../prd_oauth.md)**.
Alternatif untuk pengujian/manual: API key (`Authorization: token <api_key>:<api_secret>`).

---

## 4. CRUD & siklus hidup Stock Entry

### 4.1 CREATE (draft) — `frappe.client.insert`

**Payload minimum (data wajib):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Stock Entry",
      "stock_entry_type": "Material Transfer",
      "company": "PT Rapupa Guna Teknologi (Demo)",
      "set_posting_time": 1,
      "posting_date": "2020-12-01",
      "from_warehouse": "Finished Goods - APSD",
      "to_warehouse": "Store Semarang - APSD",
      "items": [
        { "item_code": "SKU010", "qty": 3, "uom": "Nos", "conversion_factor": 1 }
      ]
    }
  }'
```

> `purpose` tidak dikirim (ikut `stock_entry_type`). `s_warehouse`/`t_warehouse` baris terisi dari
> `from_warehouse`/`to_warehouse` header bila kosong. Tanpa `set_posting_time`, `posting_date`
> **dipaksa hari ini**.

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": {
    "name": "MAT-STE-2026-00003",
    "docstatus": 0,
    "stock_entry_type": "Material Transfer",
    "purpose": "Material Transfer",
    "company": "PT Rapupa Guna Teknologi (Demo)",
    "posting_date": "2020-12-01",
    "set_posting_time": 1,
    "from_warehouse": "Finished Goods - APSD",
    "to_warehouse": "Store Semarang - APSD",
    "total_incoming_value": 1500.0,
    "total_outgoing_value": 1500.0,
    "value_difference": 0.0,
    "per_transferred": 0.0,
    "items": [
      {
        "idx": 1,
        "item_code": "SKU010",
        "qty": 3.0,
        "uom": "Nos",
        "conversion_factor": 1.0,
        "stock_uom": "Nos",
        "transfer_qty": 3.0,
        "s_warehouse": "Finished Goods - APSD",
        "t_warehouse": "Store Semarang - APSD",
        "basic_rate": 500.0,
        "basic_amount": 1500.0,
        "cost_center": "Main - APSD",
        "expense_account": null
      }
    ]
  }
}
```

> Simpan `name` dari respons → dipakai untuk SUBMIT/CANCEL/GET. `basic_rate` pada transfer terisi
> otomatis dari **nilai stok asal** (FIFO) ✅.

**Varian payload per purpose (cukup dijadikan nilai `doc`):**

```json
// A. Material Transfer — keluar & masuk dalam SATU dokumen (sumber -> tujuan)
{
  "doctype": "Stock Entry",
  "stock_entry_type": "Material Transfer",
  "company": "PT Rapupa Guna Teknologi (Demo)",
  "set_posting_time": 1, "posting_date": "2020-12-01",
  "from_warehouse": "Finished Goods - APSD",
  "to_warehouse": "Store Semarang - APSD",
  "remarks": "Transfer antar gudang",
  "items": [
    { "item_code": "SKU010", "qty": 3, "uom": "Nos", "conversion_factor": 1 },
    { "item_code": "SKU008", "qty": 5, "uom": "Nos", "conversion_factor": 1,
      "s_warehouse": "Work In Progress - APSD" }
  ]
}
```

```json
// B. Material Transfer 2 TAHAP — keluar dulu ke gudang Transit (dipakai pada tahap 2 §6.2)
{
  "doctype": "Stock Entry",
  "stock_entry_type": "Material Transfer",
  "company": "PT Rapupa Guna Teknologi (Demo)",
  "add_to_transit": 1,
  "set_posting_time": 1, "posting_date": "2020-12-01",
  "from_warehouse": "Finished Goods - APSD",
  "to_warehouse": "Goods In Transit - APSD",
  "items": [
    { "item_code": "SKU010", "qty": 10, "uom": "Nos", "conversion_factor": 1,
      "s_warehouse": "Finished Goods - APSD", "t_warehouse": "Goods In Transit - APSD" }
  ]
}
```

```json
// C. Material Receipt — barang masuk tanpa gudang asal (mis. hasil temuan/pembelian tunai)
{
  "doctype": "Stock Entry",
  "stock_entry_type": "Material Receipt",
  "company": "PT Rapupa Guna Teknologi (Demo)",
  "set_posting_time": 1, "posting_date": "2020-12-01",
  "to_warehouse": "Gudang 2 - APSD",
  "items": [
    { "item_code": "SKU010", "qty": 10, "uom": "Nos", "conversion_factor": 1, "basic_rate": 500 }
  ]
}
```

```json
// D. Material Issue — barang keluar TANPA tujuan (pemakaian internal, rusak, hilang)
{
  "doctype": "Stock Entry",
  "stock_entry_type": "Material Issue",
  "company": "PT Rapupa Guna Teknologi (Demo)",
  "set_posting_time": 1, "posting_date": "2020-12-01",
  "from_warehouse": "Store Semarang - APSD",
  "items": [
    { "item_code": "SKU010", "qty": 2, "uom": "Nos", "conversion_factor": 1 }
  ]
}
```

> **Material Issue tanpa `expense_account` tetap sukses** ✅ — sistem mengisi akun dari setelan
> item/company (site dev: `5110.020 - Penyesuaian Stock - APSD`). Kirim `expense_account` +
> `cost_center` eksplisit hanya bila ingin akun beban tertentu.
> Beda dengan **Stock Reconciliation** (lihat §4.6).

### 4.2 SUBMIT (draft → submitted) — `frappe.client.submit`

Stok **baru bergerak** pada langkah ini: dibuat `Stock Ledger Entry` (+ jurnal GL bila perpetual
inventory aktif).

**Pola yang benar** — kirim **dokumen lengkap** (hasil CREATE atau hasil GET):

```bash
# Langkah 1 — ambil dokumen utuh (bila tidak menyimpan respons CREATE)
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Stock Entry", "name": "MAT-STE-2026-00003" }'

# Langkah 2 — submit memakai objek message dari langkah 1 (dokumen utuh, termasuk modified)
curl -X POST https://site-anda.com/api/method/frappe.client.submit \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doc": { "doctype": "Stock Entry", "name": "MAT-STE-2026-00003", ... } }'
```

**Contoh respons sukses (HTTP 200):** `message` = dokumen dengan `"docstatus": 1`.

> **Catatan penting (terverifikasi):**
> - ❌ Kirim **hanya** `{"doctype": "Stock Entry", "name": "..."}` → `417 TimestampMismatchError:
>   Error: MAT-MR-2026-00001 (Material Request) has been modified after you have opened it ...` ✅
>   (dokumen tidak lengkap dianggap salinan basi).
> - ✅ Submit ulang dokumen yang sudah submitted **tidak error** (HTTP 200, `docstatus` tetap 1) —
>   jadikan `docstatus` sebagai guard di frontend agar tidak dobel.
> - Untuk baris **serial/batch**: submit dengan respons CREATE apa adanya, atau (bila ambil ulang
>   via GET) **kosongkan `serial_no`+`batch_no`** dulu; kalau tidak → `417 ... Serial and Batch Bundle
>   <hash> has already created.` ✅ (§2.4 no. 3).

### 4.3 READ — `frappe.client.get`, `get_value`, `get_list`, `get_count`

```bash
# Detail 1 dokumen (termasuk seluruh baris items)
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Stock Entry", "name": "MAT-STE-2026-00003" }'

# Beberapa field saja (mis. progres transfer)
curl -X POST https://site-anda.com/api/method/frappe.client.get_value \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Stock Entry", "filters": { "name": "MAT-STE-2026-00003" },
        "fieldname": ["docstatus", "per_transferred", "total_outgoing_value"] }'

# Daftar — transfer antar gudang yang sudah submitted, urut terbaru
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Entry",
    "fields": ["name","posting_date","stock_entry_type","from_warehouse","to_warehouse","total_outgoing_value","docstatus"],
    "filters": [["purpose","=","Material Transfer"],["docstatus","=",1]],
    "order_by": "posting_date desc, name desc",
    "limit_start": 0,
    "limit_page_length": 50
  }'

# Total record (pagination) — Stock Entry tidak punya field disabled
curl -X POST https://site-anda.com/api/method/frappe.client.get_count \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Stock Entry", "filters": [["purpose","=","Material Transfer"]] }'
```

**Filter yang berguna untuk UI riwayat transfer:**

| Filter | Kegunaan |
|---|---|
| `["purpose","=","Material Transfer"]` | hanya transfer antar gudang |
| `["docstatus","=",0]` / `1` / `2` | draft (menunggu) / selesai / dibatalkan |
| `["from_warehouse","=","<wh>"]`, `["to_warehouse","=","<wh>"]` | per gudang (header) |
| `["outgoing_stock_entry","=","<name>"]` | dokumen penerimaan dari 1 dokumen keluar (transit) |
| `["per_transferred","<",100]` | entri transit yang **belum** diterima penuh |
| `["posting_date","between",["2020-12-01","2020-12-31"]]` | periode |

### 4.4 UPDATE — `frappe.client.save` / `frappe.client.set_value` (khusus DRAFT)

```bash
# Ubah field tunggal pada draft (aman untuk 1–2 field)
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Entry",
    "name": "MAT-STE-2026-00003",
    "fieldname": { "remarks": "Koreksi: surat jalan SJ-001", "posting_date": "2020-12-02" }
  }'

# Ubah baris items (replace-all) — kirim dokumen utuh hasil GET yang dimodifikasi
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doc": { "doctype": "Stock Entry", "name": "MAT-STE-2026-00003", "modified": "...", "items": [ ... ] } }'
```

> - `save` membangun ulang dokumen dari dict → **wajib** menyertakan `name`, `modified` (dan field
>   lain yang tidak boleh hilang) agar tidak kena `TimestampMismatchError`/nilai ter-reset.
>   `items` berlaku **replace-all**.
> - Setelah submitted, field tertentu **tidak boleh** diubah →
>   `417 UpdateAfterSubmitError: Row #1: Not allowed to change Serial No after submission from SN-AABB-001` ✅.
>   Untuk koreksi gunakan **cancel** (atau cancel + amend).

### 4.5 CANCEL, amend, dan hapus

```bash
# Cancel dokumen submitted (stok dikembalikan) — butuh doctype + name
curl -X POST https://site-anda.com/api/method/frappe.client.cancel \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Stock Entry", "name": "MAT-STE-2026-00003" }'

# Hapus DRAFT
curl -X POST https://site-anda.com/api/method/frappe.client.delete \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Stock Entry", "name": "MAT-STE-2026-00002" }'
```

- ✅ Cancel sukses → `docstatus = 2`, stok kembali ke posisi sebelum submit (dibuat SLE pembalik).
- ✅ Hapus dokumen **submitted** ditolak: `417 ValidationError: Stock Entry MAT-STE-2026-00002:
  Submitted Record cannot be deleted. You must Cancel it first.`
- ✅ Cancel **diblokir** bila stok hasil transfer sudah terpakai (lihat §6.8) →
  `417 NegativeStockError: ... needed in Warehouse <wh> to complete this transaction.`
- **Amend** (untuk revisi dokumen yang sudah di-cancel/bermasalah): buat dokumen baru dengan
  `"amended_from": "<nama dokumen lama>"` (field `amended_from` read-only pada dokumen lama).

### 4.6 Material Issue vs Stock Reconciliation (stok opname)

Dua-duanya "mengurangi stok tanpa gudang tujuan", tetapi **maksudnya berbeda**:

| Aspek | **Stock Entry — `Material Issue`** | **Stock Reconciliation** (stok opname) |
|---|---|---|
| Sifat | **Pengeluaran nyata tercatat**: barang keluar karena dipakai/rusak/hilang, dengan alasan & PIC. | **Koreksi saldo**: menyelaraskan **angka sistem** dengan hasil hitung fisik. |
| Arah | Hanya **keluar** (satu arah, `from_warehouse`). | **Naik & turun** (mengisi `qty`/`valuation_rate` target; selisih dihitung sistem). |
| Nilai stok | Memakai nilai stok berjalan (FIFO) → beban di akun `expense_account`. | Bisa **menetapkan nilai baru** (`valuation_rate`), khas untuk koreksi nilai/saldo awal. |
| Dasar keputusan | Dokumen permintaan (mis. berita acara pemusnahan) | Berita acara **penghitungan fisik** periodik (mis. akhir bulan). |
| Field penanda | `stock_entry_type = "Material Issue"`, `items[].expense_account` | `purpose = "Stock Reconciliation"`, `items[].qty`/`valuation_rate`, akun selisih di header/baris |
| Kapan dipakai | Ada pemakaian/kerusakan/kehilangan yang **diketahui penyebabnya** | Saldo sistem ≠ fisik dan **penyebab tidak diketahui** (atau perlu revaluasi) |
| Dokumen PRD | **dokumen ini** (§4.1 varian D, §6.6) | **[`prd_stock_reconciliation.md`](./prd_stock_reconciliation.md)** |

> **Aturan praktis untuk UI:** bila user memilih "barang keluar/rusak/hilang" dengan alasan →
> **Material Issue**. Bila user memilih "opname / koreksi saldo gudang" (input stok fisik hasil
> hitung) → **Stock Reconciliation**. Jangan memakai opname untuk pemakaian rutin (jejak & akun
> bebannya tidak terbentuk), dan jangan memakai Material Issue untuk menaikkan stok.

---

## 5. GET pendukung UI

### 5.1 GET Stock Entry Type (dropdown utama dokumen) — `frappe.client.get_list`

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Stock Entry Type",
    "fields": ["name","purpose","add_to_transit"],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200)** — 13 type standar; `add_to_transit` semuanya `0` di site dev ✅:

```json
{
  "message": [
    { "name": "Disassemble", "purpose": "Disassemble", "add_to_transit": 0 },
    { "name": "Manufacture", "purpose": "Manufacture", "add_to_transit": 0 },
    { "name": "Material Consumption for Manufacture", "purpose": "Material Consumption for Manufacture", "add_to_transit": 0 },
    { "name": "Material Issue", "purpose": "Material Issue", "add_to_transit": 0 },
    { "name": "Material Receipt", "purpose": "Material Receipt", "add_to_transit": 0 },
    { "name": "Material Transfer", "purpose": "Material Transfer", "add_to_transit": 0 },
    { "name": "Material Transfer for Manufacture", "purpose": "Material Transfer for Manufacture", "add_to_transit": 0 },
    { "name": "Receive from Customer", "purpose": "Receive from Customer", "add_to_transit": 0 },
    { "name": "Repack", "purpose": "Repack", "add_to_transit": 0 },
    { "name": "Return Raw Material to Customer", "purpose": "Return Raw Material to Customer", "add_to_transit": 0 },
    { "name": "Send to Subcontractor", "purpose": "Send to Subcontractor", "add_to_transit": 0 },
    { "name": "Subcontracting Delivery", "purpose": "Subcontracting Delivery", "add_to_transit": 0 },
    { "name": "Subcontracting Return", "purpose": "Subcontracting Return", "add_to_transit": 0 }
  ]
}
```

> **Untuk transfer 2 tahap**, frontend **tidak** perlu Stock Entry Type khusus: kirim
> `stock_entry_type = "Material Transfer"` + `add_to_transit: 1` (lihat §6.2). Type Transit khusus
> hanya perlu bila ingin membuat tombol/opsi terpisah — buat Stock Entry Type baru
> (`add_to_transit = 1`, `is_standard = 0`) via `frappe.client.insert` doctype `Stock Entry Type`.

### 5.2 GET Warehouse (asal/tujuan) + gudang Transit

```bash
# Semua gudang leaf (boleh dipakai transaksi) untuk 1 company
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "fields": ["name","warehouse_type","is_group","default_in_transit_warehouse"],
    "filters": [["company","=","PT Rapupa Guna Teknologi (Demo)"],["is_group","=",0],["disabled","=",0]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'

# Khusus gudang Transit (untuk entri 2 tahap)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "fields": ["name","warehouse_type"],
    "filters": [["company","=","PT Rapupa Guna Teknologi (Demo)"],["is_group","=",0],["disabled","=",0],["warehouse_type","=","Transit"]],
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):** `[ { "name": "Goods In Transit - APSD", "warehouse_type": "Transit" } ]` ✅

> **Bila belum ada gudang Transit** (ERPNext **tidak** membuatnya otomatis):
> 1. Pastikan Warehouse Type `Transit` ada (site dev sudah ada ✅) — jika belum:
>    `POST frappe.client.insert` → `{"doc": {"doctype": "Warehouse Type", "warehouse_type": "Transit"}}`
>    (autoname `Prompt` → kirim `name`= "Transit").
> 2. Buat gudang: `{"doc": {"doctype": "Warehouse", "warehouse_name": "Goods In Transit",
>    "company": "PT Rapupa Guna Teknologi (Demo)", "warehouse_type": "Transit"}}` → name otomatis
>    `Goods In Transit - APSD`.
> 3. Opsional: set default agar frontend tidak perlu memilih —
>    `frappe.client.set_value` doctype `Warehouse` (gudang asal) field `default_in_transit_warehouse`,
>    atau doctype `Company` field yang sama.
> Detail doctype Warehouse: **[`../setup/prd_warehouse.md`](../setup/prd_warehouse.md)**.

### 5.3 GET Item (barang yang bisa dipindah) — `frappe.client.get_list`

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","stock_uom","item_group","has_batch_no","has_serial_no","disabled"],
    "filters": [["is_stock_item","=",1],["disabled","=",0]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

> Tampilkan `has_batch_no`/`has_serial_no` agar UI meminta input batch/serial (§2.4).
> Filter per group daun: `["item_group","=","<leaf>"]` (lihat `prd_item.md` §6.7).

### 5.4 GET stok tersedia per gudang — `frappe.client.get_list` (doctype `Bin`)

Wajib ditampilkan agar user tidak memilih qty melebihi stok (submit akan gagal, §7).

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Bin",
    "fields": ["item_code","warehouse","actual_qty","projected_qty","reserved_qty"],
    "filters": [["warehouse","=","Finished Goods - APSD"],["item_code","=","SKU010"]],
    "limit_page_length": 1
  }'
```

**Contoh respons (HTTP 200):** `{ "message": [ { "item_code": "SKU010", "warehouse": "Finished Goods - APSD", "actual_qty": 78.0, ... } ] }` ✅

> `actual_qty` = saldo fisik; `projected_qty` memperhitungkan dokumen belum selesai. Untuk saldo
> berjalan per tanggal gunakan Stock Ledger / Stock Balance
> (**[`prd_stock_ledger.md`](./prd_stock_ledger.md)**, **`prd_stock_balance.md`**).

### 5.5 GET list Material Request (daftar pengajuan yang siap dieksekusi)

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Material Request",
    "fields": ["name","transaction_date","status","transfer_status","per_ordered","set_from_warehouse","set_warehouse"],
    "filters": [["material_request_type","=","Material Transfer"],["docstatus","=",1],["status","!=","Cancelled"]],
    "order_by": "transaction_date desc",
    "limit_page_length": 50
  }'
```

> Baris MR (gudang asal/tujuan per item) dibaca via `frappe.client.get`:
> `message.items[] → { item_code, qty, uom, warehouse (tujuan), from_warehouse (asal), ordered_qty, stock_qty }` ✅

### 5.6 Isi baris penerimaan otomatis — `run_doc_method` → `set_items_for_stock_in`

Dipakai bila frontend membuat dokumen **penerimaan (tahap 3) secara manual** (bukan lewat
`make_stock_in_entry`): kirim `outgoing_stock_entry` pada draft, lalu panggil method dokumen ini
untuk mengisi baris dari dokumen keluar.

```bash
curl -X POST https://site-anda.com/api/method/run_doc_method \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "method": "set_items_for_stock_in",
    "dt": "Stock Entry",
    "dn": "MAT-STE-2026-00002"
  }'
```

> Respons `message.docs[0].items` berisi baris baru: `s_warehouse` = gudang Transit (dari
> `t_warehouse` dokumen keluar), `against_stock_entry` + `ste_detail` = referensi baris asal,
> **`t_warehouse` masih kosong** → isi gudang tujuan (header `to_warehouse`) lalu submit.
> Bila barang sudah diterima penuh → `417 ValidationError: Goods are already received against the
> outward entry <name>`.

### 5.7 Autofill detail baris — `run_doc_method` → `get_item_details`

Opsional; berguna untuk menampilkan `conversion_factor`, `expense_account`, `cost_center` sebelum
dokumen dibuat (dokumen **in-memory**, tidak tersimpan):

```bash
curl -X POST https://site-anda.com/api/method/run_doc_method \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "method": "get_item_details",
    "docs": {
      "doctype": "Stock Entry",
      "stock_entry_type": "Material Transfer",
      "company": "PT Rapupa Guna Teknologi (Demo)",
      "from_warehouse": "Finished Goods - APSD",
      "to_warehouse": "Store Semarang - APSD",
      "items": [ { "item_code": "SKU010", "qty": 3 } ]
    },
    "args": { "item_code": "SKU010", "company": "PT Rapupa Guna Teknologi (Demo)", "uom": "Nos",
              "s_warehouse": "Finished Goods - APSD", "is_finished_item": 0 }
  }'
```

### 5.8 GET Batch / Serial No / Serial and Batch Bundle

```bash
# Batch milik 1 item (untuk dropdown saat barang ber-has_batch_no=1)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Batch", "fields": ["name","batch_id","item","batch_qty","expiry_date"],
        "filters": [["item","=","AABB"]], "order_by": "creation asc", "limit_page_length": 0 }'

# Serial No yang masih ADA di gudang asal (untuk dropdown pilih serial)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Serial No", "fields": ["name","item_code","batch_no","warehouse","status"],
        "filters": [["item_code","=","AABB"],["warehouse","=","Gudang 2 - APSD"],["status","=","Active"]],
        "limit_page_length": 0 }'

# Bundle yang terbentuk dari sebuah dokumen (audit serial/batch)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Serial and Batch Bundle",
        "fields": ["name","voucher_type","voucher_no","type_of_transaction","total_qty","docstatus"],
        "filters": [["voucher_no","=","MAT-STE-2026-00014"]], "limit_page_length": 0 }'
```

**Contoh respons Serial No setelah transfer 2 dari 3 unit ✅:**

```json
{
  "message": [
    { "name": "SN-AABB-001", "item_code": "AABB", "batch_no": "BATCH-AABB-01", "warehouse": "Finished Goods - APSD", "status": "Active" },
    { "name": "SN-AABB-002", "item_code": "AABB", "batch_no": "BATCH-AABB-01", "warehouse": "Finished Goods - APSD", "status": "Active" },
    { "name": "SN-AABB-003", "item_code": "AABB", "batch_no": "BATCH-AABB-01", "warehouse": "Gudang 2 - APSD", "status": "Active" }
  ]
}
```

> Dokumen `Serial No` **dibuat otomatis** dari teks `serial_no` (tidak perlu dibuat manual) ✅.
> Satu dokumen transfer menghasilkan 2 bundle (Outward dari gudang asal + Inward ke gudang tujuan) ✅.

### 5.9 GET Stock Settings (prasyarat serial/batch & validasi)

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_value \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Stock Settings", "filters": { "name": "Stock Settings" },
        "fieldname": ["enable_serial_and_batch_no_for_item","use_serial_batch_fields",
                      "allow_negative_stock","validate_material_transfer_warehouses"] }'
```

Site dev ✅: `enable_serial_and_batch_no_for_item = 1` (setelah diaktifkan),
`use_serial_batch_fields = 1`, `allow_negative_stock = 0`, `validate_material_transfer_warehouses = 0`.

### 5.10 Dropdown lain (Company, Cost Center, Account, Project)

```bash
# Cost Center leaf per company (untuk field cost_center)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Cost Center", "fields": ["name"],
        "filters": [["company","=","PT Rapupa Guna Teknologi (Demo)"],["is_group","=",0],["disabled","=",0]],
        "limit_page_length": 0 }'

# Account (untuk expense_account Material Issue — umumnya tidak perlu, otomatis)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Account", "fields": ["name","account_type"],
        "filters": [["company","=","PT Rapupa Guna Teknologi (Demo)"],["is_group","=",0]],
        "limit_page_length": 0 }'
```

---

## 6. Contoh kasus: transfer beberapa produk antar gudang (3 tahap)

> Semua langkah di bawah **terverifikasi** pada site dev (ERPNext v16) dengan data:
> company **`PT Rapupa Guna Teknologi (Demo)`** (abbr `APSD`), gudang asal
> **`Finished Goods - APSD`** (+ `Work In Progress - APSD`), gudang tujuan
> **`Store Semarang - APSD`**, gudang transit **`Goods In Transit - APSD`**,
> produk **`SKU010`** dan **`SKU008`**, tanggal posting **`2020-12-01`**
> (site dev hanya punya Fiscal Year 2020 — lihat §2.1 no. 3).

### 6.0 Prasyarat (cek sekali di awal aplikasi)

| # | Prasyarat | Cara cek / buat |
|---|---|---|
| 1 | Fiscal Year aktif mencakup tanggal posting | §2.1 no. 3 / §5.9 |
| 2 | Gudang asal & tujuan ada, **`is_group = 0`**, **tidak `disabled`** | §5.2 |
| 3 | Gudang **Transit** ada (khusus alur 2 tahap) | §5.2 |
| 4 | Item `is_stock_item = 1`, `disabled = 0` (+ `has_batch_no`/`has_serial_no`) | §5.3 |
| 5 | Stok tersedia di gudang asal (`Bin.actual_qty`) | §5.4 |
| 6 | Serial/batch: `Stock Settings.enable_serial_and_batch_no_for_item = 1` + master Batch bila ber-batch | §2.4, §5.9 |

**Data uji:** `SKU010` = 78 unit di `Finished Goods - APSD`; `SKU008` = 101 unit di
`Work In Progress - APSD`; keduanya UOM `Nos`, tanpa batch/serial.

### 6.1 TAHAP 1 — Pengajuan / permintaan transfer (`Material Request`)

**Langkah 1.1 — CREATE (draft).** Kirim 1 baris per produk; `from_warehouse` = gudang **asal**,
`warehouse` = gudang **tujuan**:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Material Request",
      "naming_series": "MAT-MR-.YYYY.-",
      "material_request_type": "Material Transfer",
      "company": "PT Rapupa Guna Teknologi (Demo)",
      "transaction_date": "2020-12-01",
      "schedule_date": "2020-12-01",
      "items": [
        { "item_code": "SKU010", "qty": 10, "uom": "Nos", "stock_uom": "Nos",
          "conversion_factor": 1, "from_warehouse": "Finished Goods - APSD",
          "warehouse": "Store Semarang - APSD", "schedule_date": "2020-12-01" },
        { "item_code": "SKU008", "qty": 20, "uom": "Nos", "stock_uom": "Nos",
          "conversion_factor": 1, "from_warehouse": "Work In Progress - APSD",
          "warehouse": "Store Semarang - APSD", "schedule_date": "2020-12-01" }
      ]
    }
  }'
```

**Respons (HTTP 200, dipersingkat):**

```json
{
  "message": {
    "name": "MAT-MR-2026-00002",
    "docstatus": 0,
    "status": "Draft",
    "transfer_status": "",
    "material_request_type": "Material Transfer",
    "set_from_warehouse": null,
    "set_warehouse": null,
    "company": "PT Rapupa Guna Teknologi (Demo)"
  }
}
```

> ⚠️ **Catatan penting:** header `set_from_warehouse`/`set_warehouse` dikembalikan **`null`** karena
> setiap baris sudah membawa `from_warehouse`/`warehouse` sendiri (ERPNext me-reset field default
> header agar tidak menimpa baris) ✅. Jadi **jangan** mengandalkan header MR; bacalah
> `message.items[]`. Jika ingin memakai header, kosongkan `from_warehouse`/`warehouse` di baris.

**Langkah 1.2 — SUBMIT pengajuan** (dokumen lengkap, §4.2):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.submit \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doc": { "doctype": "Material Request", "name": "MAT-MR-2026-00002", ...dokumen utuh... } }'
```

**Hasil ✅:** `docstatus = 1`, `status = "Pending"`, `per_ordered = 0`, `transfer_status = ""`.
Belum ada pergerakan stok. Simpan `name` MR (`MAT-MR-2026-00002`).

**Langkah 1.3 — (UI) READ detail baris** untuk ditampilkan sebagai daftar permintaan:

```json
{
  "name": "MAT-MR-2026-00002",
  "items": [
    { "item_code": "SKU010", "qty": 10.0, "warehouse": "Store Semarang - APSD", "from_warehouse": "Finished Goods - APSD", "ordered_qty": 0.0, "stock_qty": 10.0, "uom": "Nos", "conversion_factor": 1.0 },
    { "item_code": "SKU008", "qty": 20.0, "warehouse": "Store Semarang - APSD", "from_warehouse": "Work In Progress - APSD", "ordered_qty": 0.0, "stock_qty": 20.0, "uom": "Nos", "conversion_factor": 1.0 }
  ]
}
```

### 6.2 TAHAP 2 — Pengeluaran / transfer keluar (gudang asal → Transit)

**Langkah 2.1 — bangun draft dari MR** (`make_in_transit_stock_entry`). Method ini **tidak
menyimpan** dokumen; ia mengembalikan objek draft untuk dikirim ulang ke `frappe.client.insert`:

```bash
curl -X POST https://site-anda.com/api/method/erpnext.stock.doctype.material_request.material_request.make_in_transit_stock_entry \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "source_name": "MAT-MR-2026-00002",
    "in_transit_warehouse": "Goods In Transit - APSD"
  }'
```

**Respons (HTTP 200, dipersingkat)** — `docstatus: 0`, `add_to_transit: 1`, `to_warehouse` = Transit,
tiap baris `s_warehouse` = gudang asal MR & `t_warehouse` = Transit ✅:

```json
{
  "message": {
    "doctype": "Stock Entry",
    "name": null,
    "docstatus": 0,
    "stock_entry_type": "Material Transfer",
    "purpose": "Material Transfer",
    "add_to_transit": 1,
    "posting_date": "2026-09-24",
    "from_warehouse": null,
    "to_warehouse": "Goods In Transit - APSD",
    "total_incoming_value": 11660.0,
    "total_outgoing_value": 11660.0,
    "items": [
      { "idx": 1, "item_code": "SKU010", "qty": 10.0, "uom": "Nos", "conversion_factor": 1.0,
        "s_warehouse": "Finished Goods - APSD", "t_warehouse": "Goods In Transit - APSD",
        "basic_rate": 500.0, "cost_center": "Main - APSD",
        "material_request": "MAT-MR-2026-00002" },
      { "idx": 2, "item_code": "SKU008", "qty": 20.0, "uom": "Nos", "conversion_factor": 1.0,
        "s_warehouse": "Work In Progress - APSD", "t_warehouse": "Goods In Transit - APSD",
        "basic_rate": 333.0, "cost_center": "Main - APSD",
        "material_request": "MAT-MR-2026-00002" }
    ]
  }
}
```

> `name` masih `null` (belum tersimpan) dan `posting_date` = **hari ini** → **wajib** ditimpa bila
> perlu tanggal lain. Modifikasi minimal sebelum insert:
> `doc["name"] = None` (biarkan), `doc["set_posting_time"] = 1`, `doc["posting_date"] = "2020-12-01"`.
> Field `name`/`idx`/child `name` boleh tetap dikirim — Frappe mengabaikannya untuk dokumen baru.

**Langkah 2.2 — INSERT draft** (kirim objek `message` di atas sebagai `doc`) → `name` diberikan
sistem (`MAT-STE-2026-00001`) dengan `docstatus = 0`.

**Langkah 2.3 — SUBMIT** draft tersebut (§4.2).

**Hasil ✅ (stok & status):**

| Objek | Sebelum | Sesudah tahap 2 |
|---|---|---|
| `Bin` SKU010 @ `Finished Goods - APSD` | 78 | **68** (−10) |
| `Bin` SKU010 @ `Goods In Transit - APSD` | — | **10** (+10) |
| `Material Request.status` | `Pending` | **`Transferred`** |
| `Material Request.transfer_status` | `""` | **`In Transit`** |
| `Material Request.per_ordered` | 0 | **100** |
| `Stock Entry.per_transferred` (dok. keluar) | 0 | **0** (belum diterima) |

### 6.3 TAHAP 3 — Penerimaan / transfer masuk (Transit → gudang tujuan)

**Langkah 3.1 — bangun draft penerimaan** dari dokumen keluar (`make_stock_in_entry`):

```bash
curl -X POST https://site-anda.com/api/method/erpnext.stock.doctype.stock_entry.stock_entry.make_stock_in_entry \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "source_name": "MAT-STE-2026-00001" }'
```

**Respons (HTTP 200, dipersingkat)** — gudang asal = Transit, gudang tujuan = gudang tujuan MR ✅:

```json
{
  "message": {
    "doctype": "Stock Entry",
    "name": null,
    "docstatus": 0,
    "stock_entry_type": "Material Transfer",
    "purpose": "Material Transfer",
    "add_to_transit": 0,
    "outgoing_stock_entry": "MAT-STE-2026-00001",
    "to_warehouse": null,
    "items": [
      { "idx": 1, "item_code": "SKU010", "qty": 10.0, "uom": "Nos",
        "s_warehouse": "Goods In Transit - APSD", "t_warehouse": "Store Semarang - APSD",
        "basic_rate": 500.0, "cost_center": "Main - APSD",
        "against_stock_entry": "MAT-STE-2026-00001", "ste_detail": "vjagqlm6m2",
        "expense_account": "5110.020 - Penyesuaian Stock - APSD" },
      { "idx": 2, "item_code": "SKU008", "qty": 20.0, "uom": "Nos",
        "s_warehouse": "Goods In Transit - APSD", "t_warehouse": "Store Semarang - APSD",
        "basic_rate": 333.0, "cost_center": "Main - APSD",
        "against_stock_entry": "MAT-STE-2026-00001", "ste_detail": "vjade8ao1l",
        "expense_account": "5110.020 - Penyesuaian Stock - APSD" }
    ]
  }
}
```

> `t_warehouse` diambil dari `Material Request Item.warehouse` (gudang tujuan) — itulah sebabnya
> dokumen penerimaan **sebaiknya dibuat dari MR** (`make_in_transit_stock_entry`) agar rantainya
> lengkap. Bila stok dibagi ke beberapa tujuan, ubah `t_warehouse` per baris sebelum insert.

**Langkah 3.2 — INSERT + timpa tanggal**, **Langkah 3.3 — SUBMIT**.

**Hasil ✅ (akhir alur):**

| Objek | Nilai akhir |
|---|---|
| `Bin` SKU010 @ `Store Semarang - APSD` | **10** |
| `Bin` SKU008 @ `Store Semarang - APSD` | **20** |
| `Bin` @ `Goods In Transit - APSD` | **0** (transit kosong) |
| `Material Request.transfer_status` | **`Completed`** |
| `Stock Entry.per_transferred` (dokumen keluar) | **100.0** |
| `Stock Entry` penerimaan | `outgoing_stock_entry` = nama dokumen keluar; baris `transferred_qty` = 10/20 |

### 6.4 Verifikasi (Bin + Stock Ledger + status)

```bash
# 1) Saldo akhir per gudang
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Bin", "fields": ["item_code","warehouse","actual_qty"],
        "filters": [["item_code","in",["SKU010","SKU008"]],["warehouse","in",["Finished Goods - APSD","Work In Progress - APSD","Goods In Transit - APSD","Store Semarang - APSD"]]],
        "limit_page_length": 0 }'

# 2) Jejak pergerakan (SLE) untuk kedua dokumen
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "doctype": "Stock Ledger Entry",
        "fields": ["item_code","warehouse","actual_qty","qty_after_transaction","voucher_type","voucher_no","is_cancelled"],
        "filters": [["voucher_no","in",["MAT-STE-2026-00001","MAT-STE-2026-00002"]]],
        "order_by": "creation asc", "limit_page_length": 0 }'
```

Contoh hasil SLE ✅: tahap 2 → `SKU010 @ Finished Goods - APSD actual_qty = -10`, `SKU010 @ Goods In
Transit - APSD actual_qty = +10`; tahap 3 → `-10` dari Transit dan `+10` ke `Store Semarang - APSD`.
Total nilai keluar = masuk (11660) sehingga `value_difference = 0`.

> Struktur & pembacaan SLE lengkap: **[`prd_stock_ledger.md`](./prd_stock_ledger.md)**.

### 6.5 Varian alur

**(A) Transfer langsung 1 dokumen (tanpa Transit).** Untuk gudang yang berdekatan / kirim langsung —
pakai `make_stock_entry` dari MR yang sama (§1 no. 12) atau buat Stock Entry `Material Transfer`
manual (§4.1 varian A). Hasil ✅: `Bin` berpindah langsung sumber → tujuan, MR `per_ordered = 100`,
`transfer_status = ""` (tidak ada tahap transit).

```bash
curl -X POST https://site-anda.com/api/method/erpnext.stock.doctype.material_request.material_request.make_stock_entry \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{ "source_name": "MAT-MR-2026-00003" }'
```

**(B) Transfer sebagian (partial receipt).** Buat dokumen penerimaan dengan `qty` lebih kecil
(mis. 6 dari 10) → `per_transferred` dokumen keluar = 60, `transfer_status` MR tetap `In Transit`.
Sisa dapat diterima pada dokumen penerimaan berikutnya (`make_stock_in_entry` lagi → baris hanya
yang `qty - transferred_qty > 0` ✅) hingga `per_transferred = 100` → `Completed`.
Dokumen penerimaan kedua dibuat dengan `outgoing_stock_entry` yang sama; jangan melebihi qty yang
diminta → `417 ValidationError: Row 1: Transferred quantity cannot be greater than the requested quantity.`

**(C) Penerimaan manual.** Buat draft Stock Entry `Material Transfer` dengan
`outgoing_stock_entry = "<dokumen keluar>"`, lalu isi baris via `run_doc_method`
`set_items_for_stock_in` (§5.6), isi `to_warehouse`, submit.

### 6.6 Barang keluar karena dipakai/rusak (Material Issue)

Gunakan bila tidak ada gudang tujuan (pemakaian internal, barang rusak, hilang):

```bash
# 1) CREATE
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Stock Entry",
      "stock_entry_type": "Material Issue",
      "company": "PT Rapupa Guna Teknologi (Demo)",
      "set_posting_time": 1, "posting_date": "2020-12-01",
      "from_warehouse": "Store Semarang - APSD",
      "remarks": "Pemakaian internal - nota INT-001",
      "items": [
        { "item_code": "SKU010", "qty": 2, "uom": "Nos", "conversion_factor": 1,
          "s_warehouse": "Store Semarang - APSD" }
      ]
    }
  }'
# 2) SUBMIT (dokumen lengkap)
```

**Hasil ✅:** `Bin SKU010 @ Store Semarang - APSD` 10 → **8**; `total_outgoing_value = 1000`
(2 × `basic_rate` 500); `expense_account` terisi otomatis `5110.020 - Penyesuaian Stock - APSD`.
Bila barang rusak seharusnya dibebankan ke akun lain, kirim `expense_account` + `cost_center`
eksplisit pada baris.

> **Jangan** memakai Material Issue untuk menaikkan/menyelaraskan saldo — itu tugas
> **Stock Reconciliation** (perbandingan lengkap: §4.6).

### 6.7 Serial No & Batch No — contoh lengkap

**Prasyarat ✅:** `Stock Settings.enable_serial_and_batch_no_for_item = 1` (§2.4), item
`AABB` (`has_batch_no = 1`, `has_serial_no = 1`, `has_expiry_date = 1`), master Batch
`BATCH-AABB-01` sudah dibuat (§2.4 no. 2).

**(1) Masuk 3 unit dengan 1 batch + 3 serial:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Stock Entry",
      "stock_entry_type": "Material Receipt",
      "company": "PT Rapupa Guna Teknologi (Demo)",
      "set_posting_time": 1, "posting_date": "2020-12-01",
      "to_warehouse": "Gudang 2 - APSD",
      "items": [{
        "item_code": "AABB", "qty": 3, "uom": "Nos", "conversion_factor": 1,
        "t_warehouse": "Gudang 2 - APSD", "basic_rate": 100000,
        "batch_no": "BATCH-AABB-01",
        "serial_no": "SN-AABB-001\nSN-AABB-002\nSN-AABB-003"
      }]
    }
  }'
```

➡️ lalu **SUBMIT** memakai **respons CREATE apa adanya** (jangan ambil ulang via GET — §2.4 no. 3).

**Hasil ✅:** `Bin AABB @ Gudang 2 - APSD = 3`; dokumen `Serial No` `SN-AABB-001..003` dibuat
otomatis (`status = Active`, `batch_no = BATCH-AABB-01`, `warehouse = Gudang 2 - APSD`);
`Batch.batch_qty = 3`; satu `Serial and Batch Bundle` **Inward** (`total_qty = 3`).

**(2) Pindahkan 2 unit (2 serial) ke gudang lain:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Stock Entry",
      "stock_entry_type": "Material Transfer",
      "company": "PT Rapupa Guna Teknologi (Demo)",
      "set_posting_time": 1, "posting_date": "2020-12-01",
      "from_warehouse": "Gudang 2 - APSD",
      "to_warehouse": "Finished Goods - APSD",
      "items": [{
        "item_code": "AABB", "qty": 2, "uom": "Nos", "conversion_factor": 1,
        "s_warehouse": "Gudang 2 - APSD", "t_warehouse": "Finished Goods - APSD",
        "batch_no": "BATCH-AABB-01",
        "serial_no": "SN-AABB-001\nSN-AABB-002"
      }]
    }
  }'
```

➡️ **SUBMIT** (respons CREATE apa adanya).

**Hasil ✅:** `Bin AABB`: `Finished Goods - APSD = 2`, `Gudang 2 - APSD = 1`;
`Serial No` SN-AABB-001 & 002 pindah `warehouse`, SN-AABB-003 tetap; dua bundle terbentuk
(**Outward** dari gudang asal + **Inward** ke gudang tujuan).

> Untuk **alur 2 tahap**, setiap tahap adalah Stock Entry biasa → pola serial/batch sama; pada tahap 3
> `make_stock_in_entry` sudah menyalin `serial_no`/`batch_no` dari dokumen keluar ✅.

### 6.8 Pembatalan (cancel) di tengah alur

- **Batal sebelum dikirim (draft):** `frappe.client.delete` → dokumen hilang, stok tidak terpengaruh.
- **Batal setelah submit** (`frappe.client.cancel`, §4.5):
  - Stok dikembalikan (SLE pembalik, `is_cancelled = 1`).
  - `Material Request.transfer_status` ikut kembali: cancel dokumen **penerimaan** →
    `Completed` → **`In Transit`**; cancel dokumen **keluar** → **`Not Started`** ✅ (implementasi
    `set_material_request_transfer_status`).
  - ⚠️ Cancel **gagal** bila stok hasil transfer sudah terpakai lagi (mis. sudah di-`Material Issue`)
    karena guard stok negatif ✅:
    `417 NegativeStockError: 5.0 units of Item SKU010 needed in Warehouse Store Semarang - APSD to complete this transaction.`
    → urutan yang benar: batalkan dulu dokumen pemakaian yang mengambil stok itu (cancel dari
    dokumen terbaru ke terlama), baru cancel transfer.
- **Amend**: buat dokumen baru dengan `amended_from` = nama dokumen yang dibatalkan.

### 6.9 Ringkasan state & field penanda

| Tahap | Dokumen | Field penanda |
|---|---|---|
| 1. Pengajuan | `Material Request` (submitted) | `status = Pending`, `transfer_status = ""`, `per_ordered = 0` |
| 2. Pengeluaran | `Stock Entry` (submitted) | `purpose = Material Transfer`, `add_to_transit = 1`, `to_warehouse = <Transit>`, `per_transferred = 0`; MR → `status = Transferred`, `transfer_status = "In Transit"`, `per_ordered = 100` |
| 3. Penerimaan | `Stock Entry` (submitted) | `outgoing_stock_entry = <dok. keluar>`, baris `against_stock_entry` + `ste_detail`, `t_warehouse = <tujuan>`; MR → `transfer_status = "Completed"`; dok. keluar → `per_transferred = 100` |
| Batal | keduanya | `docstatus = 2` (dokumen asli tetap ada; SLE bertanda `is_cancelled = 1`) |

---

## 7. Penanganan error umum

| Kode | Kondisi | Contoh body (verbatim) |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | Role tidak punya akses tulis/submit | `{"message": "Not permitted"}` |
| 404 | Dokumen tidak ditemukan | `{"exc_type":"DoesNotExistError","message":"Resource Not Found"}` |
| 409 | Master Batch duplikat | `{"exc_type":"DuplicateEntryError","message":"('Batch', 'BATCH-AABB-01', IntegrityError(1062, \"Duplicate entry 'BATCH-AABB-01' for key 'PRIMARY'\"))"}` ✅ |
| 417 | Gudang tujuan belum diisi | `{"exc_type":"ValidationError","message":"Target warehouse is mandatory for row 1"}` ✅ |
| 417 | Gudang asal belum diisi | `{"exc_type":"ValidationError","message":"Source warehouse is mandatory for row 1"}` |
| 417 | Gudang asal ≠ wajib sama (purpose lain) | `{"exc_type":"ValidationError","message":"Source and target warehouse cannot be same for row 0"}` |
| 417 | Gudang asal = tujuan untuk Material Transfer **dan** `validate_material_transfer_warehouses = 1` | `{"exc_type":"ValidationError","message":"Row #1: Source and Target Warehouse cannot be the same for Material Transfer"}` |
| 417 | Item bukan stock item | `{"exc_type":"ValidationError","message":"<item> is not a stock Item"}` |
| 417 | Qty ≤ 0 | `{"exc_type":"ValidationError","message":"Row 1: The item <item>, quantity must be positive number"}` |
| 417 | UOM tanpa faktor konversi | `{"exc_type":"ValidationError","message":"Row 1: UOM Conversion Factor is mandatory"}` |
| 417 | Stok kurang (saat submit) | `{"exc_type":"NegativeStockError","message":"<strong>999934.0</strong> units of <a ...>Item SKU010: Camera</a> needed in <a ...>Warehouse Finished Goods - APSD</a> to complete this transaction."}` ✅ |
| 417 | Serial/batch: qty > ketersediaan | `{"exc_type":"ValidationError","message":"For the item <strong>AABB</strong>, the Available qty <strong>2.0</strong> is less than the Required Qty <strong>5.0</strong> in the warehouse ..."}` ✅ |
| 417 | Item ber-batch tanpa `batch_no` | `{"exc_type":"ValidationError","message":"At row 1: Batch No is mandatory for Item AABB"}` ✅ |
| 417 | `batch_no` belum terdaftar | `{"exc_type":"LinkValidationError","message":"Could not find Row #1: Batch No: BATCH-AABB-01"}` ✅ |
| 417 | Submit ulang lewat GET saat baris masih berisi `serial_no`/`batch_no` + bundle | `{"exc_type":"ValidationError","message":"At row 1: Serial and Batch Bundle <hash> has already created. Please remove the values from the serial no or batch no fields."}` ✅ (§2.4 no. 3) |
| 417 | Setting serial/batch belum aktif | `{"exc_type":"ValidationError","message":"Please check the 'Activate Serial / Batch No for Item' checkbox in the <a ...>Stock Settings</a> to make ..."}` ✅ |
| 417 | Ubah field terlarang setelah submit | `{"exc_type":"UpdateAfterSubmitError","message":"Row #1: Not allowed to change Serial No after submission from SN-AABB-001"}` ✅ |
| 417 | Hapus dokumen submitted | `{"exc_type":"ValidationError","message":"Stock Entry MAT-STE-2026-00002: Submitted Record cannot be deleted. You must Cancel it first."}` ✅ |
| 417 | Timestamp tidak cocok (submit hanya `doctype`+`name`) | `{"exc_type":"TimestampMismatchError","message":"Error: MAT-MR-2026-00001 (Material Request) has been modified after you have opened it ..."}` ✅ |
| 417 | MR → Stock Entry padahal MR belum submit | `{"exc_type":"ValidationError","message":"Cannot map because following condition fails: docstatus=1"}` ✅ |
| 417 | MR sudah diterima penuh | `{"exc_type":"ValidationError","message":"Goods are already received against the outward entry <name>"}` |
| 417 | Qty penerimaan melebihi yang diminta | `{"exc_type":"ValidationError","message":"Row 1: Transferred quantity cannot be greater than the requested quantity."}` |
| 417 | Fiscal Year belum ada untuk tanggal posting | `{"exc_type":"FiscalYearError","message":"Transaction Date 24-09-2026 is not in any active Fiscal Year for <strong>PT Rapupa Guna Teknologi (Demo)</strong>"}` ✅ |
| 500 | Parameter method kurang (mis. `in_transit_warehouse` tidak dikirim) | `{"exc_type":"TypeError","message":"make_in_transit_stock_entry() missing 1 required positional argument: 'in_transit_warehouse'"}` ✅ |

> Karena semua pemanggilan memakai `/api/method/...`, respons sukses dibungkus `message`
> (bukan `data`). Body error berbentuk
> `{ "exc_type", "exception", "message", "_exc_source", "exc" }` — **`message`** yang dipakai untuk
> pesan ke user (berisi HTML ringan, boleh di-strip).

---

## 8. Koleksi Postman

Seluruh pemanggilan di dokumen ini tersedia di koleksi
`docs/postman/postman_erpnext_api.json` (Collection v2.1) dalam **dua folder**:

- **`19. Stock Entry`** — 39 request:
  - **Pendukung & CRUD:** 19.1 GET Stock Entry Type, 19.2 GET Stock Settings, 19.3 CREATE draft
    `Material Transfer`, 19.4 SUBMIT, 19.5 READ, 19.6 GET `get_value`, 19.7 `get_list`,
    19.8 `get_count`, 19.9 `set_value` (draft), 19.10 `save` (draft), 19.11 CANCEL,
    19.12 DELETE draft, 19.13 amend (`amended_from`)
  - **Per purpose:** 19.14 Material Receipt, 19.15 Material Issue (akun otomatis),
    19.16 Material Issue (akun eksplisit), 19.17 GET stok `Bin`
  - **Alur 3 tahap:** 19.18 `make_in_transit_stock_entry`, 19.19 INSERT draft pengeluaran,
    19.20 SUBMIT pengeluaran, 19.21 GET status MR (`In Transit`), 19.22 `make_stock_in_entry`,
    19.23 INSERT draft penerimaan, 19.24 SUBMIT penerimaan, 19.25 GET status MR (`Completed`),
    19.26 GET SLE kedua tahap
  - **Manual/opsional:** 19.27 `run_doc_method` `set_items_for_stock_in`,
    19.28 `run_doc_method` `get_item_details`
  - **Serial/batch:** 19.29 CREATE Batch master, 19.30 CREATE receipt batch+serial,
    19.31 SUBMIT receipt, 19.32 CREATE transfer batch+serial, 19.33 SUBMIT transfer,
    19.34 GET Serial No, 19.35 GET Serial and Batch Bundle
  - **Error cases:** 19.36 tanpa gudang tujuan, 19.37 qty > stok, 19.38 gudang asal = tujuan,
    19.39 hapus dokumen submitted
- **`20. Material Request`** — 12 request:
  20.1 GET list pengajuan, 20.2 CREATE (transfer 2 produk), 20.3 SUBMIT, 20.4 READ (baris),
  20.5 GET status, 20.6 GET list monitoring (`transfer_status` belum `Completed`), 20.7 `set_value`,
  20.8 CANCEL, 20.9 varian `make_stock_entry` (langsung), 20.10 varian
  `make_in_transit_stock_entry` (Transit), 20.11 error map MR belum submit, 20.12 `get_count`

**Variabel yang perlu diisi** (Collection Variables):

| Variabel | Contoh | Keterangan |
|---|---|---|
| `se_company` | `PT Rapupa Guna Teknologi (Demo)` | company transaksi |
| `se_warehouse_from` / `se_warehouse_from_2` | `Finished Goods - APSD` / `Work In Progress - APSD` | gudang asal (produk 1/2) |
| `se_warehouse_to` | `Store Semarang - APSD` | gudang tujuan |
| `se_warehouse_transit` | `Goods In Transit - APSD` | gudang transit (alur 2 tahap) |
| `se_item_code` / `se_item_code_2` | `SKU010` / `SKU008` | produk 1/2 (tanpa batch) |
| `se_qty` / `se_qty_2` | `3` / `20` | qty transfer |
| `se_batch_item` | `AABB` | produk ber-batch & serial |
| `se_batch_no` | `BATCH-AABB-01` | master batch |
| `se_serial_no` | `SN-AABB-001\nSN-AABB-002\nSN-AABB-003` | serial, dipisah `\n` |
| `se_posting_date` | `2020-12-01` | wajib di dalam Fiscal Year aktif (§2.1 no. 3) |
| `se_expense_account` / `se_cost_center` | `5110.020 - Penyesuaian Stock - APSD` / `Main - APSD` | Material Issue dengan akun eksplisit |
| `se_stock_entry_id` | (otomatis dari 19.3) | dokumen draft utama |
| `se_outgoing_stock_entry_id` | (otomatis dari 19.19) | dokumen pengeluaran (tahap 2) |
| `se_stock_entry_in_id` | (otomatis dari 19.23) | dokumen penerimaan (tahap 3) |
| `se_serial_batch_entry_id` / `se_serial_transfer_id` | (otomatis dari 19.30 / 19.32) | dokumen uji serial/batch |
| `se_material_request_id` | (otomatis dari 20.2) | dokumen pengajuan |
| `se_doc_full` | (diisi pre-request script) | dokumen utuh untuk 19.4/19.10/19.20/19.24/19.31/19.33 |

> **Catatan teknis Postman:** request SUBMIT/SAVE memakai **pre-request script** yang memanggil
> `frappe.client.get` lalu menyimpan dokumen utuh ke variabel `se_doc_full` dan mengirimkannya
> sebagai body — inilah cara yang benar untuk `frappe.client.submit` (§4.2). Test script memakai
> `pm.collectionVariables.set(...)` agar tidak bergantung pada environment yang dipilih.

### Dokumen terkait

- [`prd_item.md`](./prd_item.md) — master barang (`is_stock_item`, `has_batch_no`, `has_serial_no`)
- [`prd_item_group.md`](./prd_item_group.md) — pengelompokan barang (dropdown/filter)
- [`prd_stock_ledger.md`](./prd_stock_ledger.md) — kartu stok & audit pergerakan (SLE)
- [`prd_stock_balance.md`](./prd_stock_balance.md) — saldo stok per gudang/periode
- [`prd_stock_reconciliation.md`](./prd_stock_reconciliation.md) — stok opname/koreksi saldo (§4.6)
- [`../setup/prd_warehouse.md`](../setup/prd_warehouse.md) — master gudang (+ gudang Transit)
- [`../setup/prd_pos_profile.md`](../setup/prd_pos_profile.md) — profil POS (pemakaian stok)
- [`../prd_oauth.md`](../prd_oauth.md) — autentikasi
