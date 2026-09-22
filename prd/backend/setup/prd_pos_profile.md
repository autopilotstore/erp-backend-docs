# PRD — REST API Doctype POS Profile (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **POS Profile** (Point of Sale) di
> ERPNext (Frappe), diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Accounts (ERPNext)
- **Doctype:** `POS Profile`
- **Versi API:** `/api/method/...` (API v1) — method whitelisted `frappe.client.*`, `name` dikirim di **body**
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

---

## 1. Ruang lingkup

| # | Endpoint (POST `/api/method/...`) | Operasi | `name` di |
|---|---|---|---|
| 1 | `frappe.client.insert` | Buat POS Profile baru (CREATE) | body (`doc`) |
| 2 | `frappe.client.get` | Ambil detail 1 POS Profile (READ) | body |
| 3 | `frappe.client.get_list` | Daftar POS Profile (READ list) | body (filters) |
| 4 | `frappe.client.save` | Ubah POS Profile (UPDATE) | body (`doc`) |
| 5 | `frappe.client.set_value` | Non-aktifkan (`disabled=1`) — pengganti DELETE | body |
| 6 | `frappe.client.get_list` | Daftar Company (dropdown UI, §5.1) | body |
| 7 | `frappe.client.get_list` | Daftar Warehouse (dropdown UI, §5.2) | body |
| 8 | `frappe.client.get_list` | Daftar Price List (dropdown UI, §5.3) | body |
| 9 | `frappe.client.get_list` / `insert` | Daftar / buat Mode of Payment (dropdown UI, §5.4) | body |
| 10 | `frappe.client.get_list` | Daftar akun (income/expense/write-off, §5.5) | body |
| 11 | `frappe.client.get_list` | Ambil akun `write_off_account` ter-kunci — `account_number=5510.009` (§5.9) | body |
| 12 | `frappe.client.get_list` | Ambil Cost Center `write_off_cost_center` ter-kunci — `cost_center_name="Main"` (§5.10) | body |
| 13 | `frappe.client.get_list` | Daftar Cost Center (dropdown UI, §5.6) | body |
| 14 | `frappe.client.get_list` | Daftar template pajak (dropdown UI, §5.7) | body |
| 15 | `frappe.client.get_list` | Filter & user — Item Group / Customer Group / User (dropdown UI, §5.8) | body |
| 16 | `frappe.client.get_count` | Total record POS Profile sesuai filter — pagination (§4.2) | body |
| 17 | `frappe.client.get_list` / `insert` | Daftar / buat Terms and Conditions — isi printout (kebijakan retur/catatan, §5.11b) | body |
| 18 | `frappe.client.get_list` / `insert` / `attach_file` | Letter Head & logo outlet — header cetak struk/invoice (opsional, §5.11a) | body |

> **Konvensi pemanggilan (penting):** seluruh operasi memakai method whitelisted **`frappe.client.*`**
> dengan `name` (dan filter) dikirim lewat **body JSON**, bukan di URL path — sama dengan
> **[prd_warehouse.md §1](./prd_warehouse.md)**. `name` bisa mengandung karakter seperti `/`;
> `/api/resource/{doctype}/{name}` tidak andal untuk itu di belakang nginx/proxy.
> **Format respons:** method `/api/method/...` membungkus hasil di **`"message"`** (bukan `"data"`).

> Dokumen ini hanya membahas **data wajib terisi** + field penting. Export/import masal
> **tidak dipakai** di aplikasi ini (tidak didokumentasikan). Setup jabatan/permission kasir
> dibahas di **[prd_user.md §8](../contact/prd_user.md)**; studi kasus POS multi-toko ada di §8.

---

## 2. Ringkasan field & data

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `name` | Data | Nama profil, **diisi manual** (`autoname: Prompt` / `Set by user`). Mis. `POS Toko Cikarang`. Tanpa naming series → frontend **wajib mengirim `name`** saat CREATE. |
| 🔴 **WAJIB** | `company` | Link → Company | `reqd: 1`. **Read-only setelah dibuat.** |
| 🔴 **WAJIB** | `currency` | Link → Currency | `reqd: 1`. Mata uang transaksi POS (mis. `IDR`). |
| 🔴 **WAJIB** | `payments` | Table → POS Payment Method | `reqd: 1`. Minimal 1 metode pembayaran. §2.1 no. 3. |
| 🔴 **WAJIB*** | `warehouse` | Link → Warehouse | `reqd` bila `update_stock=1` (`depends_on`). Di versi sekarang `update_stock` selalu `1` → praktis **wajib**. §2.3. |
| 🔴 **WAJIB*** | `write_off_account` | Link → Account | `reqd: 1`. Akun selisih pembulatan/write-off (biasanya `Round Off - {abbr}`). |
| 🔴 **WAJIB*** | `write_off_cost_center` | Link → Cost Center | `reqd: 1`. Cost center utk write-off (biasanya `Main - {abbr}`). |
| 🔴 **WAJIB** | `selling_price_list` | Link → Price List | Kategori harga yang digunakan di POS profil ini (`name` global, tanpa suffix company). |
| 🟠 **DISARANKAN** | `customer` | Link → Customer | Customer default (kasir walk-in), mis. `Walk-In Customer`. Kosong = wajib pilih customer tiap transaksi. |
| 🟠 | `company_address` | Link → Address | Alamat company utk dicetak di invoice. (Single link) |
| 🟠 | `income_account` | Link → Account | Akun pendapatan default (mis. `Sales - {abbr}`). |
| 🟠 | `expense_account` | Link → Account | Akun beban default utk HPP/COGS (mis. `Cost of Goods Sold - {abbr}`). |
| 🟠 | `cost_center` | Link → Cost Center | Default cost center transaksi (mis. `Main - {abbr}`). |
| 🟠 | `account_for_change_amount` | Link → Account | Akun utk menampung uang kembalian. |
| 🟠 | `taxes_and_charges` | Link → Sales Taxes and Charges Template | Template pajak default (mis. `PPN 11% - {abbr}`). |
| 🟠 | `item_groups` | Table → POS Item Group | Hanya tampilkan item dari item group ini (§5.8). Kosong = semua item. |
| 🟠 | `customer_groups` | Table → POS Customer Group | Hanya tampilkan customer dari group ini. Kosong = semua. |
| 🟠 | `applicable_for_users` | Table → POS Profile User | Batasi profil utk user tertentu (kasir). Kosong = berlaku utk semua. |
| 🟠 **DISARANKAN** | `allow_warehouse_change` | Check | Izin kasir mengubah warehouse per item (memungkinkan jual dari beberapa warehouse dalam 1 profil). Default `0`. §2.3. |
| 🟠 **DISARANKAN** | `disabled` | Check | `1` = non-aktif (tidak muncul sebagai pilihan POS). Default `0`. |
| 🟠 **DISARANKAN** | `print_format`, `letter_head`, `select_print_heading`, `tc_name` | Link | Pengaturan cetak struk (Print Format / Letter Head / Print Heading / Terms). |
| 🟠 **DISARANKAN** | `hide_images`, `hide_unavailable_items`, `auto_add_item_to_cart` | Check | Pengaturan tampilan item di POS. Default `0`. |
| 🟠 **DISARANKAN** | `write_off_limit` | Currency | Batas nominal write-off. `reqd`, default `1`. |
| ⚪ **Opsional** | `validate_stock_on_save` | Check | Validasi ketersediaan stok saat simpan (sebelum submit). Default `0`. |
| ⚪ **Opsional** | `ignore_pricing_rule` | Check | Abaikan Pricing Rule (harga selalu dari Price List). Default `0`. |
| ⚪ **Opsional** | `apply_discount_on` | Select | Basis diskon: `Grand Total` / `Net Total`. Default `Grand Total`. |
| ⚪ **Opsional** | `allow_rate_change`, `allow_discount_change` | Check | Izin kasir mengubah harga / diskon. Default `0`. |
| ⚪ **Opsional** | `print_receipt_on_order_complete` | Check | Cetak struk otomatis saat pesanan selesai. Default `0`. |
| ⚪ **Opsional** | `set_grand_total_to_default_mop` | Check | Total otomatis diarahkan ke metode pembayaran default. Default `1`. |
| ⚪ **Opsional** | `allow_partial_payment` | Check | Izinkan pembayaran sebagian (khusus transaksi POS). Default `0`. |
| ⚪ **Opsional** | `action_on_new_invoice` | Select | `Always Ask` / `Save Changes and Load New Invoice` / `Discard Changes and Load New Invoice`. Default `Always Ask`. |
| ⚪ **Opsional** | `disable_rounded_total` | Check | Matikan pembulatan total. Default `0`. |
| ⚪ **Opsional** | `tax_category`, `project`, `utm_campaign`, `utm_source`, `utm_medium` | Link | Kategori pajak / proyek / campaign UTM. |
| ⚪ **Read-only — jangan dikirim** | `country` | Read Only | `fetch_from: company.country`. Otomatis. |
| ⚪ **Otomatis — jangan dikirim** | `update_stock` | Check | Default `1`, `hidden` + `read_only` di versi sekarang (POS selalu update stok). |

### 2.1 Catatan penting

1. **`name` diisi manual** (`autoname: Prompt`) — bukan naming series. Frontend harus mengirim `name`
   saat CREATE, mis. `POS Toko Cikarang` / `POS - Gudang Pusat`. Karena deterministic, cek duplikat
   dilakukan **sebelum** POST (§4.1 Langkah 0) dengan filter `name`.
2. **`company` tidak boleh diubah** setelah POS Profile dibuat (semua link akun/warehouse/cost center
   mengikuti company). `update_stock` di versi sekarang `hidden` + `read_only` default `1` — via API
   tetap bisa dikirim `0`, tetapi **tidak disarankan** (POS tanpa update stock = stok tidak berkurang
   saat penjualan; lihat [prd_warehouse.md §2.3](./prd_warehouse.md)).
3. **`payments` (Table) wajib minimal 1 baris.** `Mode of Payment` harus **sudah ada** di sistem —
   default install ERPNext menyediakan `Cash`, `Bank Transfer`, `Credit Card`, `Check`, `Gift Card`.
   Jika klien butuh metode baru, buat dulu via `frappe.client.insert` (lihat §5.4).
   Satu baris boleh bertanda `default=1` (metode terpilih otomatis di layar POS).
4. **`write_off_account` & `write_off_cost_center` bersifat `reqd`** → CREATE gagal
   (`MandatoryError`) bila kosong. Umumnya isi `Round Off - {abbr}` (akun) dan `Main - {abbr}`
   (cost center) yang sudah dibuat otomatis oleh Chart of Accounts default.
5. **`warehouse` wajib karena `update_stock=1`.** Warehouse harus gudang **ledger** (`is_group=0`)
   di company yang sama — untuk toko/retail pakai gudang toko (`warehouse_type=Store`, lihat
   [prd_warehouse.md §2.3](./prd_warehouse.md)). Stok berkurang **otomatis** saat POS Invoice
   di-submit.
6. **`applicable_for_users`** membatasi siapa yang bisa memakai profil ini di layar POS. Kosong =
   semua user. Untuk satu POS per toko, isi dengan user kasir toko tersebut (§8).
7. **Role yang dibutuhkan** — baca: `Accounts User` / `Accounts Manager`; tulis/buat/hapus:
   `Accounts Manager` (`Sales Manager` hanya select). System Manager otomatis punya akses penuh.

### 2.2 Akun & klien non-akunting

- `income_account` / `expense_account` / `cost_center` dipakai saat Sales Invoice dari POS di-posting
  ke General Ledger. Chart of Accounts default sudah membuat minimal akun `Sales - {abbr}`,
  `Cost of Goods Sold - {abbr}`, dan cost center `Main - {abbr}` → **tidak perlu menyiapkan apa pun**,
  cukup isi nilainya di payload.
- `write_off_account` dipakai utk selisih pembulatan/write-off; akun `Round Off - {abbr}` tersedia di
  default chart of accounts.
- Jika klien **tidak memakai fitur akunting**, pastikan `Company.enable_perpetual_inventory = 0`
  (lihat [prd_warehouse.md §2.2](./prd_warehouse.md)) — namun tetap isi `income_account` /
  `expense_account` / `cost_center` agar Sales Invoice tidak error saat submit.
- **Boleh kosong** (tidak `reqd`): `income_account`, `expense_account`, `cost_center`, dan
  `account_for_change_amount` — CREATE/UPDATE tetap sukses tanpa mengisinya (validasi hanya
  memastikan *"jika diisi, harus company yang sama"*). Namun bila kosong, ERPNext me-resolve default
  dari Item/Company saat submit; bila tidak ada fallback, POS Sales Invoice bisa **gagal posting**.
  Disarankan tetap diisi dengan nilai default (lihat poin pertama) agar transaksi pasti berjalan.

### 2.3 Gudang toko (retail/POS) & update stok

**Gudang toko = gudang ledger biasa** (`is_group=0`, `warehouse_type` bukan `Transit`). POS Profile
menautkannya via field `warehouse`:
- `warehouse` = gudang toko (mis. `Toko Cikarang - PTMJ`), `update_stock = 1` → stok berkurang
  otomatis saat POS Invoice submit.
- Opsional `validate_stock_on_save = 1` → peringatan bila stok kurang saat simpan.
- Stok per item+warehouse dilacak otomatis di doctype **`Bin`** — tidak diinput manual.

> Alur implementasi: buat Warehouse toko (type `Store`) → buat POS Profile dgn `warehouse` = gudang
> toko → kasir buka POS → pilih POS Profile ini. Detail lengkap di
> [prd_warehouse.md §2.3](./prd_warehouse.md).

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Detail lengkap setup OAuth Client ada di **[`prd_oauth.md`](../prd_oauth.md)**.

---

## 4. CRUD — Doctype POS Profile

### 4.1 CREATE — `frappe.client.insert`

**Langkah 0 — Pre-check (wajib sebelum CREATE)**

Karena `name` diisi manual, cek duplikat cukup dengan filter `name` (via `frappe.client.get_list`):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "POS Profile",
    "fields": ["name","company","warehouse","disabled"],
    "filters": [["name","=","POS Toko Cikarang"]],
    "limit_page_length": 1
  }'
```

**Hasil & aturan:**
- `message` kosong (`[]`) → lanjut ke CREATE.
- `message` terisi → blokir CREATE, tampilkan pesan: *"POS Profile {name} sudah ada."*

> **Catatan backend:** tanpa pre-check, ERPNext tetap melempar `DuplicateEntryError` saat insert
> duplikat (`name` unique). Pre-check memberi pesan ramah & lebih cepat.

**Payload minimum (data wajib) — `frappe.client.insert`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "POS Profile",
      "name": "POS Toko Cikarang",
      "company": "PT Maju Jaya",
      "currency": "IDR",
      "warehouse": "Toko Cikarang - PTMJ",
      "write_off_account": "Round Off - PTMJ",
      "write_off_cost_center": "Main - PTMJ",
      "payments": [
        { "mode_of_payment": "Cash", "default": 1, "allow_in_returns": 1 }
      ]
    }
  }'
```

**Contoh request (lengkap — toko retail):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "POS Profile",
      "name": "POS Toko Cikarang",
      "company": "PT Maju Jaya",
      "currency": "IDR",
      "warehouse": "Toko Cikarang - PTMJ",
      "company_address": "Alamat Utama - PTMJ",
      "customer": "Walk-In Customer",
      "selling_price_list": "Standard Selling",
      "income_account": "Sales - PTMJ",
      "expense_account": "Cost of Goods Sold - PTMJ",
      "cost_center": "Main - PTMJ",
      "account_for_change_amount": "Kas - PTMJ",
      "write_off_account": "Round Off - PTMJ",
      "write_off_cost_center": "Main - PTMJ",
      "taxes_and_charges": "PPN 11% - PTMJ",
      "payments": [
        { "mode_of_payment": "Cash", "default": 1, "allow_in_returns": 1 },
        { "mode_of_payment": "Bank Transfer", "default": 0, "allow_in_returns": 1 }
      ],
      "item_groups": [
        { "item_group": "Products" }
      ],
      "customer_groups": [
        { "customer_group": "Individual" }
      ],
      "allow_rate_change": 1,
      "allow_discount_change": 1,
      "print_receipt_on_order_complete": 1
    }
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": {
    "name": "POS Toko Cikarang",
    "owner": "Administrator",
    "creation": "2026-08-27 09:15:00.000000",
    "modified": "2026-08-27 09:15:00.000000",
    "modified_by": "Administrator",
    "docstatus": 0,
    "company": "PT Maju Jaya",
    "currency": "IDR",
    "warehouse": "Toko Cikarang - PTMJ",
    "customer": "Walk-In Customer",
    "selling_price_list": "Standard Selling",
    "income_account": "Sales - PTMJ",
    "expense_account": "Cost of Goods Sold - PTMJ",
    "cost_center": "Main - PTMJ",
    "write_off_account": "Round Off - PTMJ",
    "write_off_cost_center": "Main - PTMJ",
    "disabled": 0,
    "payments": [
      { "mode_of_payment": "Cash", "default": 1, "allow_in_returns": 1 },
      { "mode_of_payment": "Bank Transfer", "default": 0, "allow_in_returns": 1 }
    ]
  }
}
```

> `name = "POS Toko Cikarang"` → simpan nilai ini; dipakai untuk operasi berikutnya (semuanya
> dikirim di body).

> Variants berikut cukup dijadikan nilai **`doc`** pada `frappe.client.insert` di atas
> (contoh request lengkap), lalu baca `name` dari respons `message`.

**Varian A — POS khusus user tertentu (kasir):**

```json
{
  "doctype": "POS Profile",
  "name": "POS Kasir Cikarang",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "warehouse": "Toko Cikarang - PTMJ",
  "write_off_account": "Round Off - PTMJ",
  "write_off_cost_center": "Main - PTMJ",
  "payments": [ { "mode_of_payment": "Cash", "default": 1, "allow_in_returns": 1 } ],
  "applicable_for_users": [
    { "user": "kasir.cikarang@perusahaan.co.id", "default": 1 }
  ]
}
```

**Contoh request penuh (`frappe.client.insert`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "POS Profile",
      "name": "POS Kasir Cikarang",
      "company": "PT Maju Jaya",
      "currency": "IDR",
      "warehouse": "Toko Cikarang - PTMJ",
      "write_off_account": "Round Off - PTMJ",
      "write_off_cost_center": "Main - PTMJ",
      "payments": [ { "mode_of_payment": "Cash", "default": 1, "allow_in_returns": 1 } ],
      "applicable_for_users": [
        { "user": "kasir.cikarang@perusahaan.co.id", "default": 1 }
      ]
    }
  }'
```

> Hanya user di `applicable_for_users` yang bisa memakai profil ini. Kosong = semua user.

**Varian B — POS dengan filter item group:**

```json
{
  "doctype": "POS Profile",
  "name": "POS Kafe",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "warehouse": "Kafe - PTMJ",
  "write_off_account": "Round Off - PTMJ",
  "write_off_cost_center": "Main - PTMJ",
  "payments": [ { "mode_of_payment": "Cash", "default": 1, "allow_in_returns": 1 } ],
  "item_groups": [
    { "item_group": "Minuman" },
    { "item_group": "Makanan" }
  ]
}
```

> Layar POS hanya menampilkan item dari item group di atas (`description`: "Only show Items from
> these Item Groups"). Kosong = semua item.

**Varian C — tanpa update stock (tidak disarankan):**

```json
{
  "doctype": "POS Profile",
  "name": "POS Showroom (tanpa stok)",
  "company": "PT Maju Jaya",
  "currency": "IDR",
  "update_stock": 0,
  "write_off_account": "Round Off - PTMJ",
  "write_off_cost_center": "Main - PTMJ",
  "payments": [ { "mode_of_payment": "Cash", "default": 1, "allow_in_returns": 1 } ]
}
```

> Dengan `update_stock = 0`, `warehouse` tidak wajib. Umumnya TIDAK dipakai di aplikasi ini —
> lihat §2.1 no. 2.

### 4.2 READ (satu record) & total count

**Langkah 1 — Ambil detail POS Profile (`frappe.client.get`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "POS Profile",
    "name": "POS Toko Cikarang"
  }'
```

Respons `message` berisi seluruh field (termasuk child table `payments`/`item_groups`/
`customer_groups`/`applicable_for_users`).

**Total count — `frappe.client.get_count`** (untuk pagination / lazy loading):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_count \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "POS Profile",
    "filters": [["disabled","=",0]]
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": 3
}
```

> Respons berupa `message` = jumlah record yang cocok.

### 4.3 READ (daftar) — `frappe.client.get_list`

```bash
# Daftar — halaman 1, filter company + non-disabled
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "POS Profile",
    "fields": ["name","company","currency","warehouse","customer","disabled"],
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
      "name": "POS Toko Cikarang",
      "company": "PT Maju Jaya",
      "currency": "IDR",
      "warehouse": "Toko Cikarang - PTMJ",
      "customer": "Walk-In Customer",
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
      "doctype": "POS Profile",
      "name": "POS Toko Cikarang",
      "company": "PT Maju Jaya",
      "currency": "IDR",
      "warehouse": "Toko Cikarang 2 - PTMJ",
      "customer": "Walk-In Customer",
      "selling_price_list": "Standard Selling",
      "income_account": "Sales - PTMJ",
      "expense_account": "Cost of Goods Sold - PTMJ",
      "cost_center": "Main - PTMJ",
      "write_off_account": "Round Off - PTMJ",
      "write_off_cost_center": "Main - PTMJ",
      "allow_discount_change": 0,
      "payments": [
        { "mode_of_payment": "Cash", "default": 1, "allow_in_returns": 1 },
        { "mode_of_payment": "Bank Transfer", "default": 0, "allow_in_returns": 1 }
      ]
    }
  }'
```

**Contoh respons (HTTP 200):** objek `message` berisi dokumen terbaru (field berubah).

> **Catatan:** `save` membangun ulang dokumen dari dict — kirim dokumen yang konsisten/lengkap
> (idealnya hasil GET yang diubah; child table seperti `payments` berlaku **replace-all**). Untuk
> perubahan kecil gunakan `frappe.client.set_value` (lihat §4.5).

> **Catatan:** jangan mengubah `company` begitu profil dipakai (§2.1 no. 2). Mengganti `warehouse`
> hanya berdampak pada transaksi **baru** — POS Invoice yang sudah dibuat tetap memakai warehouse
> lamanya.

### 4.5 Non-aktifkan (disarankan) — `frappe.client.set_value`

**Aturan:** data POS Profile **tidak boleh dihapus**, hanya dinon-aktifkan. Non-aktif adalah
perubahan satu field → pakai `set_value`:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "POS Profile",
    "name": "POS Kasir Cikarang",
    "fieldname": { "disabled": 1 }
  }'
```

**Contoh respons (HTTP 200):** objek `message` terbaru dengan `"disabled": 1`.

Untuk mengaktifkan kembali: `fieldname: { "disabled": 0 }`.

> **Efek non-aktif:** profil tidak muncul sebagai pilihan saat kasir membuka POS. Riwayat POS
> Invoice tetap utuh.

> ⚠️ **Jangan gunakan `frappe.client.delete`** — selain menghilangkan default setting kasir secara
> tiba-tiba, pola master data aplikasi ini adalah "jangan hapus". Non-aktifkan saja.

---

## 5. GET pendukung UI

### 5.1 GET Company — dropdown UI

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Company",
    "fields": ["name","abbr","default_currency"],
    "filters": [["is_group","=",0]],
    "limit_page_length": 0
  }'
```

### 5.2 GET Warehouse (gudang toko) — dropdown UI

Hanya gudang ledger (`is_group=0`), non-disabled, di company yang sama:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "fields": ["name","warehouse_name","warehouse_type"],
    "filters": [["company","=","PT Maju Jaya"],["is_group","=",0],["disabled","=",0]],
    "limit_page_length": 0
  }'
```

### 5.3 GET Price List — dropdown UI

Hanya price list jual (`selling=1`) yang aktif:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Price List",
    "fields": ["name","currency","enabled"],
    "filters": [["enabled","=",1],["selling","=",1]],
    "limit_page_length": 0
  }'
```

### 5.4 GET / CREATE Mode of Payment — dropdown UI

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Mode of Payment",
    "fields": ["name","type"],
    "filters": [["enabled","=",1]],
    "limit_page_length": 0
  }'
```

**CREATE metode baru** (`frappe.client.insert`; child table `accounts` memetakan akun per company):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Mode of Payment",
      "name": "QRIS",
      "type": "Bank",
      "enabled": 1,
      "accounts": [
        { "company": "PT Rapupa Guna Teknologi", "default_account": "1120.001 - Rekening Bank Utama - APS" }
      ]
    }
  }'
```

> `Mode of Payment` adalah master kecil (field: `type` Select Cash/Bank/General/Phone, `enabled`).
> `POS Payment Method` di profil cukup mereferensikan `mode_of_payment` ini. **`default_account`
> (child `accounts`) mengikuti tipe `type`-nya — ambil akun via §5.12 (resolve per `type`, jangan
> hardcode), lalu isi hasil `name`-nya di sini.** Untuk contoh di atas (`type = Bank`) akun default
> = `1120.001 - Rekening Bank Utama - APS` sesuai baris Bank di §5.12.

### 5.5 GET Account — dropdown UI

Untuk `income_account` / `expense_account` / `write_off_account` / `account_for_change_amount`,
ambil akun non-group di company yang sama:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Account",
    "fields": ["name","account_type","root_type"],
    "filters": [["is_group","=",0],["company","=","PT Maju Jaya"]],
    "limit_page_length": 0
  }'
```

### 5.6 GET Cost Center — dropdown UI

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Cost Center",
    "fields": ["name"],
    "filters": [["company","=","PT Maju Jaya"],["is_group","=",0],["disabled","=",0]],
    "limit_page_length": 0
  }'
```

### 5.7 GET Sales Taxes and Charges Template — dropdown UI

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Sales Taxes and Charges Template",
    "fields": ["name","company","is_default"],
    "filters": [["company","=","PT Maju Jaya"],["disabled","=",0]],
    "limit_page_length": 0
  }'
```

### 5.8 GET Item Group / Customer Group / User — filter & user

```bash
# Item Group (untuk isi item_groups) — hanya node daun
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Group",
    "fields": ["name"],
    "filters": [["is_group","=",0]],
    "limit_page_length": 0
  }'

# Customer Group
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Customer Group",
    "fields": ["name"],
    "filters": [["is_group","=",0]],
    "limit_page_length": 0
  }'

# User (untuk isi applicable_for_users) — hanya user internal dengan hak buka POS
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "User",
    "fields": ["name","full_name"],
    "filters": [["enabled","=",1],["user_type","=","System User"]],
    "limit_page_length": 0
  }'
```

### 5.9 GET Account — resolve `write_off_account` ter-kunci

`write_off_account` dikunci ke record akun **`5510.009 - Selisih Pembayaran Customer`**. Karena
`name` akun = `{account_name} - {abbr}` (suffix abbr company), frontend **tidak boleh hardcode**
nama lengkapnya — resolve via `account_number`:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Account",
    "fields": ["name","account_name","account_number","account_type"],
    "filters": [["account_number","=","5510.009"]],
    "limit_page_length": 1
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    {
      "name": "5510.009 - Selisih Pembayaran Customer - APS",
      "account_name": "5510.009 - Selisih Pembayaran Customer",
      "account_number": "5510.009",
      "account_type": "Expense Account"
    }
  ]
}
```

> `message` terisi → gunakan `name` hasilnya sebagai `write_off_account` (read-only di form dan
> selalu dikirim pada CREATE/PUT). `message` kosong (`[]`) → akun belum ada di sistem → blokir
> CREATE. Catatan: `account_number` tidak unik global — bila ada lebih dari satu hasil, pilih yang
> `company`-nya sama dengan POS Profile (lihat §5.5).

### 5.10 GET Cost Center — resolve `write_off_cost_center` ter-kunci

`write_off_cost_center` dikunci ke record cost center **`Main`** (default Chart of Accounts; `name` =
`{cost_center_name} - {abbr}` → `Main - APS`). Sama seperti akun, frontend **tidak boleh hardcode**
nama lengkapnya — resolve via `cost_center_name` (filter hanya cost center aktif, non-group):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Cost Center",
    "fields": ["name","cost_center_name","is_group","disabled"],
    "filters": [["cost_center_name","=","Main"],["is_group","=",0],["disabled","=",0]],
    "limit_page_length": 1
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    {
      "name": "Main - APS",
      "cost_center_name": "Main",
      "is_group": 0,
      "disabled": 0
    }
  ]
}
```

> `message` terisi → gunakan `name` hasilnya sebagai `write_off_cost_center` (read-only di form dan
> selalu dikirim pada CREATE/PUT). `message` kosong (`[]`) → cost center belum ada → blokir CREATE.
> Catatan: `cost_center_name` tidak unik global — bila ada lebih dari satu hasil, pilih yang
> `company`-nya sama dengan POS Profile (lihat §5.6).

### 5.11a GET / CREATE Letter Head & logo outlet — header cetak struk/invoice (opsional)

`letter_head` di POS Profile menunjuk ke record **`Letter Head`** (doctype **Frappe**, modul Printing)
yang dipakai sebagai kop/header saat cetak **POS Invoice / Sales Invoice**. Saat transaksi POS, nilai
`letter_head` dari POS Profile **disalin ke invoice**; saat print, `content`/`footer` Letter Head
itu dirender sebagai header dokumen. Bila `letter_head` POS Profile **kosong**, ERPNext otomatis
memakai **Letter Head default** (`is_default=1`, umumnya `Company Letterhead - Grey`).

**Dua opsi untuk user** — frontend menyediakan pilihan:

**Opsi A — "Generator" (header otomatis dari Company, tanpa membuat Letter Head).**
Bila user memilih ini, `letter_head` POS Profile **dikosongkan** (atau dibiarkan mengikuti
`Company.default_letter_head`) → header cetak otomatis berisi **nama + alamat + email/telepon
company + logo outlet** (template bawaan `Company Letterhead` memuat identitas company via Jinja).
Yang perlu dilakukan frontend: **upload logo outlet ke `Company.company_logo`** dan memastikan data
Company terisi. Contoh:

```bash
# Langkah 1 — upload logo outlet (public agar bisa dirender di PDF/struk)
curl -X POST https://site-anda.com/api/method/frappe.client.attach_file \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Company",
    "docname": "PT Rapupa Guna Teknologi",
    "filename": "logo-outlet.png",
    "filedata": "<isi file dalam base64>",
    "is_private": 0
  }'
```

```bash
# Langkah 2 — set company_logo = file_url hasil upload (frappe.client.set_value)
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Company",
    "name": "PT Rapupa Guna Teknologi",
    "fieldname": { "company_logo": "/files/logo-outlet.png" }
  }'
```

> Respons `attach_file` `message` berisi dokumen `File` (`name`, `file_name`, `file_url`).
> Alternatif tanpa upload: kirim `file_url` yang sudah ada di server (tanpa `filedata`).
> `filedata` = isi file (base64); `is_private=0` → file publik (`/files/...`). Data lain yang muncul
di header (nama/alamat/email/telepon) diambil otomatis dari Company (`company_name`, `phone_no`,
`email`, `website`) + alamat company (Address ter-link Company). `letter_head` POS Profile tidak
perlu diisi → default terpakai. Role tulis: `System Manager`.

**Opsi B — Custom Letter Head (logo outlet = gambar PER Letter Head).**
Bila user memilih ini, frontend **membuat Letter Head sendiri** lalu mengarahkan `letter_head` POS
Profile ke LH tsb. **Logo outlet disimpan di field `image` Letter Head** (khusus LH ini) — **BUKAN**
di `Company.company_logo` (logo company hanya boleh diubah lewat fitur Company). Letter Head bersifat
**global (tanpa company)**; `name` = `letter_head_name` (unique, reqd). `attach_file` butuh record
yang **sudah ada** → urutannya: **CREATE Letter Head dulu, baru lampirkan logo ke LH tsb**.

**Varian B1 — `source = "Image"` (disarankan): logo = field `image` LH, `content` dibuat otomatis.**

Langkah 1 — CREATE Letter Head (`name` hasil = `Kop Toko Cikarang`; catat utk Langkah 2):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Letter Head",
      "letter_head_name": "Kop Toko Cikarang",
      "source": "Image",
      "disabled": 0
    }
  }'
```

Langkah 2 — Lampirkan logo outlet **ke LH tsb** (`docfield = "image"`):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.attach_file \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Letter Head",
    "docname": "Kop Toko Cikarang",
    "docfield": "image",
    "filename": "logo-outlet.png",
    "filedata": "<isi file dalam base64>",
    "is_private": 0
  }'
```

> `docfield: "image"` membuat `attach_file` mengisi field `image` LH dgn `file_url` lalu menyimpan LH.
> Karena `source = "Image"`, ERPNext otomatis meng-generate `content` dari logo tsb
> (`set_image_as_html`); opsional bisa set `align` / `image_width` / `image_height` di LH. Respons
> `message` = objek `File` (`name`, `file_name`, `file_url`). **Logo tersimpan per Letter Head di
> field `image` LH — `Company.company_logo` tidak disentuh.**

**Varian B2 — `source = "HTML"`: isi `content` & `footer` sendiri (logo opsional, disisipkan manual).**

Langkah 1 — CREATE Letter Head (`source = "HTML"`):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Letter Head",
      "letter_head_name": "Kop Toko Cikarang",
      "source": "HTML",
      "content": "<h3>Outlet Cikarang</h3>",
      "footer": "<p>Jl. Contoh No. 1 — Telp 021-0000</p>",
      "disabled": 0
    }
  }'
```

Langkah 2 — (opsional) Upload logo outlet sbg file publik ke LH tsb (tanpa `docfield`), ambil
`file_url` dari respons `message`:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.attach_file \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Letter Head",
    "docname": "Kop Toko Cikarang",
    "filename": "logo-outlet.png",
    "filedata": "<isi file dalam base64>",
    "is_private": 0
  }'
```

Langkah 3 — Perbarui `content` agar memuat logo (`src` = `file_url` hasil Langkah 2, bukan nama file
lokal) — `frappe.client.save`:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Letter Head",
      "name": "Kop Toko Cikarang",
      "source": "HTML",
      "content": "<div style=\"text-align:left\"><img src=\"/files/logo-outlet.png\" style=\"max-width:90px\"></div><h3>Outlet Cikarang</h3>",
      "footer": "<p>Jl. Contoh No. 1 — Telp 021-0000</p>",
      "disabled": 0
    }
  }'
```

> Pada `source = "HTML"`, logo outlet direferensikan via `<img src="{file_url}">` di dalam `content`;
> file publik tsb tersimpan di `tabFile` (ter-attach ke LH sbg lampiran). `Company.company_logo`
> tetap tidak digunakan di Opsi B.

**Contoh respons (HTTP 200):** objek `message` berisi dokumen Letter Head; `name` =
`Kop Toko Cikarang`. Simpan `name` → kirim sebagai `letter_head` pada CREATE/PUT POS Profile:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "POS Profile",
    "name": "POS Toko Cikarang",
    "fieldname": { "letter_head": "Kop Toko Cikarang" }
  }'
```

**GET daftar Letter Head (dropdown UI) — sudah termasuk `content`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Letter Head",
    "fields": ["name","letter_head_name","is_default","disabled","source","content"],
    "filters": [["disabled","=",0]],
    "limit_page_length": 0
  }'
```

> `content` sengaja disertakan dalam **1x panggilan yang sama** (dropdown + deteksi) supaya
> frontend tidak perlu memanggil API 2x. `frappe.client.get_list` hanya mengembalikan field yang
> diminta di `fields` — jadi cukup tambahkan `"content"` di sini. Bila suatu saat payload ingin
> dipangkas (LH banyak & `content` panjang), field ini bisa dilepas; konsekuensinya deteksi
> otomatis-vs-custom (sub-bagian berikut) tidak bisa jalan tanpa `content`.

**Mendeteksi: apakah Letter Head memakai data Company (header otomatis) atau tidak?**

Tidak ada field boolean khusus di Letter Head utk menandai ini — penandanya ada di isi **`content`**.
Saat print, ERPNext **merender `content` sebagai template Jinja** dengan `doc` (invoice) sebagai
konteks. Karena itu:

- **LH "memakai data Company"** (seperti template bawaan `Company Letterhead` / `Company Letterhead - Grey`,
  dan Opsi A) → `content`-nya memuat **referensi Jinja ke Company**, contoh token yang bisa dicek:
  `{{ doc.company }}`, `frappe.db.get_value("Company", ...)`, `doc.company_address`,
  `company_logo`, `frappe.utils.get_url(...)`.
- **LH "custom / tidak memakai data Company"** (Opsi B, `source = HTML` statis atau `source = Image`)
  → `content`-nya **HTML statis** (teks / `<img src="...">` logo outlet), tanpa referensi Jinja tsb.

**Aturan deteksi (frontend)** — cek `content` dari hasil **GET daftar di atas** (cukup satu
panggilan; frontend mengecek token dengan `includes()` / regex):

| Kondisi `content` | Kesimpulan |
|---|---|
| Mengandung `frappe.db.get_value("Company"` ATAU `doc.company` ATAU `doc.company_address` ATAU `company_logo` | **Memakai data Company** (header otomatis; butuh Company/`company_logo` terisi) |
| Tidak ada token tsb (HTML statis / hanya `<img>` logo outlet) | **Custom / per Letter Head** (logo & identitas dari LH itu sendiri) |

> Catatan: `content` yg di-return berupa string HTML mentah (masih memuat sintaks Jinja `{{ ... }}`,
> belum dirender) — jadi token Jinja tetap terlihat & bisa dicek. Bila klien memakai **satu company
> per outlet**, logo per outlet cukup di-set di `Company.company_logo` (Opsi A); bila semua outlet
> satu company namun beda logo, gunakan Opsi B (satu Letter Head per outlet, `letter_head` diisi
> per POS Profile).

> **Ringkasan field Letter Head:** `letter_head_name` (= `name`, unique, reqd), `source`
> (`Image`/`HTML`), `image` (logo; `source=Image`), `content` (Header HTML), `footer` (+
> `footer_source`), `is_default`, `disabled`. Non-aktif LH = `disabled: 1` (tidak muncul di
> dropdown). `is_default` = LH default yang dipakai bila `letter_head` kosong (maks. 1 aktif;
> tidak boleh `disabled`+`is_default` sekaligus). Role: baca `Desk User`; buat/ubah
> `System Manager`. `content`/`footer` mendukung Jinja (mis. `{{ doc.company }}`,
> `{{ doc.company_address }}`) — dipakai Opsi A/template bawaan.

### 5.11b GET / CREATE Terms and Conditions — isi printout (kebijakan retur/catatan)

`tc_name` di POS Profile menunjuk ke record **`Terms and Conditions`** (modul Setup). Ini tempat
menyimpan **kebijakan retur / catatan invoice** yang dicetak di struk — frontend cukup
membuat/master data ini, lalu set `tc_name` pada POS Profile (lihat baris `tc_name` di §2,
Opsional). `name` = `title` (autoname `field:title`, unique).

**GET daftar (dropdown UI)** — hanya yang berlaku untuk penjualan (`selling=1`), aktif:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Terms and Conditions",
    "fields": ["name","title","terms","disabled"],
    "filters": [["selling","=",1],["disabled","=",0]],
    "limit_page_length": 0
  }'
```

**CREATE baru** (`frappe.client.insert`):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Terms and Conditions",
      "title": "Kebijakan Retur",
      "terms": "<p>Barang yang sudah dibeli dapat dikembalikan dalam 7 hari kerja dengan struk asli.</p>",
      "selling": 1,
      "buying": 0
    }
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": {
    "name": "Kebijakan Retur",
    "title": "Kebijakan Retur",
    "terms": "<p>Barang yang sudah dibeli dapat dikembalikan dalam 7 hari kerja dengan struk asli.</p>",
    "selling": 1,
    "buying": 0,
    "disabled": 0
  }
}
```

> **Cara pakai:** simpan `name` (= `title`) → kirim sebagai `tc_name` pada CREATE/PUT POS Profile.
> Saat POS Invoice dibuat, backend otomatis mengisi `terms` dari record ini → tercetak di struk.
> `terms` mendukung Jinja (mis. `{{ name }}`, `{{ transaction_date }}`). Role tulis:
> `Sales Master Manager` / `Accounts Manager` / `System Manager`.

### 5.12 GET Account — resolve akun default utk Mode of Payment (per `type`)

Saat membuat **Mode of Payment** baru (CREATE di §5.4), field `accounts[].default_account` diisi
akun default utk company tsb. Akun **tidak boleh hardcode** (`name` = `{account_number} - {account_name}
- {abbr}`, suffix abbr company) — resolve lewat `account_number` + `company` (logika sama dgn
resolve `write_off_account` di §5.9). Pemetaan `type` → akun yang dipakai:

| `type` Mode of Payment | `account_number` | `account_name` (yg dicari) | `account_type` akun |
|---|---|---|---|
| `Cash` | `1111.002` | `Kas Besar` | `Cash` |
| `Bank` | `1120.001` | `Rekening Bank Utama` | `Bank` |
| `General` | `5110.019` | `Biaya Penjualan Lain Lain` | (boleh kosong) |
| `Phone` | `1132.002` | `Piutang Payment Gateway` | (boleh kosong) |

**Contoh — resolve akun utk `type = Cash`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Account",
    "fields": ["name","account_name","account_number","account_type","is_group"],
    "filters": [
      ["company","=","PT Rapupa Guna Teknologi"],
      ["account_number","=","1111.002"],
      ["is_group","=",0]
    ],
    "limit_page_length": 1
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    {
      "name": "1111.002 - Kas Besar - APS",
      "account_name": "Kas Besar",
      "account_number": "1111.002",
      "account_type": "Cash",
      "is_group": 0
    }
  ]
}
```

> Ulangi pola yang sama utk tipe lain dgn `account_number` sesuai tabel (`1120.001` utk `Bank`,
> `5110.019` utk `General`, `1132.002` utk `Phone`). `account_number` tidak unik global — sertakan
> `company`. Bila `message` kosong (`[]`) → akun belum ada utk company tsb → blokir CREATE Mode of
> Payment. `account_type` boleh kosong utk `General`/`Phone` (tetap valid sbg `default_account`).
>
> **Cara pakai:** hasil `name` dari resolve (mis. `1111.002 - Kas Besar - APS`) diisi ke
> `accounts[].default_account` pada CREATE Mode of Payment (§5.4) — sesuai `type` yg dipilih.

---

## 6. Penanganan error umum

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | Role tidak punya akses | `{"message": "Not permitted"}` |
| 404 | Resource tidak ditemukan | `{"exc_type":"DoesNotExistError","message":"Resource Not Found"}` |
| 417 | Field wajib kosong (`reqd`): `name`/`company`/`currency`/`payments`/`warehouse`/`write_off_account`/`write_off_cost_center` | `{"exc_type":"MandatoryError","message":"warehouse is mandatory"}` |
| 417 | Duplikat `name` | `{"exc_type":"DuplicateEntryError","message":"POS Toko Cikarang already exists"}` |
| 417 | Validasi gagal (mis. warehouse beda company, akun salah company) | `{"exc_type":"ValidationError","message":"Warehouse Toko Cikarang - PTMJ does not belong to Company PT Maju Jaya"}` |

> **Catatan:** karena seluruh pemanggilan memakai `/api/method/...`, hasil sukses dibungkus di
> `message` (bukan `data`). Body error tetap berbentuk `{ "exc_type": ..., "exception": ..., "message": ... }`.

---

## 7. Koleksi Postman

Seluruh pemanggilan di atas mengikuti koleksi Postman yang sama:
`docs/postman/postman_erpnext_api.json` (Collection v2.1) — berisi seluruh modul API ERPNext
(OAuth 2.0 + Supplier + Customer + Contact + Address + Lead + Employee + User + Warehouse + POS Profile).
Request POS Profile sudah masuk ke koleksi sebagai folder **`9. POS Profile`** (34 request: 9.1–9.33
sesuai contoh di dokumen ini) — `pos_profile_id`/`pos_profile_name`/`tax_servis_template`/
`tax_servis_ppn_template`/`tax_tip_template`/`tax_surcharge_template`/`tax_servis_tip_ppn_template`
tersedia sebagai variabel koleksi.

---

## 8. Studi kasus — setup POS multi-toko, kasir, & pajak (tanpa desk)

**8a — Setup POS multi-toko & kasir**

Tujuan: **dua toko** — *Toko Cikarang* dan *Toko Bekasi* — masing-masing punya warehouse + POS
Profile sendiri. Kasir toko A **hanya** bisa membuka POS di toko A (tidak melihat toko B).
Pendekatan: satu POS Profile per toko + `applicable_for_users` + `User Permission` per toko.
Setup role/user kasir mengikuti **[prd_user.md §8.5](../contact/prd_user.md)** — bagian ini fokus
ke sisi **POS Profile**.

**Langkah 0 — Prasyarat (master data):**

1. **Warehouse toko** (ledger, `warehouse_type=Store`): `Toko Cikarang - PTMJ` & `Toko Bekasi - PTMJ`
   ([prd_warehouse.md §4.1 Varian B](./prd_warehouse.md)).
2. **Price List** `Standard Selling` (currency `IDR`), Item + Item Group (mis. `Products`), Customer
   `Walk-In Customer`.
3. **Akun & cost center**: `Sales - PTMJ`, `Cost of Goods Sold - PTMJ`, `Round Off - PTMJ`;
   cost center `Main - PTMJ` (sudah ada di Chart of Accounts default).
4. **User kasir**: `kasir.cikarang@perusahaan.co.id` & `kasir.bekasi@perusahaan.co.id` dengan role
   `Kasir` ([prd_user.md §8.5](../contact/prd_user.md)).

**Langkah 1 — Buat POS Profile Toko Cikarang (`frappe.client.insert`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "POS Profile",
      "name": "POS Toko Cikarang",
      "company": "PT Maju Jaya",
      "currency": "IDR",
      "warehouse": "Toko Cikarang - PTMJ",
      "customer": "Walk-In Customer",
      "selling_price_list": "Standard Selling",
      "income_account": "Sales - PTMJ",
      "expense_account": "Cost of Goods Sold - PTMJ",
      "cost_center": "Main - PTMJ",
      "write_off_account": "Round Off - PTMJ",
      "write_off_cost_center": "Main - PTMJ",
      "payments": [ { "mode_of_payment": "Cash", "default": 1, "allow_in_returns": 1 } ],
      "item_groups": [ { "item_group": "Products" } ],
      "applicable_for_users": [
        { "user": "kasir.cikarang@perusahaan.co.id", "default": 1 }
      ]
    }
  }'
```

**Langkah 2 — Buat POS Profile Toko Bekasi (`frappe.client.insert`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "POS Profile",
      "name": "POS Toko Bekasi",
      "company": "PT Maju Jaya",
      "currency": "IDR",
      "warehouse": "Toko Bekasi - PTMJ",
      "customer": "Walk-In Customer",
      "selling_price_list": "Standard Selling",
      "income_account": "Sales - PTMJ",
      "expense_account": "Cost of Goods Sold - PTMJ",
      "cost_center": "Main - PTMJ",
      "write_off_account": "Round Off - PTMJ",
      "write_off_cost_center": "Main - PTMJ",
      "payments": [ { "mode_of_payment": "Cash", "default": 1, "allow_in_returns": 1 } ],
      "item_groups": [ { "item_group": "Products" } ],
      "applicable_for_users": [
        { "user": "kasir.bekasi@perusahaan.co.id", "default": 1 }
      ]
    }
  }'
```

> `applicable_for_users` membatasi siapa yang bisa memilih profil ini di layar POS. Kasir Cikarang
> hanya muncul di profil toko Cikarang → **tidak bisa membuka POS toko Bekasi**.

**Langkah 3 — Beri role `Kasir` akses baca ke dokumen yang dipakai layar POS:**

Role custom `Kasir` default **tidak punya akses apa pun**. Selain permission di
[prd_user.md §8.5](../contact/prd_user.md), kasir butuh minimal baca `POS Profile`, `Mode of
Payment`, dan `Price List` agar layar POS bisa dimuat:

```bash
curl -X POST "https://site-anda.com/api/method/frappe.core.page.permission_manager.permission_manager.update" \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "POS Profile", "role": "Kasir", "permlevel": 0, "ptype": "read", "value": 1 }'
curl -X POST "https://site-anda.com/api/method/frappe.core.page.permission_manager.permission_manager.update" \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Mode of Payment", "role": "Kasir", "permlevel": 0, "ptype": "read", "value": 1 }'
curl -X POST "https://site-anda.com/api/method/frappe.core.page.permission_manager.permission_manager.update" \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Price List", "role": "Kasir", "permlevel": 0, "ptype": "read", "value": 1 }'
```

> Ulangi pola yang sama (baca `read`) untuk master lain yang dibutuhkan layar POS: `Warehouse`,
> `Item Group`, `Customer Group`, `Account`, `Cost Center`, `Sales Taxes and Charges Template`,
> `Print Format` — atau cukup tambahkan role bawaan `Sales User`/`Accounts User` sebagai role kedua
> user kasir agar master tersebut terbaca (lebih sederhana bila kebijakan permission mengizinkan).

**Langkah 4 — (Opsional, lebih ketat) Batasi lingkup data per toko (User Permission):**

`applicable_for_users` membatasi **profil**, sedangkan `User Permission` membatasi **record**
(warehouse) yang bisa diakses user. Untuk memastikan kasir Cikarang tidak melihat data stok toko
Bekasi:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token_system_manager>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "User Permission",
      "user": "kasir.cikarang@perusahaan.co.id",
      "allow": "Warehouse",
      "for_value": "Toko Cikarang - PTMJ",
      "is_default": 1
    }
  }'
```

> Detail User Permission ada di **[prd_user.md §4.6](../contact/prd_user.md)**.

**Ringkasan setup multi-toko:**

| Aspek | Toko Cikarang | Toko Bekasi |
|---|---|---|
| Warehouse | `Toko Cikarang - PTMJ` | `Toko Bekasi - PTMJ` |
| POS Profile | `POS Toko Cikarang` | `POS Toko Bekasi` |
| `applicable_for_users` | `kasir.cikarang@perusahaan.co.id` | `kasir.bekasi@perusahaan.co.id` |
| User Permission (Warehouse) | `Toko Cikarang - PTMJ` | `Toko Bekasi - PTMJ` |

> **Catatan akhir:** bila klien hanya butuh **satu toko**, cukup Langkah 1 + 3 — satu POS Profile
> tanpa `applicable_for_users` berlaku untuk semua user, dan tanpa User Permission semua warehouse
> bisa diakses.

**8b — Biaya tambahan & pajak: servis, tip, employee surcharge (per POS Profile)**

`taxes_and_charges` adalah field **per POS Profile**, jadi tiap profil bebas menunjuk ke template
pajak yang berbeda. Konfigurasi di sini **tidak terbatas** pada "pajak" atau "servis" saja — user
bisa menambahkan **biaya tambahan lain** seperti **tip** atau **employee surcharge**. Semua biaya
tambahan memakai **akun yang sama**, `4310.000 - Pendapatan Service - APS` (income), dengan alur
identik seperti biaya servis: tarif **persen** (`rate`), dan masing-masing bebas **dikenakan pajak
atau tidak**. Baris pajaknya sendiri memakai `2142.000 - PPN Keluaran - APS` (tax); cost center
`Main - APS`.

**Jenis biaya tambahan yang didukung (semua via akun `4310.000 - Pendapatan Service - APS`):**

| Jenis biaya | `account_head` | Contoh `rate` | `description` |
|---|---|---|---|
| Servis | `4310.000 - Pendapatan Service - APS` | `10` (%) | `Biaya Servis` |
| Tip | `4310.000 - Pendapatan Service - APS` | `5` (%) | `Tip` |
| Employee Surcharge | `4310.000 - Pendapatan Service - APS` | `3` (%) | `Employee Surcharge` |

> Semua jenis biaya tambahan **memakai akun yang sama** (`4310.000`) — pembedaannya hanya pada
> `description`. Tiap jenis bisa **berdiri sendiri** maupun **digabung** dalam satu template
> (mis. Servis + Tip), dan masing-masing bisa **ikut** atau **tidak ikut** dasar PPN.

> Tiap kasus = 1 `Sales Taxes and Charges Template` terpisah; arahkan `taxes_and_charges` POS
> Profile ke nama template sesuai kasus (Langkah C). Detail CREATE penuh mengikuti pola Langkah
> A/B di bawah (kasus 1 & kasus 4). POS Profile yang di-set harus **se-company** dengan template
> (`PT Rapupa Guna Teknologi`).

**Langkah 0 — Resolve akun `account_head` (wajib sebelum CREATE template)**

`name` akun = `{account_number} - {account_name} - {abbr}` (contoh `4310.000 - Pendapatan Service - APS`),
jadi frontend **tidak boleh hardcode** nama lengkapnya. Ambil lewat `account_number` + company
(cara sama dengan resolve `write_off_account` di §5.9):

**a) Akun biaya tambahan (servis/tip/employee surcharge; akun income):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Account",
    "fields": ["name","account_name","account_number","account_type","is_group"],
    "filters": [
      ["company","=","PT Rapupa Guna Teknologi"],
      ["account_number","=","4310.000"],
      ["is_group","=",0]
    ],
    "limit_page_length": 1
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    {
      "name": "4310.000 - Pendapatan Service - APS",
      "account_name": "Pendapatan Service",
      "account_number": "4310.000",
      "account_type": "",
      "is_group": 0
    }
  ]
}
```

> `account_type` boleh kosong (opsional di ERPNext) — tetap valid sebagai `account_head`; yang
> wajib hanya akun **bukan group** (`is_group=0`) dan **se-company** dengan template/POS Profile.

**b) Akun PPN Keluaran (baris pajak):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Account",
    "fields": ["name","account_name","account_number","account_type","is_group"],
    "filters": [
      ["company","=","PT Rapupa Guna Teknologi"],
      ["account_number","=","2142.000"],
      ["is_group","=",0]
    ],
    "limit_page_length": 1
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    {
      "name": "2142.000 - PPN Keluaran - APS",
      "account_name": "PPN Keluaran",
      "account_number": "2142.000",
      "account_type": "Tax",
      "is_group": 0
    }
  ]
}
```

> Bila `message` kosong (`[]`) → akun belum ada untuk company tsb → blokir CREATE template.
> Catatan: `account_number` tidak unik global — filter juga `company` (seperti di atas). Cost
> center resolve via `cost_center_name = "Main"` (lihat §5.10) → `Main - APS`.

**Langkah A — Buat template `Servis 10%`** (1 baris; hanya servis). `name` template otomatis =
`{title} - {abbr}` → menjadi **`Servis 10% - APS`**:

```json
{
  "doc": {
    "doctype": "Sales Taxes and Charges Template",
    "title": "Servis 10%",
    "company": "PT Rapupa Guna Teknologi",
    "taxes": [
      { "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
        "description": "Biaya Servis", "rate": 10, "custom_rates": "[5, 10, 15]",
        "cost_center": "Main - APS" }
    ]
  }
}
```

> Baca `name` dari respons (`Servis 10% - APS`) — dipakai untuk `taxes_and_charges` (Langkah C).

**Langkah B — Buat template `Servis + PPN 11% (dasar termasuk servis)`** (2 baris; baris PPN memakai
`On Previous Row Total` agar dasar pajak = net + biaya servis). `name` otomatis =
**`Servis + PPN 11% (dasar termasuk servis) - APS`**:

```json
{
  "doc": {
    "doctype": "Sales Taxes and Charges Template",
    "title": "Servis + PPN 11% (dasar termasuk servis)",
    "company": "PT Rapupa Guna Teknologi",
    "taxes": [
      { "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
        "description": "Biaya Servis", "rate": 10, "custom_rates": "[5, 10, 15]",
        "cost_center": "Main - APS" },
      { "charge_type": "On Previous Row Total", "account_head": "2142.000 - PPN Keluaran - APS",
        "description": "PPN (dasar termasuk servis)", "rate": 11, "cost_center": "Main - APS" }
    ]
  }
}
```

**Langkah C — Set `taxes_and_charges` di masing-masing POS Profile** (satu field → `set_value`):

| POS Profile | `taxes_and_charges` |
|---|---|
| `POS Toko Cikarang` | `Servis 10% - APS` |
| `POS Toko Bekasi` | `Servis + PPN 11% (dasar termasuk servis) - APS` |

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "POS Profile",
    "name": "POS Toko Bekasi",
    "fieldname": { "taxes_and_charges": "Servis + PPN 11% (dasar termasuk servis) - APS" }
  }'
```

> POS Profile yang di-set harus **se-company** dengan template (`PT Rapupa Guna Teknologi`). Bila
> contoh §8a masih memakai `- PTMJ`, samakan company-nya saat implementasi agar validasi
> "template tidak se-company" tidak muncul.

**Contoh kombinasi — biaya tambahan & pajak:**

Berikut kombinasi yang bisa dipilih user (centang + % di form), lengkap dengan isi `taxes`
template dan nama `taxes_and_charges` yang harus diarahkan. Akun: `4310.000 - Pendapatan Service - APS`
(income) & `2142.000 - PPN Keluaran - APS` (tax); cost center: `Main - APS`.

| Kasus | Pilihan user | Baris `taxes` | `charge_type` | `taxes_and_charges` |
|---|---|---|---|---|
| 1 | Servis saja (10%) | 1 baris | `On Net Total` | `Servis 10% - APS` |
| 2 | Hanya pajak (11%) | 1 baris | `On Net Total` | `PPN 11% - APS` |
| 3 | Servis (10%) + pajak (11%), servis **tidak** masuk dasar | 2 baris independen | `On Net Total` + `On Net Total` | `Servis + PPN 11% - APS` |
| 4 | Servis (10%) + pajak (11%), **servis masuk** dasar pajak | 2 baris berurutan | `On Net Total` + `On Previous Row Total` | `Servis + PPN 11% (dasar termasuk servis) - APS` |
| 5 | Tip saja (5%) | 1 baris | `On Net Total` | `Tip 5% - APS` |
| 6 | Employee Surcharge saja (3%) | 1 baris | `On Net Total` | `Employee Surcharge 3% - APS` |
| 7 | Tip (5%) + pajak (11%), **tip masuk** dasar pajak | 2 baris berurutan | `On Net Total` + `On Previous Row Total` | `Tip + PPN 11% - APS` |
| 8 | Servis (10%) + Tip (5%) + pajak (11%), semua masuk dasar | 3 baris berurutan | `On Net Total` + `On Net Total` + `On Previous Row Total` | `Servis + Tip + PPN 11% - APS` |

**Isi `doc.taxes` per kasus:**

> `custom_rates` (field dari custom app **baseapp**, lihat sub-bagian berikut) dicontohkan di baris
> biaya tambahan; bentuknya **string JSON** (bukan list mentah), mis. `"[5, 10, 15]"`.

**Kasus 1 — Servis saja:**

```json
"taxes": [
  { "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
    "description": "Biaya Servis", "rate": 10, "custom_rates": "[5, 10, 15]",
    "cost_center": "Main - APS" }
]
```

**Kasus 2 — Hanya pajak:**

```json
"taxes": [
  { "charge_type": "On Net Total", "account_head": "2142.000 - PPN Keluaran - APS",
    "description": "PPN", "rate": 11, "cost_center": "Main - APS" }
]
```

**Kasus 3 — Servis + pajak (independen, servis tidak masuk dasar):**

```json
"taxes": [
  { "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
    "description": "Biaya Servis", "rate": 10, "custom_rates": "[5, 10, 15]",
    "cost_center": "Main - APS" },
  { "charge_type": "On Net Total", "account_head": "2142.000 - PPN Keluaran - APS",
    "description": "PPN (tanpa servis di dasar)", "rate": 11, "cost_center": "Main - APS" }
]
```

**Kasus 4 — Servis + pajak (servis masuk dasar pajak):**

```json
"taxes": [
  { "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
    "description": "Biaya Servis", "rate": 10, "custom_rates": "[5, 10, 15]",
    "cost_center": "Main - APS" },
  { "charge_type": "On Previous Row Total", "account_head": "2142.000 - PPN Keluaran - APS",
    "description": "PPN (dasar termasuk servis)", "rate": 11, "cost_center": "Main - APS" }
]
```

**Kasus 5 — Tip saja (5%):**

```json
"taxes": [
  { "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
    "description": "Tip", "rate": 5, "custom_rates": "[5, 10, 15]",
    "cost_center": "Main - APS" }
]
```

**Kasus 6 — Employee Surcharge saja (3%):**

```json
"taxes": [
  { "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
    "description": "Employee Surcharge", "rate": 3, "custom_rates": "[1, 2, 3]",
    "cost_center": "Main - APS" }
]
```

**Kasus 7 — Tip + pajak (tip masuk dasar pajak):**

```json
"taxes": [
  { "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
    "description": "Tip", "rate": 5, "custom_rates": "[5, 10, 15]",
    "cost_center": "Main - APS" },
  { "charge_type": "On Previous Row Total", "account_head": "2142.000 - PPN Keluaran - APS",
    "description": "PPN (dasar termasuk tip)", "rate": 11, "cost_center": "Main - APS" }
]
```

**Kasus 8 — Servis + Tip + pajak (semua masuk dasar pajak):**

```json
"taxes": [
  { "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
    "description": "Biaya Servis", "rate": 10, "custom_rates": "[5, 10, 15]",
    "cost_center": "Main - APS" },
  { "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
    "description": "Tip", "rate": 5, "custom_rates": "[5, 10, 15]",
    "cost_center": "Main - APS" },
  { "charge_type": "On Previous Row Total", "account_head": "2142.000 - PPN Keluaran - APS",
    "description": "PPN (dasar termasuk servis + tip)", "rate": 11, "cost_center": "Main - APS" }
]
```

> **Catatan penting:**
> - `account_head` pada tiap kasus harus **hasil resolve Langkah 0** (jangan hardcode) dan
>   se-company dengan template. `account_type` akun biaya tambahan boleh kosong — tetap valid.
> - Template per-company dan **boleh banyak per company** — tidak bentrok antar profil.
> - Mengubah % pajak = **update template-nya** (`frappe.client.save` dengan `taxes` baru; child
>   table berlaku **replace-all**, jadi baris `tabSales Taxes and Charges` milik template **diganti,
>   tidak menumpuk**). POS Profile sendiri tidak menambah baris pada tabel itu.
> - Yang membuat `tabSales Taxes and Charges` bertambah adalah **invoice per transaksi** (POS
>   Invoice/Sales Invoice menyalin baris pajak dari template) — itu normal, bukan konfigurasi.

**Field `custom_rates` — daftar pilihan persentase (custom app `baseapp`):**

`custom_rates` adalah **Custom Field yang dibuat oleh custom app `baseapp`** (fungsi
`ensure_sales_taxes_custom_fields` di `baseapp/utils.py`) pada **child table `Sales Taxes and
Charges`** — jadi melekat di **tiap baris `taxes[]`**, bukan di parent template. Spesifikasi:
`label` = `Rates`, `fieldtype` = **JSON**, `insert_after` = `rate`.

Fungsinya: menampung **daftar opsi persentase alternatif** (mis. `[5, 10, 15]`) yang ditampilkan UI
sebagai pilihan cepat, *selain* nilai default yang tersimpan di field `rate`. **Nilai yang benar-benar
dipakai untuk menghitung pajak/biaya tetap `rate`** — `custom_rates` **tidak** direferensikan oleh
logika kalkulasi server mana pun (murni bantu UI). Jadi bila user memilih salah satu opsi, frontend
harus **menulis nilai terpilih itu ke `rate`** juga.

**Aturan pengiriman (penting — fieldtype JSON):** karena `fieldtype = JSON`, Frappe menolak **list
mentah** dengan error `Value for Rates cannot be a list` (`frappe/model/base_document.py:548`).
Yang diterima hanya **string JSON** atau **object (dict)**. Jadi:

| Salah (ditolak) | Benar |
|---|---|
| `"custom_rates": [5, 10, 15]` | `"custom_rates": "[5, 10, 15]"` |

**Contoh payload (baris `taxes[]`, `custom_rates` sebagai string):**

```json
{ "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
  "description": "Biaya Servis", "rate": 10, "custom_rates": "[5, 10, 15]",
  "cost_center": "Main - APS" }
```

**Contoh respons (HTTP 200)** — perhatikan `custom_rates` **kembali sebagai string**, bukan array:

```json
{
  "message": {
    "name": "Tax POS Toko Cikarang - APS",
    "company": "PT Rapupa Guna Teknologi",
    "taxes": [
      {
        "charge_type": "On Net Total",
        "account_head": "4310.000 - Pendapatan Service - APS",
        "description": "Biaya Servis",
        "rate": 10,
        "custom_rates": "[5, 10, 15]",
        "cost_center": "Main - APS"
      }
    ]
  }
}
```

> **Untuk frontend:** `custom_rates` **selalu dibaca sebagai string**. Lakukan
> `JSON.parse(row.custom_rates)` untuk mendapatkan array-nya, dan `JSON.stringify(array)` sebelum
> mengirim. Nilai `null`/kosong berarti tidak ada opsi alternatif.

**Alur frontend — menyimpan konfigurasi biaya tambahan & pajak (CREATE & EDIT):**

Centang + nilai % di form **bukan** field POS Profile langsung — frontend harus **mewujudkannya
menjadi `Sales Taxes and Charges Template`** (baris `taxes`), lalu arahkan `taxes_and_charges`
POS Profile ke template itu. **Rekomendasi: satu template khusus per POS Profile** (nama
deterministik, mis. `Tax POS Toko Cikarang`) agar perubahan di satu profil **tidak menimpa**
profil lain. `name` otomatis = `{title} - {abbr}`.

**Aturan membentuk baris `taxes` dari centang:**

| Pilihan user | Baris `taxes` yang dibentuk |
|---|---|
| Biaya tambahan saja (servis/tip/employee surcharge) | 1 baris `On Net Total` (rate biaya) + `custom_rates` (opsional) |
| Hanya pajak | 1 baris: `On Net Total` (rate pajak) |
| Biaya tambahan + pajak (biaya **tidak** masuk dasar) | 2 baris `On Net Total` (biaya, pajak) |
| Biaya tambahan + pajak (**biaya masuk** dasar pajak) | 2 baris: biaya `On Net Total` + pajak `On Previous Row Total` |
| Beberapa biaya tambahan (mis. Servis + Tip) + pajak | 3 baris: tiap biaya `On Net Total`, lalu pajak `On Previous Row Total` (agar semua biaya masuk dasar) |

**Skenario 1 — Membuat POS Profile baru (biaya tambahan saja, mis. servis):**

1. **Pre-check** duplikat `name` (§4.1 Langkah 0).
2. **Resolve akun** biaya tambahan & PPN (Langkah 0 di atas).
3. **Buat template** `Tax POS Toko Cikarang` (jika belum ada) via `frappe.client.insert`:

```json
{
  "doc": {
    "doctype": "Sales Taxes and Charges Template",
    "title": "Tax POS Toko Cikarang",
    "company": "PT Rapupa Guna Teknologi",
    "taxes": [
      { "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
        "description": "Biaya Servis", "rate": 10, "custom_rates": "[5, 10, 15]",
        "cost_center": "Main - APS" }
    ]
  }
}
```

4. **CREATE POS Profile** — set `taxes_and_charges` = `name` hasilnya (`Tax POS Toko Cikarang - APS`)
   langsung di payload `doc` (atau via `set_value` setelahnya). Tersimpan; invoice toko ini
   otomatis kena biaya servis.

**Skenario 2 — Mengedit POS Profile lama (servis saja → servis + pajak, servis masuk dasar pajak):**

1. Frontend menghitung `taxes` baru = 2 baris (servis `On Net Total` + pajak `On Previous Row Total`).
2. **Update template milik profil tsb** via `frappe.client.save` dengan `taxes` baru (child table
   **replace-all** — baris lama dihapus, baris baru di-insert):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Sales Taxes and Charges Template",
      "name": "Tax POS Toko Cikarang - APS",
      "company": "PT Rapupa Guna Teknologi",
      "taxes": [
        { "charge_type": "On Net Total", "account_head": "4310.000 - Pendapatan Service - APS",
          "description": "Biaya Servis", "rate": 10, "custom_rates": "[5, 10, 15]",
          "cost_center": "Main - APS" },
        { "charge_type": "On Previous Row Total", "account_head": "2142.000 - PPN Keluaran - APS",
          "description": "PPN (dasar termasuk servis)", "rate": 11, "cost_center": "Main - APS" }
      ]
    }
  }'
```

3. `taxes_and_charges` di POS Profile **tetap** menunjuk ke template yang sama → **tidak perlu**
   update POS Profile. Perubahan langsung berlaku untuk transaksi baru toko tsb.

**Jawaban 3 pertanyaan kunci:**

| Pertanyaan | Jawaban |
|---|---|
| Apakah membuat template baru? | **Tidak** (pola satu-template-per-profil) — frontend cukup `save` (update) template milik profil tsb. Template baru hanya dibuat saat pertama kali. |
| Mengupdate nilai biaya/pajak lama atau membuat data baru? | **Update (replace-all)** — baris lama dihapus, baris baru di-insert sesuai centang terbaru. Tidak menumpuk data lama. |
| Menimpa nilai di POS Profile lain? | **Tidak**, selama tiap profil punya template sendiri (dedicated). Hanya bila dua profil **berbagi satu template** yang sama, update akan memengaruhi keduanya — karena itu hindari berbagi template antar profil. |

> **Alternatif nama-per-konfigurasi:** bila sengaja membuat template baru tiap kombinasi berubah
> (mis. `Servis 10%`, `Tip 5%`, `Employee Surcharge 3%`, `PPN 11%`,
> `Servis + PPN 11%`, `Servis + PPN 11% (dasar termasuk servis)`),
> aman terhadap profil lain selama nama unik, tetapi **menumpuk template** dan wajib
> `set_value taxes_and_charges` ke nama baru setiap berubah. Rekomendasi tetap
> **satu-template-per-profil + update**.




