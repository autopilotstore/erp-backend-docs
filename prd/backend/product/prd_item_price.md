# PRD — REST API Doctype Price List & Item Price (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Price List** (daftar/kategori harga) dan
> **Item Price** (baris harga per item) di ERPNext (Frappe), diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Stock (ERPNext) — path `erpnext.stock.doctype.price_list`, `erpnext.stock.doctype.item_price`
- **Doctype:** `Price List` (master kategori harga) + `Item Price` (nilai harga)
- **Versi API:** `/api/method/...` (API v1) — method whitelisted `frappe.client.*`, `name` dikirim di **body**
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

> Dokumen ini adalah lanjutan **[prd_item.md](./prd_item.md)** §9 — bagian **harga produk** yang di sana
> disebut "PRD Price List terpisah". Item sendiri (master produk) tetap mengacu ke dokumen tersebut.
> Seluruh aturan, pesan error, dan contoh di dokumen ini **sudah diverifikasi langsung** pada instance
> ERPNext **v16** (uji dibuat lalu di-*rollback*, sehingga tidak mengubah data dev).

> **Ringkas — dua doctype, satu alur:**
> 1. **`Price List`** = *daftar/kategori harga* (mis. `Standard Selling`, `Grosir`, `Member`).
>    `name = price_list_name`; menyimpan mata uang + penanda `selling`/`buying`.
> 2. **`Item Price`** = *baris harga*: (item + price list + rate + UOM) + masa berlaku, opsional khusus
>    customer/supplier. Satu item boleh punya banyak baris.
>
> Alur UI normal: **buat Price List** (§4.1) → *(opsional)* jadikan default di Selling/Buying Settings
> (§4.3) → **buat baris Item Price per item** (§4.5).

---

## 1. Ruang lingkup

| # | Endpoint (POST `/api/method/...`) | Operasi | `name` di |
|---|---|---|---|
| 1 | `frappe.client.insert` | Buat Price List baru (CREATE, §4.1) | body (`doc`) |
| 2 | `frappe.client.get` | Ambil detail 1 Price List (READ, §4.2) | body |
| 3 | `frappe.client.get_list` | Daftar Price List (READ list / dropdown, §4.2, §5.1) | body (filters) |
| 4 | `frappe.client.get_count` | Total Price List sesuai filter — pagination (§4.2) | body |
| 5 | `frappe.client.set_value` | Ubah 1 field Price List — non-aktif/aktif, jadikan default (§4.3) | body |
| 6 | `frappe.client.save` | Ubah Price List (UPDATE dokumen penuh, §4.3) | body (`doc`) |
| 7 | `frappe.rename_doc` | Ganti nama Price List (`name` + link Item Price ikut berubah, §4.4) | body |
| 8 | `frappe.client.insert` | Buat baris harga (CREATE Item Price, §4.5) | body (`doc`) |
| 9 | `frappe.client.get` | Ambil detail 1 baris harga (READ, §4.6) | body |
| 10 | `frappe.client.get_list` | Daftar harga per item / per price list (§4.6, §5.8) | body (filters) |
| 11 | `frappe.client.get_count` | Total baris harga sesuai filter (§4.6) | body |
| 12 | `frappe.client.set_value` | Ubah 1 field harga — rate, valid_upto, packing_unit (§4.7) | body |
| 13 | `frappe.client.save` | Ubah baris harga (UPDATE penuh, §4.7) | body (`doc`) |
| 14 | `frappe.client.delete` | Hapus baris harga (satu-satunya penghapusan yang diizinkan, §4.8) | body |
| 15 | *(otomatis backend)* | Item Price dibuat dari `Item.standard_rate` saat CREATE Item (§4.9) | — |
| 16 | `upload_file` + `Data Import` + `form_start_import` | Bulk update harga via CSV/Excel (§4.10) | body |
| 17 | `frappe.client.get_list` | Dropdown pendukung UI: Currency, Item, Customer, Supplier (§5.2–§5.6) | body (filters) |
| 18 | `frappe.client.get` | Default price list dari Selling/Buying Settings (§5.7) | body |

> **Konvensi pemanggilan (penting):** seluruh operasi memakai method whitelisted **`frappe.client.*`**
> dengan `name` (dan filter) dikirim lewat **body JSON**, bukan di URL path — konsisten dengan
> [prd_warehouse.md](../setup/prd_warehouse.md) & [prd_item.md](./prd_item.md). (`Price List` memang
> memakai `name` sederhana tanpa karakter khusus, tetapi konvensi body-based dipakai seragam agar UI
> hanya punya satu pola.)
> **Format respons:** method `/api/method/...` membungkus hasil di **`"message"`** (bukan `"data"`
> seperti `/api/resource/...`).

> Dokumen ini **tidak** membahas: `Pricing Rule` (diskon/harga bertingkat `min_qty`/`max_qty`),
> `Promotional Scheme`, dan perhitungan pajak — sudah tersedia di
> **[prd_item_pricing_rule.md](./prd_item_pricing_rule.md)** (Promotional Scheme menyusul).

---

## 2. Ringkasan field & data

### 2.1 Doctype `Price List`

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `price_list_name` | Data | Nama daftar harga. `reqd: 1`, `unique: 1`, **sekaligus menjadi `name`** (autoname `field:price_list_name`). Duplikat ditolak `DuplicateEntryError`. Tidak dapat diubah setelah dibuat (§2.3 no. 4). |
| 🔴 **WAJIB** | `currency` | Link → Currency | Mata uang harga pada price list ini. Bila dikosongkan, backend mengisi dari **Global Defaults** (uji: kosong → tersimpan `IDR`). |
| 🔴 **WAJIB** (salah satu) | `selling` / `buying` | Check | Minimal **satu** bernilai `1`. Keduanya `0` → error *"Price List must be applicable for Buying or Selling"* (§6). |
| 🟠 | `enabled` | Check | Default `1`. `0` = price list tidak bisa dipakai untuk membuat **dan** menemukan harga (§4.4). |
| ⚪ **Informatif** | `countries` | Table (`Price List Country`) | Child table berisi `country` (Link → Country). **Tidak dipakai mesin harga ERPNext** (tidak ada konsumen di kode) — boleh diisi sebagai metadata UI. `country` kosong → diisi otomatis dari Global Defaults. |
| ⚪ **Jangan diandalkan** | `price_not_uom_dependent` | Check | Default `0`. Flag "harga tidak bergantung UOM" — pada v16 **tidak dikonsumsi konsisten** oleh mesin harga (§2.3 no. 6). Untuk harga per UOM, buat baris `Item Price` per UOM. |
| ⚪ **Otomatis — jangan dikirim** | `name` | — | `name = price_list_name`. |

> Tidak ada field `company` di `Price List` — daftar harga bersifat **global** (§2.3 no. 3).

### 2.2 Doctype `Item Price`

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `item_code` | Link → Item | Item yang diberi harga. Item **template** (`has_variants: 1`) **ditolak** → *"Item Price cannot be created for the template item X"*; harga template dipakai varian sebagai fallback (§2.5). |
| 🔴 **WAJIB** | `price_list` | Link → Price List | Harus Price List yang **`enabled = 1`**, jika tidak → error *"The price list X does not exist or is disabled"*. |
| 🔴 **WAJIB** | `price_list_rate` | Currency | Nilainya harus `> 0` secara bisnis — **backend tidak menolak nilai negatif**, jadi validasi angka minimal dilakukan di frontend. |
| 🔴 **WAJIB** | `uom` | Link → UOM | **Diisi otomatis dari `Item.stock_uom`** bila dikosongkan (`fetch_if_empty`). UOM **wajib sudah terdaftar** di child `uoms` item tersebut (UOM Conversion Detail) → jika tidak: *"UOM X not found in Item Y"*. |
| 🟠 | `valid_from` | Date | Awal masa berlaku. Default `Today` (diisi backend bila dikosongkan). |
| 🟠 | `valid_upto` | Date | Akhir masa berlaku. Harus ≥ `valid_from` (*"Valid Up To must be after Valid From"*). Kosong = berlaku tanpa batas. |
| 🟠 | `customer` | Link → Customer | **Harga khusus customer** — hanya relevan pada price list `selling = 1`. Menang atas harga umum (§2.5). |
| 🟠 | `supplier` | Link → Supplier | **Harga khusus supplier** — hanya relevan pada price list `buying = 1`. |
| ⚪ | `batch_no` | Link → Batch | Harga khusus per batch (hanya item ber-batch). |
| ⚪ | `packing_unit` | Int | Default `0`. "Quantity that must be bought or sold per UOM" — harga hanya berlaku bila **qty transaksi kelipatan** nilai ini (§2.5). |
| ⚪ | `lead_time_days` | Int | Default `0`. Informasi lead time (dipakai alur pembelian/POS). |
| ⚪ | `note` | Text | Catatan bebas. |
| ⚪ **Read-only — jangan dikirim** | `buying`, `selling`, `currency`, `item_name`, `item_description`, `brand` | — | Diturunkan backend dari Price List & Item. Nilai yang dikirim frontend **diabaikan** (uji: kirim `currency: "USD"` → tersimpan `IDR`; kirim `selling: 0` → tersimpan `1`). |
| ⚪ **Otomatis — jangan dikirim** | `name`, `reference` | — | `name` = hash 10 karakter (autoname `hash`, mis. `19qob2i71u`). `reference` diisi otomatis = `customer` (selling) atau `supplier` (buying). |
| ✖️ **Tidak ada di doctype** | `min_qty`, `max_qty`, `enabled`, `disabled` | — | Harga bertingkat per qty memakai doctype **`Pricing Rule`** (`min_qty`/`max_qty`) — lihat [prd_item_pricing_rule.md](./prd_item_pricing_rule.md). Tidak ada field non-aktif → mengakhiri harga = hapus baris / isi `valid_upto` (§4.8). |

### 2.3 Catatan penting

1. **`name` Price List = `price_list_name` (autoname `field:`).** Nilai `name` inilah yang dipakai di
   `Item Price.price_list` dan di seluruh filter — bukan label lain. Duplikat `price_list_name` ditolak
   backend (`DuplicateEntryError`), cek dulu sebelum CREATE (§4.1 Langkah 0).
2. **`Item Price` tidak punya naming series** — `name` = hash acak. Karena itu **selalu** simpan `name`
   Item Price hasil CREATE; seluruh operasi berikutnya (ubah/hapus) memakai `name` tersebut.
3. **Tidak ada field `company` di kedua doctype.** `Price List` dan `Item Price` bersifat **global
   lintas company**; pembedaan per company dilakukan dengan **beberapa Price List** lalu diarahkan lewat:
   - `Selling Settings.selling_price_list` / `Buying Settings.buying_price_list` (default global situs,
     §5.7) — **Single doctype, bukan per company**;
   - `Customer.default_price_list` / `Supplier.default_price_list` (per pelanggan/pemasok);
   - `POS Profile.selling_price_list` (per toko, lihat [prd_pos_profile.md](../setup/prd_pos_profile.md));
   - `Item.item_defaults.default_price_list` (per item per company, lihat [prd_item.md](./prd_item.md)).
4. **`price_list_name` efektif tidak bisa diubah setelah dibuat.** Uji: mengubah field lalu `save`
   → nilai **kembali ke nilai lama** (dan `name` tidak berubah). Bila benar-benar perlu ganti nama,
   gunakan **`frappe.rename_doc`** — terbukti `name` berubah **dan** kolom `Item Price.price_list`
   ikut ter-update otomatis (§4.4).
5. **`currency`/`buying`/`selling` pada `Item Price` adalah cerminan Price List.** Perubahan pada Price
   List (mis. ganti `currency`) akan **meng-update seluruh baris Item Price** milik price list itu
   melalui SQL langsung (`PriceList.update_item_price`) — **tanpa** menjalankan validasi controller.
   Artinya: ganti mata uang Price List = ganti mata uang semua harganya sekaligus (§4.3).
6. **`price_not_uom_dependent` jangan diandalkan.** Mesin harga membaca flag internal
   `price_list_uom_dependant` yang **tidak ada** di doctype Price List v16, sehingga flag ini praktis
   tidak berefek. Yang berlaku: bila baris harga yang ditemukan UOM-nya berbeda dari UOM transaksi,
   rate dikalikan `conversion_factor` item (kecuali UOM sama). Untuk harga beda per UOM, buat baris
   `Item Price` terpisah per UOM (§4.5 Varian D).
7. **Duplikat harga divalidasi ketat.** Kombinasi `item_code + price_list + uom + valid_from +
   valid_upto + customer + supplier + batch_no + packing_unit` harus unik. Melanggar → error
   `ItemPriceDuplicateItem` (§4.5 Langkah 0).
8. **`valid_upto` yang sudah lewat tidak otomatis dinonaktifkan.** Backend tidak menandai baris
   kedaluwarsa; baris tetap ada dan hanya **diabaikan saat resolusi harga** karena filter tanggal
   (§2.5). Untuk UI, tampilkan status "kedaluwarsa" dengan membandingkan `valid_upto` dengan tanggal hari ini.
9. **Perubahan `Item.standard_rate` tidak meng-update harga yang sudah ada.** `standard_rate` hanya
   membuat Item Price **saat CREATE Item** (§4.9); pada Item yang sudah ada, backend hanya menyegarkan
   `item_name`/`item_description`/`brand` di Item Price, **bukan** `price_list_rate`. Ubah harga lewat
   Item Price (§4.7) atau bulk (§4.10).
10. **Transaksi bisa membuat harga sendiri.** `Stock Settings.auto_insert_price_list_rate_if_missing`
    (default **`1`**) membuat sistem menambahkan `Item Price` otomatis ketika sebuah transaksi diisi rate
    untuk price list yang belum punya harga — jika user punya izin `write` pada Item Price. Bila UI
    memerlukan kontrol penuh atas daftar harga, matikan flag ini.

### 2.4 Role yang dibutuhkan (v16)

Diambil langsung dari `DocPerm` doctype (bukan asumsi):

| Doctype | Baca | Tulis / Buat / Hapus |
|---|---|---|
| `Price List` | `Sales User`, `Purchase User`, `Manufacturing User` | `Sales Master Manager`, `Purchase Master Manager` |
| `Item Price` | **`Sales Master Manager`, `Purchase Master Manager`** | `Sales Master Manager`, `Purchase Master Manager` |

> ⚠️ **Berbeda dari Item/Warehouse:** peran `Item Manager`, `Stock Manager`, `Stock User`, dan bahkan
> `System Manager` **tidak punya** akses ke `Item Price` di v16 (uji `has_permission` → `read: false`,
> `write: false`). Praktisnya: **token/user yang dipakai layar "Kelola Harga" wajib memiliki role
> `Sales Master Manager` (jual) dan/atau `Purchase Master Manager` (beli)** — atau role custom dengan
> izin yang sama. Administrator situs selalu boleh.
> Role `Sales User` hanya bisa **membaca** Price List, **tidak** membaca Item Price.

### 2.5 Cara ERPNext memilih harga (penting untuk UI)

Saat transaksi (Sales Invoice/Delivery Note/Purchase Order/POS) membutuhkan harga, backend menjalankan
`get_item_price` (`erpnext.stock.get_item_details`) dengan aturan berikut:

1. **Filter:** `item_code` + `price_list` + `uom` (baris dengan UOM kosong ikut dipertimbangkan).
2. **Masa berlaku:** `valid_from <= tanggal transaksi <= valid_upto` (kosong = `2000-01-01` / `2500-12-31`).
3. **Prioritas baris:** `customer`/`supplier` **spesifik menang** atas baris umum; bila transaksi tidak
   punya customer/supplier, hanya baris umum yang dipakai.
4. **Urutan penentu:** `valid_from` terbaru → `batch_no` → `uom`, ambil **1 baris** teratas.
5. **Varian:** bila varian tidak punya harga, backend jatuh ke harga **template** (`variant_of`).
6. **Packing unit:** bila `packing_unit` terisi dan qty transaksi **bukan kelipatannya**, harga dianggap
   tidak berlaku (dihitung ulang/tanpa harga).
7. **Fallback price list (opsional):** `Selling Settings.fallback_to_default_price_list = 1` membuat
   sistem mencari ke price list default jual bila harga tidak ditemukan di price list yang dipakai.
   (Pada site dev nilainya `0`.)

> Implikasi untuk frontend: **tampilkan harga per kombinasi (item, price list, UOM, customer)** dengan
> memanggil `frappe.client.get_list` Item Price + filter tanggal (§5.8) — jangan mengandalkan satu field
> harga di Item.

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Detail lengkap setup OAuth Client ada di **[`prd_oauth.md`](../prd_oauth.md)**.

---

## 4. CRUD — Price List & Item Price

### 4.1 CREATE Price List — `frappe.client.insert`

**Langkah 0 — Pre-check duplikat (disarankan).**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Price List",
    "fields": ["name","currency","selling","buying","enabled"],
    "filters": [["price_list_name","=","Grosir"]],
    "limit_page_length": 1
  }'
```

- `message` kosong (`[]`) → lanjut CREATE.
- `message` terisi → blokir, tampilkan: *"Price List Grosir sudah ada."*

> Tanpa pre-check, backend tetap melempar `DuplicateEntryError`
> (`('Price List', 'Grosir', IntegrityError(1062, "Duplicate entry 'Grosir' for key 'PRIMARY'"))`).

**Payload minimum (harga jual):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Price List",
      "price_list_name": "Grosir",
      "currency": "IDR",
      "selling": 1
    }
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": {
    "name": "Grosir",
    "owner": "Administrator",
    "creation": "2026-09-15 09:15:00.000000",
    "modified": "2026-09-15 09:15:00.000000",
    "modified_by": "Administrator",
    "docstatus": 0,
    "idx": 0,
    "enabled": 1,
    "price_list_name": "Grosir",
    "currency": "IDR",
    "buying": 0,
    "selling": 1,
    "price_not_uom_dependent": 0,
    "countries": []
  }
}
```

> `name` = `Grosir`. Simpan nilai ini — dipakai sebagai `price_list` pada setiap `Item Price` (§4.5).

> Varian berikut cukup dijadikan nilai **`doc`** pada `frappe.client.insert` di atas, lalu baca `name`.

**Varian A — harga beli (supplier):**

```json
{
  "doctype": "Price List",
  "price_list_name": "Harga Beli Distributor",
  "currency": "IDR",
  "buying": 1
}
```

**Varian B — dua flag sekaligus (jual + beli):**

```json
{
  "doctype": "Price List",
  "price_list_name": "Internal Transfer",
  "currency": "IDR",
  "selling": 1,
  "buying": 1
}
```

**Varian C — dengan daftar negara (informatif, §2.1):**

```json
{
  "doctype": "Price List",
  "price_list_name": "Ekspor",
  "currency": "USD",
  "selling": 1,
  "countries": [
    { "country": "Singapore" },
    { "country": "Malaysia" }
  ]
}
```

### 4.2 READ Price List

**Detail 1 Price List — `frappe.client.get`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Price List",
    "name": "Grosir"
  }'
```

**Daftar Price List — `frappe.client.get_list`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Price List",
    "fields": ["name","currency","selling","buying","enabled"],
    "filters": [["enabled","=",1],["selling","=",1]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    {
      "name": "Grosir",
      "currency": "IDR",
      "selling": 1,
      "buying": 0,
      "enabled": 1
    },
    {
      "name": "Standard Selling",
      "currency": "IDR",
      "selling": 1,
      "buying": 0,
      "enabled": 1
    }
  ]
}
```

> `limit_page_length: 0` = ambil **semua** record (disarankan untuk dropdown karena jumlah Price List
> sedikit). Untuk daftar berpaginasi naikkan `limit_start` kelipatan `limit_page_length`.

**Total count — `frappe.client.get_count`** (untuk pagination):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_count \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Price List",
    "filters": [["enabled","=",1]]
  }'
```

```json
{ "message": 2 }
```

### 4.3 UPDATE Price List — `frappe.client.set_value` / `frappe.client.save`

Perubahan kecil (satu field) sebaiknya memakai `set_value`:

```bash
# jadikan default harga jual situs (Selling Settings), sekaligus contoh ubah 1 field Price List
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Price List",
    "name": "Grosir",
    "fieldname": { "currency": "USD" }
  }'
```

**Contoh respons (HTTP 200):** objek `message` berisi dokumen terbaru (`"currency": "USD"`).

> ⚠️ **Efek samping ubah `currency`/`selling`/`buying`:** backend menjalankan
> `update tabItem Price set currency=..., buying=..., selling=... where price_list=...` — **semua** baris
> Item Price milik Price List ini ikut berubah (tanpa validasi). Perubahan ini juga menghapus cache
> `price_list_details`. Lakukan sebelum harga-harga diisi, atau pastikan UI menampilkan konfirmasi.

Untuk mengubah beberapa field sekaligus, kirim dokumen hasil GET yang dimodifikasi (field `name` ikut):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Price List",
      "name": "Grosir",
      "price_list_name": "Grosir",
      "currency": "IDR",
      "selling": 1,
      "buying": 0,
      "enabled": 1
    }
  }'
```

**Menjadikan Price List sebagai default situs:** backend **otomatis** melakukannya pada `on_update` bila
`Selling Settings.selling_price_list` (atau `Buying Settings.buying_price_list`) masih **kosong**
(`set_default_if_missing`). Untuk memaksa mengganti default, set langsung ke Single-nya:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Selling Settings",
    "name": "Selling Settings",
    "fieldname": { "selling_price_list": "Grosir" }
  }'
```

> Jangan kirim `price_list_name` yang berbeda dari `name` — nilainya akan kembali ke nilai lama (§2.3 no. 4).

### 4.4 Non-aktif, rename, dan larangan DELETE Price List

**Non-aktifkan (disarankan) — `enabled = 0`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Price List",
    "name": "Grosir",
    "fieldname": { "enabled": 0 }
  }'
```

Aktifkan kembali: `fieldname: { "enabled": 1 }`.

> **Efek non-aktif:** price list tidak muncul di dropdown (`filter enabled=1`), dan **tidak bisa dipakai
> membuat harga baru** — insert Item Price ke price list non-aktif ditolak dengan
> *"The price list Grosir does not exist or is disabled"*. Baris Item Price lama **tidak dihapus**.

**⚠️ Jangan gunakan DELETE.** `frappe.client.delete` pada Price List yang masih memiliki baris
`Item Price` **ditolak backend** dengan pesan yang sekaligus jadi panduan resmi:

```json
{
  "exc_type": "LinkExistsError",
  "message": "You can disable this Price List instead of deleting it."
}
```

**Ganti nama (`name` Price List) — `frappe.rename_doc`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.rename_doc \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Price List",
    "old": "Grosir",
    "new": "Grosir 2026",
    "merge": 0
  }'
```

```json
{ "message": "Grosir 2026" }
```

> Terverifikasi: rename mengubah `name` Price List **dan** memperbarui `Item Price.price_list` pada
> semua baris terkait secara otomatis. Setelah rename, `price_list_name` juga mengikuti `name` baru.
> Kirim `merge: 1` hanya bila memang ingin menggabungkan ke Price List yang sudah ada (berisiko).

### 4.5 CREATE Item Price — `frappe.client.insert`

**Langkah 0 — Pre-check duplikat (wajib, karena pesan error backend generik).**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Price",
    "fields": ["name","price_list_rate","uom","valid_from","valid_upto"],
    "filters": [
      ["item_code","=","MIN-001"],
      ["price_list","=","Grosir"],
      ["uom","=","Nos"],
      ["valid_from","=","2026-09-15"]
    ],
    "limit_page_length": 1
  }'
```

> Tambahkan `["customer","=","Toko Sumber Rejeki"]` (atau `["supplier","=","PT Sumber Air"]`) pada filter
> bila yang dicek adalah harga khusus — kombinasi duplikat diperiksa **termasuk** kolom customer/supplier.

- `message` kosong (`[]`) → lanjut ke CREATE.
- `message` terisi → blokir dan tampilkan: *"Harga item MIN-001 di price list Grosir untuk UOM Nos sudah ada (berlaku sejak 2026-09-15)."*

> **Catatan backend:** tanpa pre-check, duplikat menghasilkan `ItemPriceDuplicateItem` dengan pesan
> *"Item Price appears multiple times based on Price List, Supplier/Customer, Currency, Item, Batch,
> UOM, Qty, and Dates."* — tidak menyebut field mana yang bentrok. Bandingkan kombinasi
> `item_code + price_list + uom + valid_from + valid_upto + customer + supplier + batch_no + packing_unit`.

**Payload minimum (harga umum) — `frappe.client.insert`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item Price",
      "item_code": "MIN-001",
      "price_list": "Grosir",
      "price_list_rate": 12000,
      "valid_from": "2026-09-15"
    }
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": {
    "name": "h6o7fl78so",
    "owner": "Administrator",
    "creation": "2026-09-15 09:31:03.903381",
    "modified": "2026-09-15 09:31:03.903381",
    "modified_by": "Administrator",
    "docstatus": 0,
    "idx": 0,
    "item_code": "MIN-001",
    "uom": "Nos",
    "packing_unit": 0,
    "item_name": "Air Mineral 600ml",
    "brand": null,
    "item_description": null,
    "price_list": "Grosir",
    "customer": null,
    "supplier": null,
    "batch_no": null,
    "buying": 0,
    "selling": 1,
    "currency": "IDR",
    "price_list_rate": 12000.0,
    "valid_from": "2026-09-15",
    "lead_time_days": 0,
    "valid_upto": null,
    "note": null,
    "reference": null
  }
}
```

> Perhatikan: **`uom` terisi otomatis** dari `Item.stock_uom` (`Nos`), `valid_from` default `Today`,
> `buying`/`selling`/`currency` mengikuti Price List, dan `item_name` disalin dari Item.
> Simpan `name` (di sini `h6o7fl78so`) untuk update/hapus.

**Varian A — masa berlaku (promo):**

```json
{
  "doctype": "Item Price",
  "item_code": "MIN-001",
  "price_list": "Grosir",
  "price_list_rate": 10000,
  "uom": "Nos",
  "valid_from": "2026-09-01",
  "valid_upto": "2026-09-30",
  "note": "Promo bulan September"
}
```

**Varian B — harga khusus customer:**

```json
{
  "doctype": "Item Price",
  "item_code": "MIN-001",
  "price_list": "Standard Selling",
  "price_list_rate": 3500,
  "uom": "Nos",
  "customer": "Toko Sumber Rejeki",
  "valid_from": "2026-09-15"
}
```

> `reference` otomatis = nama customer. Harga ini **menang** atas harga umum untuk customer tersebut
> (§2.5 no. 3). Satu customer boleh punya lebih dari satu baris (beda `valid_from`/UOM).

**Varian C — harga beli khusus supplier:**

```json
{
  "doctype": "Item Price",
  "item_code": "MIN-001",
  "price_list": "Harga Beli Distributor",
  "price_list_rate": 2000,
  "uom": "Nos",
  "supplier": "PT Sumber Air",
  "valid_from": "2026-09-15"
}
```

> `price_list` **wajib** price list dengan `buying = 1`; di respons `selling` otomatis `0` dan
> `reference` = supplier.

**Varian D — harga per UOM (mis. per krat):**

```json
{
  "doctype": "Item Price",
  "item_code": "MIN-001",
  "price_list": "Grosir",
  "price_list_rate": 110000,
  "uom": "Krat",
  "valid_from": "2026-09-15"
}
```

> Prasyarat: UOM `Krat` **sudah ada** di item (child `Item.uoms` — UOM Conversion Detail). Bila belum,
> tambahkan dulu lewat `frappe.client.save` pada dokumen Item ([prd_item.md](./prd_item.md)), jika tidak
> → *"UOM Krat not found in Item MIN-001"*.

**Varian E — harga dengan `packing_unit` (qty harus kelipatan):**

```json
{
  "doctype": "Item Price",
  "item_code": "MIN-001",
  "price_list": "Grosir",
  "price_list_rate": 108000,
  "uom": "Krat",
  "packing_unit": 12,
  "valid_from": "2026-09-15"
}
```

> Harga berlaku hanya bila qty transaksi kelipatan `12` (§2.5 no. 6).

**Varian F — harga per batch:**

```json
{
  "doctype": "Item Price",
  "item_code": "SUSU-001",
  "price_list": "Standard Selling",
  "price_list_rate": 8500,
  "uom": "Nos",
  "batch_no": "BATCH-2026-09",
  "valid_from": "2026-09-15"
}
```

**Yang TIDAK boleh dikirim:** `currency`, `buying`, `selling`, `item_name`, `item_description`, `brand`,
`reference` (read-only / otomatis). Nilai yang dikirim akan **diabaikan** tanpa error — jadi jangan
mengandalkannya untuk mengubah mata uang atau flag.

### 4.6 READ Item Price

**Detail 1 baris harga — `frappe.client.get`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Price",
    "name": "h6o7fl78so"
  }'
```

**Semua harga 1 item (disarankan untuk layar detail produk / tabel harga):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Price",
    "fields": ["name","price_list","price_list_rate","uom","currency","customer","supplier","valid_from","valid_upto","packing_unit"],
    "filters": [["item_code","=","MIN-001"]],
    "order_by": "price_list asc, valid_from desc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    {
      "name": "h6o7fl78so",
      "price_list": "Grosir",
      "price_list_rate": 12000.0,
      "uom": "Nos",
      "currency": "IDR",
      "customer": null,
      "supplier": null,
      "valid_from": "2026-09-15",
      "valid_upto": null,
      "packing_unit": 0
    },
    {
      "name": "h6pleec2r8",
      "price_list": "Standard Selling",
      "price_list_rate": 3500.0,
      "uom": "Nos",
      "currency": "IDR",
      "customer": "Toko Sumber Rejeki",
      "supplier": null,
      "valid_from": "2026-09-15",
      "valid_upto": null,
      "packing_unit": 0
    }
  ]
}
```

**Semua harga pada satu Price List** (mis. layar "Isi Harga" per daftar harga — padanan tombol
*Add / Edit Prices* di Desk):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Price",
    "fields": ["name","item_code","item_name","price_list_rate","uom","valid_from","valid_upto"],
    "filters": [["price_list","=","Grosir"],["valid_upto",">=","2026-09-15"]],
    "limit_start": 0,
    "limit_page_length": 50
  }'
```

> Halaman berikutnya naikkan `limit_start` kelipatan `50`; `limit_page_length: 0` = ambil semua.
> Total record untuk pagination diambil dari `frappe.client.get_count` (filter sama).

### 4.7 UPDATE Item Price — `frappe.client.set_value` / `frappe.client.save`

**Ubah harga (satu field) — `set_value`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Price",
    "name": "h6o7fl78so",
    "fieldname": { "price_list_rate": 13500 }
  }'
```

**Contoh respons (HTTP 200):** objek `message` berisi dokumen terbaru dengan `"price_list_rate": 13500`.

**Ubah beberapa field sekaligus — `frappe.client.save`** (kirim hasil GET yang dimodifikasi):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item Price",
      "name": "h6o7fl78so",
      "item_code": "MIN-001",
      "price_list": "Grosir",
      "price_list_rate": 13500,
      "uom": "Nos",
      "valid_from": "2026-09-15",
      "valid_upto": "2026-12-31",
      "packing_unit": 0,
      "note": "Harga naik September"
    }
  }'
```

> **`set_value` tidak menjalankan validasi controller** (update langsung ke kolom) — validasi duplikat,
> `valid_from`/`valid_upto`, dan uom **tidak** dicek. Karena itu frontend tetap harus mengecek duplikat
> sendiri (§4.5 Langkah 0) sebelum memanggil `set_value`. Gunakan `save` bila ingin validasi backend
> berjalan.
> `uom`, `price_list`, dan `item_code` **boleh** diubah, tetapi mengubahnya = mengubah identitas baris
> harga; lebih aman hapus baris lama (§4.8) lalu buat baris baru (§4.5).

### 4.8 Mengakhiri / menghapus harga — `frappe.client.delete` atau `valid_upto`

`Item Price` **tidak punya field `disabled`** dan **tidak menyimpan riwayat transaksi**, sehingga baris
harga adalah **data assignment (bukan master data)** — hapus aman, seperti `User Permission`:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.delete \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Price",
    "name": "h6o7fl78so"
  }'
```

```json
{ "message": "ok" }
```

> Terverifikasi: hapus Item Price **berhasil** (tidak ada validasi `on_trash`). Transaksi lama **tidak**
> terpengaruh karena rate sudah tersimpan di dokumen transaksi.

**Alternatif tanpa hapus (disarankan bila ingin menyimpan jejak harga):** tutup masa berlaku.

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Price",
    "name": "h6o7fl78so",
    "fieldname": { "valid_upto": "2026-09-14" }
  }'
```

> Baris tetap ada sebagai riwayat, tetapi **diabaikan** saat resolusi harga untuk tanggal setelah
> `valid_upto` (§2.5 no. 2). UI sebaiknya menandainya "kedaluwarsa".
> **Catatan jangan terbalik:** `Price List` (master) **tidak boleh** dihapus → non-aktifkan `enabled = 0`
> (§4.4); `Item Price` (baris harga) **boleh** dihapus (§4.8).

### 4.9 Item Price otomatis saat Item dibuat

Bila Item dibuat, ERPNext (`Item.after_insert → add_price`) tetap membuat Item Price dari
`standard_rate` sesuai price list default (satu baris per baris `item_defaults`). Selain itu, custom app
`baseapp` memastikan harga umum untuk UOM stok di kedua price list standar sesuai pemetaan berikut:

Untuk perilaku bawaan ERPNext, price list ditentukan dari `Item.item_defaults[].default_price_list`;
jika kosong, dipakai `Selling Settings.selling_price_list`, lalu fallback ke `Standard Selling`.

| Price List | Field Item | Field Item Price |
|---|---|---|
| `Standard Selling` | `valuation_rate` | `price_list_rate` |
| `Standard Buying` | `standard_rate` | `price_list_rate` |

Baris khusus customer/supplier, batch, UOM selain `Item.stock_uom`, atau masa berlaku yang tidak aktif
tidak dipakai sebagai baris master. Rate positif membuat baris bila belum ada; Item Price Standard Selling
yang dibuat bawaan ERPNext akan diselaraskan dengan `valuation_rate`. Baris bawaan untuk price list selain
`Standard Selling`/`Standard Buying` tetap dibuat seperti semula dan tidak disinkronkan ke rate Item.

```json
{
  "doctype": "Item",
  "item_code": "MIN-001",
  "item_name": "Air Mineral 600ml",
  "item_group": "Minuman",
  "stock_uom": "Nos",
  "is_stock_item": 1,
  "standard_rate": 3500,
  "item_defaults": [
    { "company": "PT Maju Jaya", "default_price_list": "Grosir" }
  ]
}
```

> Pada contoh di atas `standard_rate: 3500` menghasilkan **1 baris** Item Price di price list `Grosir`
> (diambil dari `item_defaults.default_price_list`). Backend melakukan *loop* per baris `item_defaults`,
> jadi: **tanpa** `item_defaults` → 1 baris di `Selling Settings.selling_price_list`; `item_defaults`
> tanpa `default_price_list` → 1 baris di `Selling Settings.selling_price_list`; `item_defaults` dengan
> 2 baris bermacam `default_price_list` → 2 baris Item Price. Lihat [prd_item.md](./prd_item.md) untuk
> penyimpanan `item_defaults` saat CREATE Item.

> **Catatan sinkronisasi baseapp:** saat `price_list_rate` pada baris umum dengan UOM stok di
> `Standard Selling` diedit, `Item.valuation_rate` ikut diperbarui; perubahan di `Standard Buying`
> memperbarui `Item.standard_rate`. Harga khusus customer/supplier, batch, UOM lain, dan baris
> kedaluwarsa/tanggal mendatang tidak mengubah rate Item.
>
> Mengubah field rate pada Item yang sudah ada tidak otomatis mengubah Item Price; perubahan rate master
> dilakukan melalui Item Price (§4.7) atau bulk update (§4.10).
> Selain itu `Stock Settings.auto_insert_price_list_rate_if_missing = 1` dapat **menambah Item Price
> otomatis** saat transaksi diisi rate untuk price list yang belum punya harga (§2.3 no. 10).

### 4.10 Bulk update harga — Data Import (CSV/Excel)

`Item Price` sudah mengizinkan import (`allow_import = 1`), jadi update harga massal memakai alur Data
Import standar (sama seperti PRD Supplier/Customer), **3 langkah**:

**Langkah 1 — upload berkas → `file_url`:**

```bash
curl -X POST https://site-anda.com/api/method/upload_file \
  -H 'Authorization: Bearer <access_token>' \
  -F 'file=@harga_grosir.csv' \
  -F 'is_private=1'
```

```json
{ "message": { "file_url": "/private/files/harga_grosir.csv", "name": "a1b2c3d4e5" } }
```

**Langkah 2 — buat dokumen Data Import:**

```bash
curl -X POST https://site-anda.com/api/resource/Data%20Import \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Data Import",
    "reference_doctype": "Item Price",
    "import_type": "Update",
    "import_file": "/private/files/harga_grosir.csv",
    "status": "Pending"
  }'
```

**Langkah 3 — jalankan import:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.core.doctype.data_import.data_import.form_start_import \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "data_import": "a1b2c3d4e5" }'
```

**Pemantauan status:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.core.doctype.data_import.data_import.get_import_status \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "data_import": "a1b2c3d4e5" }'
```

**Aturan berkas (penting untuk `Item Price`):**

| Kolom | Wajib | Catatan |
|---|---|---|
| `ID` (`name` Item Price) | **ya untuk `import_type: "Update"`** | Hash Item Price dari §4.6 — inilah kunci update harga lama. |
| `item_code` | ya (Insert) | |
| `price_list` | ya | Harus price list `enabled = 1`. |
| `price_list_rate` | ya | Kolom "Rate". |
| `uom` | disarankan | Kosong → diisi `Item.stock_uom` saat import. |
| `valid_from` / `valid_upto` | disarankan | Format `YYYY-MM-DD`. |

- `import_type`: `Insert` (harga baru) · `Update` (ubah harga lama, butuh kolom `ID`) ·
  `Insert or Update` (campuran — dipakai untuk sinkronisasi harga massal).
- Jangan sertakan kolom `currency`/`buying`/`selling` (read-only, akan diabaikan/dianggap error validasi kolom tidak dikenal).
- Syarat hak akses: user harus punya izin **Import** dan role tulis Item Price (§2.4).
- Alternatif dari sisi Desk: halaman `/app/data-import-tool/Item Price` (padanan tautan *"Import in Bulk"*
  pada form Item Price). Untuk mengambil **template kolom kosong**, gunakan
  `frappe.core.doctype.data_import.data_import.download_template`.

---

## 5. GET pendukung UI

### 5.1 GET Price List — dropdown harga jual / beli

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Price List",
    "fields": ["name","currency","selling","buying","enabled"],
    "filters": [["enabled","=",1],["selling","=",1]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

> Ganti filternya menjadi `["buying","=",1]` untuk dropdown sisi pembelian. `name` yang dikirim ke field
> `price_list` Item Price / `selling_price_list` POS Profile.

### 5.2 GET Currency — dropdown `currency`

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Currency",
    "fields": ["name","enabled"],
    "filters": [["enabled","=",1]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

### 5.3 GET Item — dropdown pilih produk

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","stock_uom"],
    "filters": [["is_sales_item","=",1],["disabled","=",0]],
    "order_by": "item_name asc",
    "limit_page_length": 50
  }'
```

> Untuk sisi pembelian ganti filter menjadi `["is_purchase_item","=",1]`. **Wajib mengecualikan item
> template** dari daftar pilihan harga: tambahkan `["has_variants","=",0]` — backend menolak Item Price
> untuk item `has_variants = 1` (§2.2). Filter per Item Group: lihat [prd_item.md §6.7](./prd_item.md).

### 5.4 GET UOM milik Item — validasi `uom` harga

UOM pada Item Price **harus** sudah terdaftar di item. Ambil daftarnya dari dokumen Item:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "name": "MIN-001"
  }'
```

> Respons `message.uoms` berisi baris `{ "uom": "Nos", "conversion_factor": 1.0 }` — pakai kolom `uom`
> sebagai pilihan dropdown harga. Tambah UOM baru lewat `frappe.client.save` dokumen Item
> ([prd_item.md](./prd_item.md)).

### 5.5 GET Customer — dropdown harga khusus jual

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Customer",
    "fields": ["name","customer_name","default_price_list","disabled"],
    "filters": [["disabled","=",0]],
    "order_by": "customer_name asc",
    "limit_page_length": 50
  }'
```

> `default_price_list` di sini adalah Price List default customer tersebut (dipakai transaksi) — berguna
> untuk mengisi awal dropdown price list. Detail: [prd_customer.md](../contact/prd_customer.md).

### 5.6 GET Supplier — dropdown harga khusus beli

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Supplier",
    "fields": ["name","supplier_name","default_price_list","disabled"],
    "filters": [["disabled","=",0]],
    "order_by": "supplier_name asc",
    "limit_page_length": 50
  }'
```

> Detail: [prd_supplier.md](../contact/prd_supplier.md).

### 5.7 GET default price list — Selling / Buying Settings

Untuk menampilkan/menandai "price list default" di UI (Single doctype, `name` = nama doctype-nya):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Selling Settings",
    "name": "Selling Settings"
  }'
```

> Field relevan pada `message`: `selling_price_list`, `fallback_to_default_price_list`,
> `editable_price_list_rate`, `validate_selling_price`, `allow_negative_rates_for_items`.
> Untuk pembelian, ganti menjadi `Buying Settings` → `buying_price_list`.
> Default yang dipakai mesin harga = `Selling Settings.selling_price_list`
> (site dev contohnya `Standard Selling`).

### 5.8 GET harga siap-pakai untuk 1 item (komponen harga UI)

Gabungan §4.6 + §5.7 untuk menampilkan harga efektif di layar produk/POS:

1. Ambil default price list dari Settings (§5.7) atau dari `Customer/Supplier.default_price_list` (§5.5/§5.6);
2. Ambil baris harga item pada price list itu (§4.6) dengan filter tanggal berlaku:
   `[["item_code","=","MIN-001"],["price_list","=","Standard Selling"],["valid_from","<=","2026-09-15"]]`
   lalu saring `valid_upto` kosong atau `>=` tanggal di frontend (atau tambahkan
   `["valid_upto",">=","2026-09-15"]` bila ingin hanya yang berlaku);
3. Pilih baris dengan `customer`/`supplier` sesuai lawan transaksi bila ada; kalau tidak, pakai baris umum
   (urutan penentu: `valid_from` terbaru).

---

## 6. Penanganan error umum

Pesan di bawah ini **persis** seperti yang dikembalikan instance v16 (format mandatory error v16 berbeda
dari versi lama — berbentuk daftar `[Doctype, name]: field`).

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | Role tidak punya akses (mis. hanya `Stock Manager`) | `{"message": "Not permitted"}` |
| 404 | `name` tidak ditemukan (`get`/`save`) | `{"exc_type":"DoesNotExistError","message":"Resource Not Found"}` |
| 417 | `price_list_name` Price List kosong | `{"exc_type":"ValidationError","message":"Price List Name is required"}` |
| 417 | Price List tanpa `selling` & `buying` | `{"exc_type":"ValidationError","message":"Price List must be applicable for Buying or Selling"}` |
| 417 | Price List sudah ada (tanpa pre-check) | `{"exc_type":"DuplicateEntryError","message":"('Price List', 'Grosir', IntegrityError(1062, \"Duplicate entry 'Grosir' for key 'PRIMARY'\"))"}` |
| 417 | Item Price: `item_code` tidak ada / item template | `{"exc_type":"ValidationError","message":"Item None not found."}` · `{"exc_type":"ValidationError","message":"Item Price cannot be created for the template item <strong>KAOS-POLOS</strong>"}` |
| 417 | Item Price: UOM tidak terdaftar di Item | `{"exc_type":"ValidationError","message":"UOM Krat not found in Item MIN-001"}` |
| 417 | Item Price: `price_list` tidak ada | `{"exc_type":"LinkValidationError","message":"Could not find Price List: Grosir"}` |
| 417 | Item Price: price list **disabled** | `{"exc_type":"ValidationError","message":"The price list Grosir does not exist or is disabled"}` |
| 417 | Item Price: kombinasi harga duplikat | `{"exc_type":"ItemPriceDuplicateItem","message":"Item Price appears multiple times based on Price List, Supplier/Customer, Currency, Item, Batch, UOM, Qty, and Dates."}` |
| 417 | Item Price: `valid_upto` < `valid_from` | `{"exc_type":"InvalidDates","message":"<strong>Valid Up To</strong> must be after <strong>Valid From</strong>"}` |
| 417 | Field wajib Item Price kosong (v16) | `{"exc_type":"MandatoryError","message":"[Item Price, h6o7fl78so]: price_list_rate"}` |
| 417 | Hapus Price List yang masih dipakai | `{"exc_type":"LinkExistsError","message":"You can disable this Price List instead of deleting it."}` |

> **Catatan:** karena seluruh pemanggilan memakai `/api/method/...`, hasil sukses dibungkus di
> `message` (bukan `data`). Body error tetap berbentuk `{ "exc_type": ..., "exception": ..., "message": ... }`.
> Beberapa pesan berisi HTML (`<strong>`, `<a href>`) — bersihkan/escape saat menampilkan di UI.

---

## 7. Catatan & PRD lanjutan

1. **Pricing Rule & harga bertingkat — sudah tersedia.** Dokumen ini **tidak** mencakup harga
   bertingkat (`min_qty`/`max_qty`), diskon, harga per Item Group/Brand, coupon, dan skema promo.
   Hal tersebut berada di doctype **`Pricing Rule`** (field relevan: `apply_on` Item Code/Item Group/Brand,
   `price_or_product_discount`, `min_qty`, `max_qty`, `min_amt`, `rate_or_discount`, `for_price_list`,
   `valid_from`, `valid_upto`, `company`) — dibahas lengkap di
   **[prd_item_pricing_rule.md](./prd_item_pricing_rule.md)** (termasuk contoh tiap jenis promo, kupon,
   prioritas/tumpukan rule, dan simulasi promo untuk keranjang).
   `Item Price` sendiri **tidak punya** field `min_qty`/`max_qty`.
2. **Export/Import masal:** import (bulk update) sudah didokumentasikan di §4.10. Ekspor daftar harga
   massal mengikuti pola Export Excel/CSV yang dipakai PRD lain
   (`frappe.core.doctype.data_export.exporter.export_data`).
3. **Koleksi Postman:** folder **`15. Price List & Item Price`** (30 request) sudah ditambahkan ke
   `docs/postman/postman_erpnext_api.json`:
   - **Price List (§4.1–§4.4)** → `15.1`–`15.13`: pre-check, CREATE (jual / beli / dua flag + countries), READ
     single & list/dropdown, count, update (`set_value` & `save`), non-aktif, `rename_doc`, contoh DELETE yang
     ditolak, dan default price list dari Selling Settings;
   - **Item Price (§4.5–§4.10, §5)** → `15.14`–`15.30`: pre-check duplikat, 6 varian CREATE (umum, berperiode,
     khusus customer, khusus supplier, per UOM, `packing_unit`), READ single, list per item, list per price
     list, count, update (`set_value` & `save`), akhiri harga via `valid_upto`, DELETE, daftar UOM milik item,
     dan Data Import untuk bulk update harga.
   Variabel baru: `price_list_name`, `price_list_name_2`, `item_price_id`, `item_price_id_2`, `item_price_rate`
   (test script pada request CREATE menyimpan `name`). Nomor folder lain yang sudah terpakai: `11. Item`,
   `13. Stock Ledger`, `14. Pricing Rule`.
4. **File terkait:** Item [prd_item.md](./prd_item.md) (master produk, `standard_rate`,
   `item_defaults.default_price_list`) · Item Group [prd_item_group.md](./prd_item_group.md) ·
   Warehouse [prd_warehouse.md](../setup/prd_warehouse.md) · POS Profile
   [prd_pos_profile.md](../setup/prd_pos_profile.md) (memakai `selling_price_list`) ·
   Customer [prd_customer.md](../contact/prd_customer.md) · Supplier
   [prd_supplier.md](../contact/prd_supplier.md) (memakai `default_price_list`) · OAuth
   [prd_oauth.md](../prd_oauth.md).
