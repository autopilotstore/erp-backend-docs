# PRD — REST API Doctype Pricing Rule (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Pricing Rule** di ERPNext (Frappe) — aturan diskon/harga
> promo yang dihitung otomatis saat transaksi — diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Accounts (ERPNext) — path `erpnext.accounts.doctype.pricing_rule`
- **Doctype:** `Pricing Rule` (+ child table `Pricing Rule Item Code`, `Pricing Rule Item Group`, `Pricing Rule Brand`)
- **Versi API:** `/api/method/...` (API v1) — method whitelisted `frappe.client.*` (+ 1 method ERPNext untuk simulasi promo), `name` dikirim di **body**
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

> Dokumen ini lanjutan dari **[prd_item_price.md](./prd_item_price.md)**. Di dokumen tersebut `Pricing Rule`
> disebut sebagai tempatnya **harga bertingkat (`min_qty`/`max_qty`)**, diskon, dan skema promo — semuanya
> dibahas tuntas di sini.
> Seluruh aturan, pesan error, dan contoh telah **diverifikasi langsung pada instance ERPNext v16** (data uji
> dibuat lalu di-*rollback*, kecuali pesan yang ditandai `*` di §6 yang berasal dari pembacaan kode sumber).

> **Ringkas — beda peran `Item Price` vs `Pricing Rule`:**
> | | `Item Price` | `Pricing Rule` |
> |---|---|---|
> | Isi | **harga dasar** per item + price list | **aturan** (diskon/harga promo/gratis item) |
> | Kapan dipakai | selalu saat item diambil | menimpa harga dasar **saat transaksi** |
> | Satuan data | 1 baris = 1 harga | 1 dokumen = 1 skema promo |
> | Contoh | "MIN-001 = Rp12.000 di Grosir" | "Beli min. 5 dus → diskon 10%" |
>
> Urutan perhitungan: **Item Price (harga dasar)** → **Pricing Rule (potongan/penimpaan)** → pajak.
> Pricing Rule **tidak menyimpan harga** dan bukan pengganti Item Price.

---

## 1. Ruang lingkup

| # | Endpoint (POST `/api/method/...`) | Operasi | `name` di |
|---|---|---|---|
| 1 | `frappe.client.insert` | Buat Pricing Rule baru (CREATE, §4.1) | body (`doc`) |
| 2 | `frappe.client.get` | Ambil detail 1 Pricing Rule (READ, §4.2) | body |
| 3 | `frappe.client.get_list` | Daftar Pricing Rule / tabel "Daftar Promo" (READ list, §4.2, §5.7) | body (filters) |
| 4 | `frappe.client.get_count` | Total Pricing Rule sesuai filter — pagination (§4.2) | body |
| 5 | `frappe.client.set_value` | Ubah 1 field — non-aktifkan (`disable`), ubah prioritas (§4.3) | body |
| 6 | `frappe.client.save` | Ubah Pricing Rule (UPDATE dokumen penuh, §4.3) | body (`doc`) |
| 7 | `frappe.client.delete` | Hapus Pricing Rule (§4.4) | body |
| 8 | `erpnext.accounts.doctype.pricing_rule.pricing_rule.apply_pricing_rule` | **Simulasi promo** untuk daftar item (preview keranjang, §4.5) | body (`args`) |
| 9 | `frappe.client.insert` | Buat Coupon Code utk rule kupon (ringkas, §4.6) | body (`doc`) |
| 10 | `frappe.client.get_list` | Daftar Coupon Code aktif (dropdown kupon, §4.6, §5.9) | body (filters) |
| 11 | `frappe.client.get_list` | Dropdown Item / Item Group / Brand (isi child table, §5.1) | body (filters) |
| 12 | `frappe.client.get_list` | Dropdown Customer / Customer Group / Territory / Sales Partner / Campaign (§5.2) | body (filters) |
| 13 | `frappe.client.get_list` | Dropdown Supplier / Supplier Group (§5.3) | body (filters) |
| 14 | `frappe.client.get_list` | Dropdown Warehouse (§5.4) | body (filters) |
| 15 | `frappe.client.get_list` | Dropdown UOM milik item (child `items.uom`, §5.5) | body (filters) |
| 16 | `frappe.client.get_list` | Dropdown Price List (`for_price_list`) + default Selling Settings (§5.6) | body (filters) |
| 17 | `frappe.client.get_list` | Opsi semua field Select dari `DocField` (dropdown UI, §5.8) | body (filters) |
| 18 | `upload_file` + `Data Import` + `form_start_import` | Import massal Pricing Rule (§4.8) | body |

> **Konvensi pemanggilan:** method whitelisted **`frappe.client.*`** dengan `name` & filter di **body JSON**
> (bukan URL path) — konsisten dengan [prd_item_price.md](./prd_item_price.md),
> [prd_warehouse.md](../setup/prd_warehouse.md), dan [prd_item.md](./prd_item.md).
> **Format respons:** `/api/method/...` membungkus hasil di **`"message"`** (bukan `"data"`). Pesan
> peringatan/saran dari backend muncul di **`_server_messages`** (mis. saran `threshold_percentage`, §4.1 varian T).

> **Di luar lingkup dokumen ini:** doctype `Promotional Scheme` (skema promo dengan *slab* bertingkat yang
> membuat beberapa Pricing Rule sekaligus), `Coupon Code` secara utuh (hanya bagian yang diperlukan untuk
> kupon, §4.6), `Pricing Rule Item Code/Group/Brand` sebagai dokumen tersendiri, dan `Promotional Scheme`.
> Lihat §7.

---

## 2. Ringkasan field & data

### 2.1 Tabel field `Pricing Rule`

Kolom "Kapan perlu" merujuk ke varian contoh di §4.1.

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `title` | Data | Nama skema promo (tampil di UI & di pesan saran promo). |
| 🔴 **WAJIB** | `apply_on` | Select | Cakupan aturan: `Item Code` · `Item Group` · `Brand` · `Transaction` (diskon level nota). Lihat catatan no. 12. |
| 🔴 **WAJIB** | `price_or_product_discount` | Select | `Price` (potong/menimpa harga) atau `Product` (memberi **item gratis**). |
| 🔴 **WAJIB** | `currency` | Link → Currency | Mata uang aturan. **Wajib sama** dengan currency transaksi & price list (`for_price_list`) — lihat §6. |
| 🔴 **WAJIB** (salah satu) | `selling` / `buying` | Check | Minimal satu `1`. `selling` untuk transaksi jual, `buying` untuk beli. Satu rule boleh **keduanya**. |
| 🔴 **WAJIB** | `items` / `item_groups` / `brands` | Table | Child table sesuai `apply_on` (§2.1.1). `apply_on = Transaction` **tidak** butuh child table. |
| 🔴 **WAJIB** | `rate_or_discount` | Select | `Rate` · `Discount Percentage` · `Discount Amount` — wajib bila `price_or_product_discount = Price`. Default form `Discount Percentag
| 🔴 **WAJIB** | `apply_multiple_pricing_rules` | Check | Wajib diisi = 1 agar multiple promo dapat berjalan bersamaan dalam 1 transaksi. Bila **semua** rule yang cocok mengaktifkan ini → aturan **ditumpuk** (bukan saling menggantikan), diurut prioritas. |
| 🟠 | `discount_percentage` | Float | Dipakai bila `rate_or_discount = Discount Percentage`. |
| 🟠 | `discount_amount` | Currency | Dipakai bila `rate_or_discount = Discount Amount` (per unit). |
| 🟠 | `rate` | Currency | Dipakai bila `rate_or_discount = Rate` — **harga tetap** penimpa `price_list_rate`. |
| 🟠 | `apply_discount_on` | Select | `Grand Total` · `Net Total` — dipakai **hanya** untuk `apply_on = Transaction` (dasar perhitungan diskon nota). |
| 🟠 | `company` | Link → Company | **Perhatikan:** bila tidak dikirim, backend mengisi otomatis dari **default company user** dan aturan hanya berlaku di company itu (catatan no. 2). |
| 🟠 | `applicable_for` | Select | Pembatasan pihak: `Customer` · `Customer Group` · `Territory` · `Sales Partner` · `Campaign` · `Supplier` · `Supplier Group`. Kosong = berlaku umum. |
| 🟠 | `customer`, `customer_group`, `territory`, `sales_partner`, `campaign`, `supplier`, `supplier_group` | Link | **Hanya satu** yang diisi, sesuai `applicable_for` (field lain otomatis dikosongkan backend, catatan no. 3). |
| 🟠 | `min_qty`, `max_qty` | Float | Ambang **kuantitas** (stock qty) berlakunya promo — dasar promo bertingkat. |
| 🟠 | `min_amt`, `max_amt` | Currency | Ambang **nilai baris** (harga × qty, catatan no. 10). |
| 🟠 | `valid_from` | Date | Default `Today`; awal masa berlaku. |
| 🟠 | `valid_upto` | Date | Akhir masa berlaku (harus ≥ `valid_from`). |
| 🟠 | `warehouse` | Link → Warehouse | Batasi promo ke gudang tertentu (mis. hanya gudang toko). |
| 🟠 | `for_price_list` | Link → Price List | Batasi promo ke price list tertentu (currency price list wajib = `currency` rule). |
| 🟠 | `disable` | Check | Default `0`. `1` = aturan tidak dipakai sama sekali (pengganti hapus, §4.4). |
| ⚪ | `priority` + `has_priority` | Select `1`–`20` + Check | Prioritas saat beberapa rule cocok: **nilai lebih tinggi menang**. `has_priority` otomatis `1` bila `priority` diisi. |
| ⚪ | `apply_discount_on_rate` | Check | Diskon bertingkat di atas harga yang **sudah** didiskon (butuh `priority > 1`). |
| ⚪ | `same_item` | Check | Untuk `Product`: item **yang sama** digratiskan (beli X gratis X). Sering diisi otomatis (catatan no. 6). |
| ⚪ | `free_item`, `free_qty`, `free_item_uom`, `free_item_rate` | Link/Float/Link/Currency | Definisi item gratis (`free_qty` = jumlah gratis; `free_item_rate` 0 = benar-benar gratis). |
| ⚪ | `dont_enforce_free_item_qty` | Check | `1` = user boleh mengubah qty item gratis di transaksi tanpa validasi. |
| ⚪ | `round_free_qty` | Check | Bulatkan qty item gratis ke bilangan bulat. |
| ⚪ | `is_recursive`, `recurse_for`, `apply_recursion_over` | Check/Float/Float | Item gratis berulang ("beli 2 gratis 1" terus-menerus). `recurse_for` default terisi `1` bila `free_item`/`same_item` aktif. |
| ⚪ | `mixed_conditions`, `is_cumulative` | Check | `mixed_conditions` = syarat qty/nilai dihitung dari gabungan item; `is_cumulative` = akumulasi qty/nilai dari transaksi sebelumnya dalam periode (wajib isi `valid_from` **dan** `valid_upto`). |
| ⚪ | `coupon_code_based` | Check | Promo hanya jalan bila transaksi mengirim `coupon_code` yang benar (§4.6). |
| ⚪ | `apply_rule_on_other` + `other_item_code` / `other_item_group` / `other_brand` | Select + Link | Promo diterapkan pada **item lain** (mis. beli Item A → Item B dapat harga khusus). |
| ⚪ | `threshold_percentage` | Percent | Batas toleransi untuk menampilkan **saran promo** ("sale 5 lagi, diskon 40%") alih-alih rule gagal diam-diam. |
| ⚪ | `validate_applied_rule` | Check | Untuk diskon level nota: bila user mengubah diskon manual di bawah nilai rule → backend hanya memperingatkan, tidak menimpa (catatan no. 13). |
| ⚪ | `rule_description` | Small Text | Deskripsi internal promo. |
| ⚪ | `promotional_scheme`, `promotional_scheme_id` | Link/Data | Diisi otomatis bila rule dibuat dari `Promotional Scheme` (di luar scope, §7). |
| ✖️ **Tidak dibahas** | `condition` | Code (Python) | Kondisi Python lanjutan (dievaluasi dengan konteks dokumen). Tidak disarankan untuk UI; validasinya hanya menolak ekspresi ber-`=` sederhana (§6). |
| ⚪ **Otomatis — jangan dikirim** | `name`, `naming_series` | — | `name` memakai seri `PRLE-.####` (contoh `PRLE-0001`). Non-submittable (`docstatus` selalu `0`). |

#### 2.1.1 Child table `items` / `item_groups` / `brands`

| Apply on | Child table | Field tiap baris |
|---|---|---|
| `Item Code` | `items` (Pricing Rule Item Code) | `item_code` (Link → Item), `uom` (Link → UOM, **opsional**) |
| `Item Group` | `item_groups` (Pricing Rule Item Group) | `item_group` (Link → Item Group), `uom` (opsional) |
| `Brand` | `brands` (Pricing Rule Brand) | `brand` (Link → Brand), `uom` (opsional) |
| `Transaction` | — | tidak ada child table |

> `uom` pada baris child = **batasi harga promo ke satuan tertentu** (terverifikasi: rule dengan `uom: "Krat"`
> tidak berlaku saat transaksi bersatuan `Nos`). Kosongkan bila berlaku untuk semua UOM.

### 2.2 Catatan penting

1. **Pricing Rule bukan master harga.** Harga dasar tetap di `Item Price`
   ([prd_item_price.md](./prd_item_price.md)); Pricing Rule hanya **menimpa/memotong** saat transaksi dihitung
   (Sales Invoice, POS Invoice, Quotation, Sales Order, Delivery Note, Purchase Order, dst.).
2. **`company` diisi otomatis dari default company milik user** (`frappe.defaults.get_user_default("Company")`)
   dan rule **hanya berlaku untuk company itu**. Pada uji, rule yang tidak menyertakan `company` terisi company
   *default user* yang **berbeda** dari company transaksi → promo **tidak pernah** kena. Karena itu:
   **selalu kirim `company` eksplisit** dan pastikan sama dengan company transaksi/POS Profile.
3. **Field yang tidak relevan otomatis dikosongkan backend** (`cleanup_fields_value`). Contoh terverifikasi:
   - `apply_on: "Item Code"` + `item_groups` diisi → `item_groups` **dikosongkan**;
   - `rate_or_discount: "Discount Percentage"` + `rate`/`discount_amount` diisi → keduanya **dikosongkan**;
   - `applicable_for: "Customer"` + `supplier` diisi → `supplier` **dikosongkan**;
   - `apply_rule_on_other: "Item Code"` → `other_item_group`/`other_brand` **dikosongkan**.
   Jadi jangan mengirim field di luar logika yang dipilih — nilainya tidak akan tersimpan.
4. **`null` ≠ menghapus nilai.** Mengirim `"field": null` di JSON **diabaikan** sehingga nilai **default**
   tetap dipakai (uji: `rate_or_discount: null` → tersimpan `Discount Percentage`). Untuk mengosongkan field
   opsional, kirim **string kosong** — contoh `"margin_type": ""` (lihat no. 5).
5. **`margin_type` default `Percentage`** → hasil perhitungan promo **selalu** memuat `has_margin: true` walau
   `margin_rate_or_amount` 0. Bila tidak ingin efek margin, kirim `"margin_type": ""` (terverifikasi: kunci
   margin **hilang** dari hasil). Margin hanya dipakai bila currency rule = currency transaksi atau
   `margin_type = Percentage` (§4.1 varian N).
6. **`price_or_product_discount: "Product"` tanpa `free_item`** → `same_item` otomatis `1` (beli X gratis X).
   Bila `mixed_conditions: 1` dan `free_item` kosong → error *"Free item code is not selected"*.
7. **Kupon memakai `name` record, bukan teks kode.** `coupon_code_based: 1` + Coupon Code yang menunjuk rule.
   Saat memanggil simulasi/transaksi, field `coupon_code` harus berisi **`name` record Coupon Code**
   (mis. `CPN-PROMO25`) — **bukan** teks yang diketik pelanggan (`PROMO25`). Terverifikasi: kirim teks kode →
   rule kupon **dilewati**; kirim `name` → promo jalan.
8. **Prioritas menentukan pemenang.** Bila beberapa rule cocok dan tidak ada yang lebih tinggi prioritasnya,
   backend melempar `MultiplePricingRuleConflict` (pesan menyebut nama rule) — kecuali dipanggil dari
   shopping cart/POS (`for_shopping_cart`). Solusi: beri `priority` berbeda, atau aktifkan
   `apply_multiple_pricing_rules` pada **semua** rule yang ingin ditumpuk.
9. **`min_qty`/`max_qty` dibandingkan dengan qty (stock qty)**, sedangkan **`min_amt`/`max_amt` dibandingkan
   dengan nilai baris = `price_list_rate × qty`** — terverifikasi: `min_amt: 5000` cocok pada rate 10.000 × qty 1,
   sedangkan `min_amt: 50000` tidak.
10. **`valid_from` default `Today`.** Konsekuensinya: membuat rule yang "sudah kedaluwarsa" **harus** menyertakan
    `valid_from` di masa lalu pula; kalau tidak, backend menolak (`Valid Up To must be after Valid From`).
    Baris kedaluwarsa **tidak** dihapus/dinonaktifkan otomatis — hanya tidak dipakai.
11. **`is_cumulative` wajib punya `valid_from` dan `valid_upto`** (*"Valid from and valid upto fields are
    mandatory for the cumulative"*), dan **tidak bisa digabung** dengan `is_recursive`
    (*"Recursive Discounts with Mixed condition is not supported by the system"* untuk `mixed_conditions`).
12. **`apply_on = "Transaction"` = diskon level nota**, bukan per item. Ia **tidak** ikut terhitung pada method
    simulasi per item (§4.5); nilainya baru dihitung backend saat dokumen transaksi diproses
    (`apply_pricing_rule_on_transaction`) dan hasilnya masuk ke field **`additional_discount_percentage`** /
    **`discount_amount`** + `apply_discount_on` (§4.7).
13. **`validate_applied_rule` = "jangan timpa keputusan user".** Untuk rule level nota, bila user sudah mengisi
    diskon lebih kecil dari nilai rule dan `validate_applied_rule = 1`, backend hanya menampilkan pesan
    *"User has not applied rule on the invoice {name}"* — tidak menimpa.
14. **Role yang dibutuhkan (v16, dari `DocPerm` + uji `has_permission`):** `Accounts Manager`, `Sales Manager`,
    `Purchase Manager`, `Website Manager`, `System Manager` (read/write/create/delete). Peran `Item Manager`,
    `Stock Manager`, dan `Sales Master Manager` **tidak punya akses** ke Pricing Rule.
    Untuk `Coupon Code`: `Accounts User`, `Sales Manager`, `System Manager`, `Website Manager`.

### 2.3 Cara backend memilih & menerapkan rule

Urutan yang diverifikasi dari kode `erpnext.accounts.doctype.pricing_rule.utils`:

1. **Kumpulkan kandidat per tingkat `apply_on`**: `Item Code` → `Item Group` → `Brand`. Pencarian berhenti pada
   tingkat pertama yang menemukan rule, **kecuali** rule tersebut punya prioritas — maka tingkat berikutnya ikut
   dikumpulkan.
2. **Filter SQL**: `disable = 0`; `selling`/`buying` sesuai dokumen (Sales Invoice → selling, Purchase Invoice →
   buying); `currency` cocok; `for_price_list` cocok atau kosong; `warehouse` cocok atau kosong; `company` cocok;
   `customer`/`supplier`/`campaign`/`sales_partner` cocok **atau rule-nya kosong** (berlaku umum);
   `customer_group`/`territory`/`supplier_group` dicek **hirarkis** (rule pada grup induk ikut berlaku untuk
   sub-grup); `transaction_date` di antara `valid_from`–`valid_upto`.
3. **Filter qty/nilai**: `min_qty`/`max_qty` (stock qty, memperhitungkan konversi UOM baris child) dan
   `min_amt`/`max_amt` (nilai baris). Bila gagal **dan** ada `threshold_percentage`, backend menghasilkan pesan
   saran promo (§4.1 varian T). `apply_rule_on_other` dan `mixed_conditions`/`is_cumulative` memakai konteks
   dokumen (item lain / akumulasi transaksi).
4. **Pilih pemenang**: bila lebih dari satu kandidat → saring yang currency-nya cocok → ambil prioritas
   tertinggi → (khusus `Discount Percentage`) utamakan yang `for_price_list` cocok → bila masih lebih dari satu
   → **error konflik** (`MultiplePricingRuleConflict`).
5. **Terapkan** (`apply_price_discount_rule`) sesuai `rate_or_discount`:
   - `Rate` → **menimpa** `price_list_rate` (dikali `conversion_factor` bila UOM baris child ≠ UOM transaksi),
     `discount_percentage` di-set `0`;
   - `Discount Percentage`/`Discount Amount` → **menambah** `discount_percentage`/`discount_amount`
     (bila `apply_discount_on_rate = 1`, diskon dihitung di atas diskon sebelumnya);
   - `margin_type`/`margin_rate_or_amount` → mengisi margin (`has_margin: true`);
   - `price_or_product_discount = "Product"` → mengisi `free_item_data` (daftar item gratis yang harus
     ditambahkan ke keranjang).

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Detail lengkap setup OAuth Client ada di **[`prd_oauth.md`](../prd_oauth.md)**.

---

## 4. CRUD & contoh jenis promo

### 4.1 CREATE Pricing Rule — `frappe.client.insert`

**Langkah 0 — Pre-check duplikat (disarankan).** Nama rule memakai seri `PRLE-.####` (tidak menyimpan
`title` sebagai kunci), sehingga duplikat tidak diblokir database — cek dulu berdasarkan `title` + `company`:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Pricing Rule",
    "fields": ["name","title","apply_on","rate_or_discount","discount_percentage","valid_from","valid_upto","disable"],
    "filters": [["title","=","Diskon Grosir 10%"],["company","=","PT Maju Jaya"]],
    "limit_page_length": 1
  }'
```

- `message` kosong (`[]`) → lanjut CREATE.
- `message` terisi → tampilkan: *"Promo 'Diskon Grosir 10%' sudah ada (PRLE-0007)."* dan tawarkan buka/ubah.

**Payload minimum (diskon persen per item):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Pricing Rule",
      "title": "Diskon Grosir 10%",
      "apply_on": "Item Code",
      "price_or_product_discount": "Price",
      "company": "PT Maju Jaya",
      "currency": "IDR",
      "selling": 1,
      "rate_or_discount": "Discount Percentage",
      "discount_percentage": 10,
      "valid_from": "2026-09-15",
      "items": [
        { "item_code": "MIN-001" }
      ]
    }
  }'
```

**Contoh respons sukses (HTTP 200, dipersingkat):**

```json
{
  "message": {
    "name": "PRLE-0001",
    "naming_series": "PRLE-.####",
    "docstatus": 0,
    "title": "Diskon Grosir 10%",
    "disable": 0,
    "apply_on": "Item Code",
    "price_or_product_discount": "Price",
    "selling": 1,
    "buying": 0,
    "applicable_for": null,
    "min_qty": 0.0,
    "max_qty": 0.0,
    "min_amt": 0.0,
    "max_amt": 0.0,
    "valid_from": "2026-09-15",
    "valid_upto": null,
    "company": "PT Maju Jaya",
    "currency": "IDR",
    "margin_type": "Percentage",
    "margin_rate_or_amount": 0.0,
    "rate_or_discount": "Discount Percentage",
    "apply_discount_on": "Grand Total",
    "rate": 0.0,
    "discount_amount": 0.0,
    "discount_percentage": 10.0,
    "priority": "",
    "has_priority": 0,
    "apply_multiple_pricing_rules": 1,
    "apply_discount_on_rate": 0,
    "threshold_percentage": 0.0,
    "validate_applied_rule": 0,
    "is_cumulative": 0,
    "mixed_conditions": 0,
    "coupon_code_based": 0,
    "is_recursive": 0,
    "items": [
      { "doctype": "Pricing Rule Item Code", "item_code": "MIN-001", "uom": null }
    ]
  }
}
```

> Simpan `name` (`PRLE-0001`) untuk operasi berikutnya. `margin_type` terisi `Percentage` dari default form —
> lihat catatan §2.2 no. 5 bila tidak ingin efek margin.

### 4.1.1 Daftar varian jenis promo

Semua varian di bawah cukup dijadikan nilai **`doc`** pada `frappe.client.insert` di atas (field `doctype`,
`title`, `currency`, `company`, `selling`/`buying` sudah disertakan di tiap contoh). Kolom "Field kunci"
membantu memetakan ke UI.

| Kode | Jenis promo | Field kunci |
|---|---|---|
| A | Diskon persen | `rate_or_discount: "Discount Percentage"` |
| B | Diskon nominal per unit | `rate_or_discount: "Discount Amount"` |
| C | Harga tetap (menimpa harga) | `rate_or_discount: "Rate"` |
| D | Diskon level nota | `apply_on: "Transaction"` + `min_amt` + `apply_discount_on` |
| E | Diskon per Item Group / Brand | `apply_on: "Item Group"` / `"Brand"` |
| F | Harga khusus per UOM | `items[].uom` |
| G | Promo bertingkat qty | `min_qty` / `max_qty` |
| H | Gratis item lain (bonus) | `price_or_product_discount: "Product"` + `free_item` |
| I | Beli X gratis X | `same_item: 1` (+ `is_recursive`) |
| J | Beli item A → item B dapat harga | `apply_rule_on_other` + `other_*` |
| K | Khusus customer / customer group / territory | `applicable_for` + `customer` / `customer_group` / `territory` |
| L | Khusus Sales Partner / Campaign | `applicable_for: "Sales Partner"` / `"Campaign"` |
| M | Khusus supplier / supplier group (beli) | `buying: 1` + `applicable_for` |
| N | Margin (harga jual dari harga beli) | `margin_type` + `margin_rate_or_amount` |
| O | Promo kupon | `coupon_code_based: 1` (+ Coupon Code, §4.6) |
| P | Khusus warehouse (mis. toko) | `warehouse` |
| Q | Khusus price list | `for_price_list` |
| R | Prioritas & tumpukan aturan | `priority`/`has_priority` + `apply_multiple_pricing_rules` |
| S | Diskon bertingkat di atas diskon | `apply_discount_on_rate: 1` + `priority > 1` |
| T | Periode promo + saran promo | `valid_from`/`valid_upto` + `threshold_percentage` |
| U | Diskon kumulatif & syarat gabungan | `is_cumulative` / `mixed_conditions` |
| V | Rule khusus lewat kondisi Python | `condition` (lanjutan, tidak disarankan) |

---

**Varian A — Diskon persen (per item):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Diskon 10% Minuman",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 10,
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" }, { "item_code": "MIN-002" } ]
}
```

> Hasil di transaksi (rate 10.000 × qty 1): `discount_percentage: 10`, `discount_amount: 1000`.

**Varian B — Diskon nominal per unit:**

```json
{
  "doctype": "Pricing Rule",
  "title": "Potongan Rp1.500",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Amount",
  "discount_amount": 1500,
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ]
}
```

**Varian C — Harga tetap (menimpa harga Item Price):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Harga Spesial Rp7.500",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Rate",
  "rate": 7500,
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ]
}
```

> Hasil: `price_list_rate` menjadi `7500` (harga dasar diabaikan) dan `discount_percentage` = 0.

**Varian D — Diskon level nota (total belanja minimum):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Diskon Nota 5% min. Rp500.000",
  "apply_on": "Transaction",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 5,
  "min_amt": 500000,
  "apply_discount_on": "Net Total",
  "valid_from": "2026-09-15"
}
```

> Tidak butuh child table. Diterapkan saat dokumen dihitung → `additional_discount_percentage = 5`,
> `discount_amount = 5% × net`, `apply_discount_on = "Net Total"` (§4.7). Tambahkan
> `"validate_applied_rule": 1` bila diskon manual user tidak boleh ditimpa (catatan §2.2 no. 13).

**Varian E — Diskon per Item Group / Brand:**

```json
{
  "doctype": "Pricing Rule",
  "title": "Diskon Minuman 12%",
  "apply_on": "Item Group",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 12,
  "valid_from": "2026-09-15",
  "item_groups": [ { "item_group": "Minuman" } ]
}
```

```json
{
  "doctype": "Pricing Rule",
  "title": "Diskon Brand Aqua 8%",
  "apply_on": "Brand",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 8,
  "valid_from": "2026-09-15",
  "brands": [ { "brand": "Aqua" } ]
}
```

> Berlaku hirarkis: rule pada Item Group induk ikut berlaku untuk sub-grup di bawahnya.

**Varian F — Harga khusus per UOM:**

```json
{
  "doctype": "Pricing Rule",
  "title": "Harga Krat Rp105.000",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Rate",
  "rate": 105000,
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001", "uom": "Krat" } ]
}
```

> Prasyarat: UOM `Krat` ada di child `uoms` item (lihat [prd_item_price.md §4.5 varian D](./prd_item_price.md)).
> Rule ini **tidak** berlaku bila transaksi memakai UOM lain (terverifikasi).

**Varian G — Promo bertingkat qty (inilah pengganti "harga bertingkat" yang tidak ada di Item Price):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Diskon 10% min. 5 Krat",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 10,
  "min_qty": 5,
  "max_qty": 20,
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001", "uom": "Krat" } ]
}
```

> Untuk **tier** (5–9 krat 10%, ≥10 krat 15%) buat **rule terpisah** per rentang qty dengan `title` jelas;
> tiap rule punya `min_qty`/`max_qty` sendiri. Jangan lupa atur prioritas bila rentangnya bisa tumpang tindih
> (varian R). Promo bertingkat berbasis **slab dalam satu dokumen** tersedia lewat `Promotional Scheme` (§7).

**Varian H — Gratis item lain (bonus):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Beli 1 MIN-001 gratis 2 MIN-002",
  "apply_on": "Item Code",
  "price_or_product_discount": "Product",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ],
  "free_item": "MIN-002",
  "free_qty": 2,
  "free_item_uom": "Nos",
  "dont_enforce_free_item_qty": 0
}
```

> Hasil resolusi memuat `free_item_data` berisi item gratis (`is_free_item: 1`, `rate: 0`) yang harus
> ditambahkan ke keranjang oleh UI. `free_item_rate` hanya diisi bila item bonus tetap berbayar sebagian.

**Varian I — Beli X gratis X (repeat/recursive):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Beli 2 gratis 1",
  "apply_on": "Item Code",
  "price_or_product_discount": "Product",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "min_qty": 2,
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ],
  "same_item": 1,
  "free_qty": 1,
  "is_recursive": 1,
  "recurse_for": 2,
  "apply_recursion_over": 2,
  "round_free_qty": 1
}
```

> `free_item` boleh dikosongkan: backend otomatis mengisi `same_item = 1`. `is_recursive` (berulang tiap
> 2 qty) tidak dapat digabung dengan `mixed_conditions`.

**Varian J — Beli item A → item B dapat harga khusus:**

```json
{
  "doctype": "Pricing Rule",
  "title": "Beli MIN-001, SKU010 diskon 15%",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 15,
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ],
  "apply_rule_on_other": "Item Group",
  "other_item_group": "Makanan"
}
```

> `apply_rule_on_other` menerima `Item Code` / `Item Group` / `Brand` dan **wajib** diikuti field
> `other_item_code` / `other_item_group` / `other_brand` yang sesuai (kalau tidak → error, §6).

**Varian K — Khusus customer / customer group / territory:**

```json
{
  "doctype": "Pricing Rule",
  "title": "Harga Distributor 20%",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 20,
  "applicable_for": "Customer Group",
  "customer_group": "Distributor",
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ]
}
```

> Ganti `applicable_for` menjadi `Customer` + `customer`, atau `Territory` + `territory`, dst.
> **Hanya satu** field pihak yang boleh diisi (catatan §2.2 no. 3). Rule dengan `applicable_for` jual
> **wajib** `selling = 1` (dan sebaliknya untuk supplier → `buying = 1`), kalau tidak → error.

**Varian L — Khusus Sales Partner / Campaign:**

```json
{
  "doctype": "Pricing Rule",
  "title": "Promo Kemerdekaan 17%",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 17,
  "applicable_for": "Campaign",
  "campaign": "Promo Kemerdekaan",
  "valid_from": "2026-08-01",
  "valid_upto": "2026-08-31",
  "items": [ { "item_code": "MIN-001" } ]
}
```

> `Sales Partner` berguna untuk promo yang hanya berlaku pada penjualan lewat reseller mitra.

**Varian M — Khusus supplier / supplier group (sisi pembelian):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Diskon Supplier Distributor 5%",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "buying": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 5,
  "applicable_for": "Supplier",
  "supplier": "PT Sumber Air",
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ]
}
```

**Varian N — Margin (menentukan harga jual dari harga beli):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Markup 20% dari harga beli",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "buying": 1,
  "margin_type": "Percentage",
  "margin_rate_or_amount": 20,
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ]
}
```

> Hasil: `has_margin: true`, `margin_type: "Percentage"`, `margin_rate_or_amount: 20`. Margin dipakai backend
> untuk menghitung harga jual dari rate beli (dipakai juga di alur purchase → selling).
> Kirim `"margin_type": ""` bila rule murni diskon dan Anda tidak ingin backend memproses margin sama sekali.

**Varian O — Promo kupon:**

```json
{
  "doctype": "Pricing Rule",
  "title": "Kupon HEMAT25",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 25,
  "coupon_code_based": 1,
  "valid_from": "2026-09-15",
  "valid_upto": "2026-10-15",
  "items": [ { "item_code": "MIN-001" } ]
}
```

> Setelah rule dibuat, buat record **Coupon Code** yang menunjuk rule ini (§4.6). Tanpa record tersebut,
> promo kupon **tidak akan pernah** aktif.

**Varian P — Khusus warehouse (mis. hanya di gudang toko):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Promo Outlet Depan Rumah 15%",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 15,
  "warehouse": "Toko Cikarang - PTMJ",
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ]
}
```

> Cocok bila `args.warehouse` (baris/transaksi) sama dengan warehouse rule. Rule tanpa `warehouse` berlaku di
> semua gudang. Untuk POS, warehouse diambil dari `POS Profile.warehouse`
> ([prd_pos_profile.md](../setup/prd_pos_profile.md)).

**Varian Q — Khusus price list:**

```json
{
  "doctype": "Pricing Rule",
  "title": "Diskon khusus price list Grosir",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 18,
  "for_price_list": "Grosir",
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ]
}
```

> `currency` rule **wajib sama** dengan currency price list, jika tidak → error
> *"Currency should be same as Price List Currency: IDR"*. Rule tanpa `for_price_list` berlaku di semua
> price list.

**Varian R — Prioritas & tumpukan aturan:**

```json
{
  "doctype": "Pricing Rule",
  "title": "Promo Bertumpuk A 10%",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 10,
  "priority": "1",
  "has_priority": 1,
  "apply_multiple_pricing_rules": 1,
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ]
}
```

> Pasangannya: rule kedua dengan `discount_percentage: 5`, `priority: "1"`, dan
> `apply_multiple_pricing_rules: 1` juga. Terverifikasi hasilnya **ditumpuk** → `discount_percentage: 15` dan
> `pricing_rules: ["PRLE-0002","PRLE-0001"]`.
> Bila **tidak** semua rule yang cocok memakai `apply_multiple_pricing_rules`, backend hanya memakai satu rule:
> yang **prioritasnya tertinggi** menang. Bila prioritasnya sama dan tidak ada penanda tumpuk → backend
> menolak dengan `MultiplePricingRuleConflict` (pesan menyebut nama rule yang bentrok).

**Varian S — Diskon bertingkat di atas diskon (compound):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Diskon tambahan 5% dari harga diskon",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 5,
  "priority": "2",
  "has_priority": 1,
  "apply_multiple_pricing_rules": 1,
  "apply_discount_on_rate": 1,
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ]
}
```

> `apply_discount_on_rate` **wajib** punya `priority` dan nilainya **lebih dari 1** (kalau tidak → error, §6).
> Efeknya: diskon dihitung dari harga yang sudah didiskon rule berprioritas lebih rendah.

**Varian T — Periode promo & saran promo (`threshold_percentage`):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Diskon Grosir 40% min. 5 dus",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 40,
  "min_qty": 5,
  "threshold_percentage": 70,
  "valid_from": "2026-09-15",
  "valid_upto": "2026-09-30",
  "items": [ { "item_code": "MIN-001" } ]
}
```

> Bila qty nyata belum memenuhi `min_qty` tetapi selisihnya dalam `threshold_percentage`, backend **tidak**
> sekadar melewati rule — ia mengirim pesan saran (muncul di `_server_messages`):
> *"If you sale 5.0 quantities of the item **AABB**, the scheme **Diskon Grosir 40%** will be applied on the
> item."* → tampilkan sebagai toast agar pelanggan menambah qty. (Terverifikasi.)
> `valid_from` default `Today`; untuk rule yang sudah berakhir isi juga `valid_from` di masa lalu (catatan §2.2 no. 10).

**Varian U — Diskon kumulatif (`is_cumulative`) & syarat gabungan (`mixed_conditions`):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Akumulasi belanja bulan ini 10%",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 10,
  "is_cumulative": 1,
  "valid_from": "2026-09-01",
  "valid_upto": "2026-09-30",
  "items": [ { "item_code": "MIN-001" } ]
}
```

> `is_cumulative` menghitung qty/nilai **ditambah** transaksi lain pelanggan pada periode yang sama
> (`valid_from` **dan** `valid_upto` wajib). Berbeda dari `mixed_conditions` yang menghitung gabungan baris
> **di dalam** dokumen yang sama (dipakai bila promo butuh kombinasi beberapa item). Keduanya butuh konteks
> dokumen, jadi tidak ikut terhitung di simulasi per item (§4.5).

**Varian V — Kondisi Python (lanjutan, tidak disarankan untuk UI):**

```json
{
  "doctype": "Pricing Rule",
  "title": "Diskon hari Jumat",
  "apply_on": "Item Code",
  "price_or_product_discount": "Price",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "selling": 1,
  "rate_or_discount": "Discount Percentage",
  "discount_percentage": 5,
  "condition": "frappe.utils.nowdate() == '2026-09-18'",
  "valid_from": "2026-09-15",
  "items": [ { "item_code": "MIN-001" } ]
}
```

> `condition` dievaluasi backend dengan konteks dokumen transaksi. Validasi hanya menolak bentuk ekspresi
> ber-`=` sederhana (*"Invalid condition expression"*). **Disarankan tidak dipakai** pada promo yang
> dikelola lewat UI — gunakan `valid_from`/`valid_upto`, qty/amount, atau party.

### 4.2 READ Pricing Rule

**Detail 1 rule — `frappe.client.get`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Pricing Rule",
    "name": "PRLE-0001"
  }'
```

**Daftar promo aktif (tabel "Daftar Promo" / dropdown) — `frappe.client.get_list`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Pricing Rule",
    "fields": ["name","title","apply_on","price_or_product_discount","rate_or_discount","rate","discount_percentage","discount_amount","selling","buying","applicable_for","customer","customer_group","valid_from","valid_upto","priority","disable","company"],
    "filters": [["disable","=",0],["company","=","PT Maju Jaya"]],
    "order_by": "valid_from desc, name desc",
    "limit_start": 0,
    "limit_page_length": 50
  }'
```

**Contoh respons (HTTP 200, dipersingkat):**

```json
{
  "message": [
    {
      "name": "PRLE-0001",
      "title": "Diskon Grosir 10%",
      "apply_on": "Item Code",
      "price_or_product_discount": "Price",
      "rate_or_discount": "Discount Percentage",
      "rate": 0.0,
      "discount_percentage": 10.0,
      "discount_amount": 0.0,
      "selling": 1,
      "buying": 0,
      "applicable_for": null,
      "customer": null,
      "customer_group": null,
      "valid_from": "2026-09-15",
      "valid_upto": null,
      "priority": "",
      "disable": 0,
      "company": "PT Maju Jaya"
    }
  ]
}
```

> Filter yang berguna di UI: `disable = 0`; `selling`/`buying`; `company`; masa berlaku
> (`[["valid_upto",">=","2026-09-15"]]` untuk yang masih aktif); `apply_on`; `coupon_code_based`.

**Total count (pagination) — `frappe.client.get_count`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_count \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Pricing Rule",
    "filters": [["disable","=",0],["company","=","PT Maju Jaya"]]
  }'
```

```json
{ "message": 12 }
```

### 4.3 UPDATE Pricing Rule — `frappe.client.set_value` / `frappe.client.save`

**Non-aktifkan / ubah satu field (disarankan) — `set_value`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Pricing Rule",
    "name": "PRLE-0001",
    "fieldname": { "disable": 1 }
  }'
```

```json
{ "message": { "name": "PRLE-0001", "title": "Diskon Grosir 10%", "disable": 1, "...": "..." } }
```

> `set_value` juga dipakai untuk mengubah `priority`, `valid_upto`, atau `discount_percentage` satuan.
> **Catatan:** `set_value` menulis langsung ke kolom — **validasi controller tidak dijalankan** (mis. kombinasi
> duplikat child table atau `min_qty > max_qty` tidak dicek). Untuk perubahan yang perlu validasi, gunakan
> `frappe.client.save` (kirim dokumen hasil GET yang dimodifikasi):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Pricing Rule",
      "name": "PRLE-0001",
      "title": "Diskon Grosir 10%",
      "apply_on": "Item Code",
      "price_or_product_discount": "Price",
      "company": "PT Maju Jaya",
      "currency": "IDR",
      "selling": 1,
      "rate_or_discount": "Discount Percentage",
      "discount_percentage": 12,
      "valid_from": "2026-09-15",
      "valid_upto": "2026-12-31",
      "apply_multiple_pricing_rules": 0,
      "margin_type": "",
      "items": [ { "item_code": "MIN-001" } ]
    }
  }'
```

> **Wajib ingat saat UPDATE (`save`):**
> - child table berlaku **replace-all** — kirim daftar lengkap (`items`/`item_groups`/`brands`);
> - field di luar logika yang dipilih akan **dikosongkan** backend (catatan §2.2 no. 3), jadi kirim apa adanya
>   hasil GET (idealnya: GET → ubah → POST);
> - kembali ke default form (`margin_type` dsb.) bisa mengaktifkan kembali efek margin (catatan §2.2 no. 5);
> - `name` tidak boleh diubah.

### 4.4 Non-aktifkan & hapus

**Cara disarankan — non-aktifkan (`disable = 1`)** mempertahankan riwayat & nama rule:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Pricing Rule",
    "name": "PRLE-0001",
    "fieldname": { "disable": 1 }
  }'
```

> Efek: rule **tidak dipakai sama sekali** oleh mesin harga (terverifikasi — transaksi berikutnya tidak mendapat
> promo), tidak muncul di dropdown `disable = 0`, namun tetap terlihat di daftar promo bila filter dihilangkan.

**Alternatif "berakhir otomatis"** — isi `valid_upto` (hari ini/lewah). Rule tetap aktif-statusnya, tetapi
terlewat oleh filter tanggal:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Pricing Rule",
    "name": "PRLE-0001",
    "fieldname": { "valid_upto": "2026-09-14" }
  }'
```

**Hapus — `frappe.client.delete`** (dipakai bila promo salah buat & belum pernah dipakai):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.delete \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Pricing Rule",
    "name": "PRLE-0013"
  }'
```

> Terverifikasi: DELETE berhasil (tidak ada validasi `on_trash` dan Pricing Rule tidak direferensikan dokumen
> transaksi — aturan promo **disalin** ke baris transaksi saat dokumen dihitung). Untuk promo yang sudah
> berjalan, lebih aman **non-aktifkan** agar riwayat nama rule tetap ada.

### 4.5 Simulasi promo untuk keranjang — `apply_pricing_rule`

Method whitelisted ini menghitung promo untuk **daftar item** tanpa menyimpan transaksi — ideal untuk preview
keranjang/POS sebelum checkout.

```bash
curl -X POST https://site-anda.com/api/method/erpnext.accounts.doctype.pricing_rule.pricing_rule.apply_pricing_rule \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "args": {
      "doctype": "Sales Invoice",
      "company": "PT Maju Jaya",
      "currency": "IDR",
      "conversion_rate": 1,
      "plc_conversion_rate": 1,
      "price_list": "Standard Selling",
      "transaction_date": "2026-09-15",
      "customer": "Toko Sumber Rejeki",
      "ignore_pricing_rule": 0,
      "items": [
        { "item_code": "MIN-001", "qty": 5, "stock_qty": 5, "price_list_rate": 10000, "uom": "Nos", "conversion_factor": 1 }
      ]
    }
  }'
```

**Contoh respons (HTTP 200) — hasil per baris item:**

```json
{
  "message": [
    {
      "doctype": "Sales Invoice",
      "has_pricing_rule": 1,
      "pricing_rules": "[\n \"PRLE-0001\"\n]",
      "price_or_product_discount": "Price",
      "rate_or_discount": "Discount Percentage",
      "discount_percentage": 10.0,
      "discount_amount": 5000.0,
      "has_margin": true,
      "margin_type": "Percentage",
      "margin_rate_or_amount": 0.0,
      "free_item_data": []
    }
  ]
}
```

> **Cara membaca respons:**
> - `pricing_rules` = daftar nama rule yang terpakai (JSON **string**, perlu `JSON.parse`); `[]` = tidak ada promo;
> - `has_pricing_rule: 1` walau `pricing_rules` kosong berarti rule **ditemukan tapi tidak diterapkan**
>   (mis. kupon tanpa kode, qty belum memenuhi ambang);
> - `discount_percentage` / `discount_amount` = potongan yang harus dipakai keranjang;
> - `price_list_rate` (bila `rate_or_discount = Rate`) = harga pengganti dari rule;
> - `free_item_data` = daftar item bonus (`item_code`, `qty`, `is_free_item: 1`, `rate: 0`) yang harus
>   ditambahkan UI ke keranjang;
> - `has_margin`/`margin_type`/`margin_rate_or_amount` = informasi margin (lihat catatan §2.2 no. 5).
>
> **Batasan yang sudah diverifikasi:** rule `apply_on = "Transaction"` **tidak** muncul di sini (discount nota
> dihitung saat dokumen transaksi dihitung, §4.7); sama halnya `is_cumulative`/`mixed_conditions` yang butuh
> konteks dokumen. Bila beberapa rule cocok dan tidak ada prioritas, method ini **melempar**
> `MultiplePricingRuleConflict` (§6) — tangkap dan tampilkan pilihan promo ke user.
> Backend juga bisa mengirim `_server_messages` berisi saran promo (varian T).

### 4.6 Promo kupon — `coupon_code_based` + doctype `Coupon Code`

Setelah rule kupon dibuat (§4.1 varian O), buat **Coupon Code** yang menunjuk rule tersebut:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Coupon Code",
      "coupon_name": "CPN-HEMAT25",
      "coupon_type": "Promotional",
      "coupon_code": "HEMAT25",
      "pricing_rule": "PRLE-0019",
      "valid_from": "2026-09-15",
      "valid_upto": "2026-10-15",
      "maximum_use": 5,
      "customer": null
    }
  }'
```

> - `coupon_name` = `name` record (autoname `field:coupon_name`), `coupon_code` = teks yang **diketik
>   pelanggan** (boleh berbeda dari nama record).
> - `coupon_type`: `Promotional` (promo) atau `Gift Card`.
> - `maximum_use` + `used` mengontrol kuota; `customer` mengunci kupon ke satu pelanggan.
> - **Penting:** saat memanggil simulasi/transaksi, kirim **`coupon_code` = `coupon_name`** (nama record,
>   mis. `CPN-HEMAT25`), **bukan** teks `HEMAT25` — terbukti rule dilewati bila dikirim teks kode.
>   Karena field `coupon_code` pada transaksi adalah **Link → Coupon Code**, UI sebaiknya:
>   1. minta pelanggan mengetik kode,
>   2. cari record-nya (`filters: [["coupon_code","=","<ketikan>"]]`, §5.9),
>   3. kirim **`name`** hasil pencarian tersebut.

**Validasi kupon** (dijalankan backend, pesan dari kode sumber): kupon belum mulai
(*"Sorry, this coupon code's validity has not started"*), kedaluwarsa (*"...has expired"*), kuota habis
(*"Sorry, this coupon code is no longer valid"* / *"Coupon used are N. Allowed quantity is exhausted"*).
Saat transaksi diproses, kolom `used` bertambah dan berkurang kembali bila transaksi dibatalkan.

### 4.7 Penerapan di dokumen transaksi (diskon level nota)

Backend memanggil `apply_pricing_rule_on_transaction(doc)` saat dokumen transaksi dihitung. Terverifikasi pada
dokumen `Sales Invoice` in-memory (net Rp200.000) dengan rule `apply_on = "Transaction"`, `min_amt: 100000`,
`discount_percentage: 5`, `apply_discount_on: "Net Total"`:

| | `total` | `net_total` | `grand_total` | `additional_discount_percentage` | `discount_amount` | `apply_discount_on` |
|---|---|---|---|---|---|---|
| sebelum | 200.000 | 200.000 | 200.000 | – | – | – |
| sesudah | 200.000 | **190.000** | **190.000** | **5.0** | **10.000** | **Net Total** |

Untuk sisi UI:

- diskon level nota tersimpan di field **`additional_discount_percentage`** / **`discount_amount`** +
  **`apply_discount_on`** pada dokumen transaksi (Sales Invoice/POS Invoice);
- `POST /api/method/frappe.client.insert` pada Sales Invoice sudah otomatis menerapkan promo ini — UI cukup
  menampilkan nilai hasilnya (jangan menghitung sendiri);
- transaksi yang **tidak boleh** memakai promo sama sekali (mis. nota khusus) → kirim
  **`"ignore_pricing_rule": 1`** pada dokumen; POS Profile juga punya opsi `ignore_pricing_rule`
  ([prd_pos_profile.md](../setup/prd_pos_profile.md));
- rule dengan `validate_applied_rule: 1` **tidak menimpa** diskon manual user yang nilainya lebih kecil —
  backend hanya memperingatkan (catatan §2.2 no. 13).

### 4.8 Import massal (opsional)

`Pricing Rule` mengizinkan import (`allow_import = 1`), jadi pembuatan promo massal (mis. promo per cabang)
dapat memakai alur Data Import 3 langkah yang sama seperti pada
[prd_item_price.md §4.10](./prd_item_price.md) dengan `reference_doctype: "Pricing Rule"` — kolom child table
ditulis sebagai `items` + `items.item_code` (`Import` tipe `Insert`). Karena `name` memakai seri otomatis,
**jangan** mengirim kolom `ID`/`name` untuk penambahan baru.

---

## 5. GET pendukung UI

### 5.1 GET Item / Item Group / Brand — isi child table

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","stock_uom","brand","item_group"],
    "filters": [["disabled","=",0],["has_variants","=",0]],
    "order_by": "item_name asc",
    "limit_page_length": 50
  }'
```

> Ganti `doctype` menjadi `Item Group` (`filters: [["is_group","=",0]]`) atau `Brand` untuk dropdown lainnya.
> Filter per Item Group (hirarkis) mengikuti pola [prd_item.md §6.7](./prd_item.md).

### 5.2 GET Customer / Customer Group / Territory / Sales Partner / Campaign

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Customer",
    "fields": ["name","customer_name","customer_group","territory","default_price_list","disabled"],
    "filters": [["disabled","=",0]],
    "order_by": "customer_name asc",
    "limit_page_length": 50
  }'
```

> Untuk grup: `doctype: "Customer Group"` (leaf: `[["is_group","=",0]]`);
> territory: `doctype: "Territory"`; campaign: `doctype: "UTM Campaign"`;
> sales partner: `doctype: "Sales Partner"` (`filters: [["disabled","=",0]]`).

### 5.3 GET Supplier / Supplier Group

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Supplier",
    "fields": ["name","supplier_name","supplier_group","default_price_list","disabled"],
    "filters": [["disabled","=",0]],
    "order_by": "supplier_name asc",
    "limit_page_length": 50
  }'
```

### 5.4 GET Warehouse — batasi promo per gudang

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "fields": ["name","warehouse_name","company","is_group"],
    "filters": [["is_group","=",0],["disabled","=",0],["company","=","PT Maju Jaya"]],
    "limit_page_length": 0
  }'
```

### 5.5 GET UOM milik item — validasi `items[].uom`

UOM pada child table harus terdaftar pada item. Ambil dari dokumen Item:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "name": "MIN-001"
  }'
```

> Pakai `message.uoms[].uom` sebagai pilihan dropdown `uom` pada baris `items`.

### 5.6 GET Price List & default price list

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Price List",
    "fields": ["name","currency","selling","buying","enabled"],
    "filters": [["enabled","=",1]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

> `currency` price list hasil ini **wajib sama** dengan `currency` rule yang sedang dibuat bila
> `for_price_list` diisi. Default price list jual/beli diambil dari `Selling Settings.selling_price_list` /
> `Buying Settings.buying_price_list` — lihat [prd_item_price.md §5.7](./prd_item_price.md).

### 5.7 GET daftar rule (tabel "Daftar Promo")

Pola lengkap ada di §4.2. Untuk badge "sedang berjalan / belum mulai / berakhir" di UI, hitung dari
`valid_from` & `valid_upto` terhadap tanggal hari ini, dan tampilkan `disable` sebagai status non-aktif.

### 5.8 GET opsi field Select — dari `DocField`

Agar dropdown UI selalu sinkron dengan definisi doctype (tidak perlu hardcode):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer $ACCESS_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "DocField",
    "fields": ["fieldname","fieldtype","options"],
    "filters": [["parent","=","Pricing Rule"],["fieldname","in",["apply_on","price_or_product_discount","rate_or_discount","applicable_for","apply_rule_on_other","apply_discount_on","margin_type"]]],
    "limit_page_length": 0
  }'
```

**Hasil verifikasi (v16)** — opsi dipisah `\n`, entri kosong di depan berarti **boleh dikosongkan**:

| Field | Opsi |
|---|---|
| `apply_on` | `Item Code` · `Item Group` · `Brand` · `Transaction` |
| `price_or_product_discount` | `Price` · `Product` |
| `rate_or_discount` | *(kosong)* · `Rate` · `Discount Percentage` · `Discount Amount` |
| `applicable_for` | *(kosong)* · `Customer` · `Customer Group` · `Territory` · `Sales Partner` · `Campaign` · `Supplier` · `Supplier Group` |
| `apply_rule_on_other` | *(kosong)* · `Item Code` · `Item Group` · `Brand` |
| `apply_discount_on` | `Grand Total` · `Net Total` |
| `margin_type` | *(kosong)* · `Percentage` · `Amount` |
| `priority` | *(kosong)* · `1` … `20` |

### 5.9 GET Coupon Code — dropdown kupon

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Coupon Code",
    "fields": ["name","coupon_code","pricing_rule","valid_from","valid_upto","maximum_use","used","customer"],
    "filters": [["coupon_type","=","Promotional"],["pricing_rule","!=",""]],
    "limit_page_length": 0
  }'
```

> Untuk validasi "kode yang diketik pelanggan", cari record-nya dengan
> `filters: [["coupon_code","=","HEMAT25"]]` lalu kirim kolom **`name`** ke field `coupon_code` transaksi/args
> simulasi (§4.6).

---

## 6. Penanganan error umum

Pesan berikut **persis** seperti yang dikembalikan instance v16. Baris bertanda `*` berasal dari pembacaan
kode sumber (belum dipicu pada uji), sisanya terverifikasi langsung.

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | Role tidak punya akses (mis. hanya `Item Manager`) | `{"message": "Not permitted"}` |
| 404 | `name` tidak ditemukan | `{"exc_type":"DoesNotExistError","message":"Resource Not Found"}` |
| 417 | Rule tanpa `selling` & `buying` | `{"exc_type":"ValidationError","message":"At least one of the Selling or Buying must be selected"}` |
| 417 | `apply_on` dipilih tapi child table kosong | `{"exc_type":"MandatoryError","message":"Item Code is not added in the table"}` |
| 417 | `applicable_for` dipilih tapi field pihaknya kosong | `{"exc_type":"MandatoryError","message":"Customer is required"}` |
| 417 | `applicable_for` jual tapi `selling` belum dicentang | `{"exc_type":"ValidationError","message":"Selling must be checked, if Applicable For is selected as Customer"}` |
| 417 | `applicable_for` beli tapi `buying` belum dicentang | `{"exc_type":"ValidationError","message":"Buying must be checked, if Applicable For is selected as Supplier"}` |
| 417 | `has_priority` tanpa `priority` | `{"exc_type":"MandatoryError","message":"Priority is mandatory"}` |
| 417 | `price_or_product_discount = Price` tanpa `rate_or_discount` | `{"exc_type":"MandatoryError","message":"Rate or Discount is required for the price discount."}` |
| 417 | `price_or_product_discount = Product` + `mixed_conditions` tanpa `free_item` | `{"exc_type":"ValidationError","message":"Free item code is not selected"}` |
| 417 | Qty/amount terbalik | `{"exc_type":"ValidationError","message":"Min Qty can not be greater than Max Qty"}` · `{"exc_type":"ValidationError","message":"Min Amt can not be greater than Max Amt"}` |
| 417 | `rate` negatif | `{"exc_type":"ValidationError","message":"Rate can not be negative"}` |
| 417 | `currency` ≠ currency `for_price_list` | `{"exc_type":"ValidationError","message":"Currency should be same as Price List Currency: IDR"}` |
| 417 | `is_cumulative` tanpa tanggal | `{"exc_type":"ValidationError","message":"Valid from and valid upto fields are mandatory for the cumulative"}` |
| 417 | `valid_upto` < `valid_from` | `{"exc_type":"InvalidDates","message":"<strong>Valid Up To</strong> must be after <strong>Valid From</strong>"}` |
| 417 | Item sama dua kali pada `items` | `{"exc_type":"ValidationError","message":"Duplicate Item Code found in the table"}` |
| 417 | `apply_discount_on_rate` dengan `priority` 1 | `{"exc_type":"ValidationError","message":"As the field <strong>Apply Discount on Discounted Rate</strong> is enabled, the value of the field <strong>Priority</strong> should be more than 1."}` |
| 417 | `apply_rule_on_other` tanpa field `other_*` | `{"exc_type":"ValidationError","message":"For the 'Apply Rule On Other' condition the field <strong>Item Code</strong> is mandatory"}` |
| 417 | Beberapa rule cocok tanpa prioritas | `{"exc_type":"MultiplePricingRuleConflict","message":"Multiple Price Rules exists with same criteria, please resolve conflict by assigning priority. Price Rules: PRLE-0002 PRLE-0001"}` |
| 417 | Item template + variannya sekaligus `*` | `{"exc_type":"ValidationError","message":"Variant KAOS-POLOS-M and its template KAOS-POLOS cannot both be added to the same Pricing Rule"}` |
| 417 | Batas diskon item (field `max_discount` pada Item) `*` | `{"exc_type":"ValidationError","message":"Max discount allowed for item: MIN-001 is 20%"}` |
| 417 | Rekursi tidak masuk akal `*` | `{"exc_type":"ValidationError","message":"Min Qty should be greater than Recurse Over Qty"}` · `{"exc_type":"ValidationError","message":"Recurse Over Qty cannot be less than 0"}` |
| 417 | Kombinasi rekursi & syarat gabungan `*` | `{"exc_type":"ValidationError","message":"Recursive Discounts with Mixed condition is not supported by the system"}` |
| 417 | `condition` berbentuk ekspresi `=` sederhana `*` | `{"exc_type":"ValidationError","message":"Invalid condition expression"}` |
| 417 | Kupon belum mulai / kedaluwarsa / kuota habis `*` | `{"exc_type":"ValidationError","message":"Sorry, this coupon code's validity has not started"}` · `"...has expired"` · `"Sorry, this coupon code is no longer valid"` |

> **Catatan:** karena seluruh pemanggilan memakai `/api/method/...`, hasil sukses dibungkus di `message`.
> Beberapa pesan memuat HTML (`<strong>`) — escape/bersihkan saat menampilkan.
> Peringatan/saran non-fatal (mis. saran `threshold_percentage`, *"User has not applied rule on the invoice X"*)
> **tidak** muncul sebagai error, melainkan pada `_server_messages`.

---

## 7. Catatan & PRD lanjutan

1. **`Promotional Scheme` — PRD terpisah (menyusul).** Untuk skema promo yang butuh **beberapa slab dalam satu
   dokumen** (mis. 5–9 pcs −10%, 10–19 pcs −15%, ≥20 pcs −20%) dengan **satu** master promo, ERPNext
   menyediakan doctype `Promotional Scheme` (`price_discount_slabs`, `product_discount_slabs`, party multi-select).
   Pricing Rule yang lahir dari skema itu menyimpan `promotional_scheme` + `promotional_scheme_id`. Di luar scope
   dokumen ini; akan dibuat sebagai `prd_promotional_scheme.md` bila diperlukan.
   **Alternatif tanpa doctype tambahan:** buat beberapa Pricing Rule dengan `min_qty`/`max_qty` per tier
   (varian G) — sudah cukup untuk mayoritas kebutuhan toko.
2. **`Coupon Code`** hanya dibahas seperlunya (§4.6, §5.9) karena dipilih sebagai bagian dari varian promo.
   Pengelolaan penuh kupon (gift card, kuota, `used` per pelanggan, pembatalan) belum didokumentasikan di sini.
3. **Koleksi Postman:** folder **`14. Pricing Rule`** (28 request) sudah ditambahkan ke
   `docs/postman/postman_erpnext_api.json`: pre-check (14.1), 14 contoh CREATE per jenis promo (14.2–14.15,
   termasuk tumpukan 14.13a/14.13b dan kupon + Coupon Code 14.14/14.15), READ/list/count (14.16–14.18),
   simulasi promo (14.19), update/delete (14.20–14.22), dan dropdown pendukung (14.23–14.27).
   Variabel baru: `pricing_rule_id`, `pricing_rule_id_2`, `pricing_rule_title`, `coupon_code_id`,
   `promo_company`, `promo_currency`, `promo_price_list`, `free_item_code` — **isi `promo_company` lebih dulu**
   (wajib sama dengan company transaksi, §2.2 no. 2). Jalankan folder `0. OAuth 2.0` sebelum request `14.x`;
   test script pada request CREATE menyimpan `name` ke `pricing_rule_id`.
   Folder terkait (harga dasar item): **`15. Price List & Item Price`** — lihat
   [prd_item_price.md §7](./prd_item_price.md).
4. **Verifikasi:** seluruh perilaku di dokumen ini diuji pada instance dev ERPNext v16 dengan pola
   *create → periksa → rollback* (tidak ada data promo yang tertinggal). Pengujian pemakaian promo lewat UI
   kasir (POS) belum dilakukan — di luar lingkup verifikasi backend.
5. **File terkait:** Item Price [prd_item_price.md](./prd_item_price.md) (harga dasar, `standard_rate`,
   `packing_unit`) · Item [prd_item.md](./prd_item.md) (`max_discount`, `item_group`, `brand`) ·
   POS Profile [prd_pos_profile.md](../setup/prd_pos_profile.md) (`ignore_pricing_rule`, `selling_price_list`) ·
   Warehouse [prd_warehouse.md](../setup/prd_warehouse.md) · Customer
   [prd_customer.md](../contact/prd_customer.md) · Supplier [prd_supplier.md](../contact/prd_supplier.md) ·
   OAuth [prd_oauth.md](../prd_oauth.md).
