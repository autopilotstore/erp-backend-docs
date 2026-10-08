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
| 7 | `frappe.client.get_count` | Total record Item sesuai filter — pagination (termasuk saat memakai `or_filters`, §2.1 no. 9) | body |
| 8 | `frappe.client.save` | Ubah Item (UPDATE) | body (`doc`) |
| 9 | `frappe.client.set_value` | Ubah field tunggal — non-aktif, pindah `item_group`, ubah harga, dst. | body |
| 10 | `frappe.client.attach_file` | Upload foto produk (multi foto → `tabFile`, §4.8) | body (base64) |
| 11 | `frappe.client.get_list` | Daftar foto Item dari `tabFile` (READ list, §4.8) | body (filters) |
| 12 | `frappe.client.insert` | Tag foto template ke varian — buat record `File` baru menunjuk `file_url` yang sama (Pendekatan A, §4.8) | body (`doc`) |
| 13 | `frappe.client.delete` | Hapus foto (File) / Item — **dengan batasan** (§4.8, §4.9) | body |
| 14 | `frappe.client.submit` | Submit **Stock Reconciliation** — eksekusi zero stok sebelum non-aktif (§4.9) | body (`doc`) |
| 15 | `erpnext.stock.doctype.item.item.get_uom_conv_factor` | Resolve faktor konversi UOM (pendukung §5) | body |
| 16 | `erpnext.stock.doctype.item.item.get_item_attribute` | Autocomplete nilai Item Attribute (dropdown varian, §6) | body |
| 17 | `frappe.client.get_list` | Baca stok via **Bin / `tabBin`** (info stok, §4.10) | body (filters) |
| 18 | `baseapp.api.check_item_name` | Pre-check nama Item duplikat — **custom endpoint app `baseapp`** (§4.11) | body |

> **Konvensi pemanggilan (penting):** seluruh operasi memakai method whitelisted **`frappe.client.*`**
> (kecuali method ERPNext khusus) dengan `name` (dan filter) dikirim lewat **body JSON**, bukan di
> URL path. `item_code` bisa mengandung spasi / karakter khusus — mengirim `name` di body menghindari
> masalah URL-encoding di belakang nginx/proxy.
> **Format respons:** method `/api/method/...` membungkus hasil di **`"message"`** (bukan `"data"`
> seperti `/api/resource/...`).

> **Perbedaan utama dengan Item Group:**
> - `name = item_code` — tapi kodenya **di-generate backend** lewat naming series `YY.MM.######`
>   (frontend tidak lagi menentukan kodenya). Lihat §2.1 no. 1.
> - Item **punya field `disabled`** → operasi non-aktif (soft-delete) berlaku (§4.9).
> - Stok Item **tidak disimpan di `tabItem`**, melainkan di `tabBin` per item+warehouse (§4.10) dan
>   riwayatnya di Stock Ledger Entry (PRD stok terpisah, §9).

---

## 2. Ringkasan field & data

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| ⚪ **Otomatis — dibuat backend** | `item_code` | Data | Kode produk **sekaligus `name`**. **Dibuat backend** dari naming series `YY.MM.######` (mis. `2609000001`) — lihat §2.1 no. 1. Kiriman frontend **ditimpa/diabaikan** — **tanpa pengecualian, termasuk varian**. Tidak perlu dikirim. |
| 🔴 **WAJIB** | `item_name` | Data | **Nama tampilan produk** — inilah identitas yang dibaca user. Kosong → backend menyalin `item_code`, yang sekarang berupa angka seri (`2609000001`), sehingga **selalu kirim `item_name`**. **Harus unik di antara Item yang aktif** (§2.1 no. 8) — pre-check dulu dengan `baseapp.api.check_item_name` (§4.11). |
| 🔴 **WAJIB** | `item_group` | Link → Item Group | Grup produk (pilih node **leaf**/daun, `is_group=0`). Detail: [prd_item_group.md §5.2](./prd_item_group.md). |
| 🔴 **WAJIB** | `stock_uom` | Link → UOM | Satuan dasar stok (Unit of Measure). Semua qty stok & konversi dihitung relatif ke UOM ini. |
| 🟠 | `is_stock_item` | Check | `1` = produk stok (dikelola via Stock Ledger, punya `tabBin`). `0` = jasa/non-stok. |
| 🟠 | `is_dynamic_product_bundle` | Check | `1` = Item ini **paket dinamis** — komponen/isiannya dipilih kasir saat transaksi (berbeda dari `Product Bundle` bawaan ERPNext yang statis — paket statis dibahas di [prd_item_product_bundle.md](./prd_item_product_bundle.md)). **Custom field dari app `baseapp`** (fieldname tanpa prefix `custom_`, posisi setelah `is_stock_item`); hanya ada bila app tersebut terpasang. Default `0`. Struktur & CRUD pilihannya: [prd_item_dynamic_product_bundle.md](./prd_item_dynamic_product_bundle.md). |
| 🟠 | `is_product_bundle` | Check | `1` = Item ini **paket statis** (doctype `Product Bundle`, tabel `tabProduct Bundle`) — ERPNext otomatis memecahnya jadi komponen saat transaksi. **Custom field dari app `baseapp`** (fieldname tanpa prefix `custom_`, posisi setelah `is_dynamic_product_bundle`); hanya ada bila app tersebut terpasang. Default `0`, tersedia sebagai **filter standar**, dan **diisi manual oleh user/frontend** — **tidak** diturunkan dari dokumen `Product Bundle` dan **tidak terikat** status aktif/non-aktifnya (§2.1 no. 9). Detail paket statis: [prd_item_product_bundle.md](./prd_item_product_bundle.md). |
| 🟠 | `is_manufactured_item` | Check | `1` = Item ini **produk manufaktur** (diproduksi sendiri). **Custom field dari app `baseapp`** (fieldname tanpa prefix `custom_`, posisi setelah `is_product_bundle`); hanya ada bila app tersebut terpasang. Default `0` dan tersedia sebagai **filter standar**. **Diisi manual oleh user/frontend — bukan turunan dari BOM** (ERPNext tidak menyimpan penanda "manufactured" di Item; yang tersedia hanya `default_bom`, `include_item_in_manufacturing`, dan `is_sub_contracted_item`). |
| 🟠 | `is_sales_item` | Check | `1` = produk bisa dijual (muncul di Quotation/Sales Order/Sales Invoice/POS). |
| 🟠 | `min_sales_qty` | Float | Jumlah jual minimum per Item dalam `stock_uom`; default `0` (tanpa batas minimum). Custom field bawaan app `baseapp`; field tersedia setelah app terpasang. |
| 🟠 | `max_sales_qty` | Float | Jumlah jual maksimum per Item dalam `stock_uom`; default `0` (tanpa batas maksimum). Custom field bawaan app `baseapp`; field tersedia setelah app terpasang. |
| 🟠 | `sales_qty_multiple` | Float | Kelipatan jumlah jual per Item dalam `stock_uom`; default `0` (aturan kelipatan tidak digunakan). Custom field bawaan app `baseapp`; field tersedia setelah app terpasang. |
| 🟠 | `is_purchase_item` | Check | `1` = produk bisa dibeli (muncul di Request for Quotation/Purchase Order/Purchase Receipt). |
| 🟠 | `purchase_uom` | Link → UOM | UOM default saat transaksi **beli** (bisa berbeda dari `stock_uom`; faktor konversi di `uoms`, §2.3/§5). |
| 🟠 | `has_variants` | Check | `1` = Item ini **template** varian (wajib isi `attributes`, §2.5). `0`/kosong = Item tunggal. |
| 🟠 | `is_fixed_asset` | Check | `1` = Item **aset tetap** (mis. mesin/kendaraan) yang dibeli untuk dikapitalisasi jadi **Asset** (modul Assets; `is_stock_item` ikut jadi `0`, butuh `asset_category`). Default `0` — untuk produk jualan biasa kirim `0`. |
| 🟠 | `weight_per_unit` | Float | Berat per unit produk (dipakai hitung `total_weight` pada transaksi) -> berat yang digunakan untuk perhitungan logistik/ongkos kirim. |
| 🟠 | `weight_uom` | Link → UOM | Satuan berat (mis. `Kg`, `Gram`) -> satuan produk yang digunakan untuk perhitungan logistik/ongkos kirim. |
| 🟠 | `image` | AttachImage | **Foto utama** produk — disimpan sebagai **string URL file** (mis. `/files/kaos.jpg` atau URL lengkap). Foto tambahan (multi) disimpan di `tabFile` (§4.8). |
| 🟠 | `allow_negative_stock` | Check | `1` = izinkan stok menjadi negatif (transaksi tetap jalan walau stok kurang). Default `0`. |
| 🟠 | `has_serial_no` | Check | `1` = tiap unit produk punya **nomor seri unik** (dilacak per unit via doctype `Serial No`; hanya untuk item stok). Default `0`. Menjadi **read-only** setelah Item punya riwayat stok (§2.1 no. 3). |
| 🟠 | `has_batch_no` | Check | `1` = produk dikelola per **Batch** (stok & `batch_no` dilacak per batch; item ber-batch memakai doctype `Batch`). **Wajib diisi `1` jika `has_expiry_date` diisi `1`** — backend menolak bila kedaluwarsa diaktifkan tanpa batch. Menjadi **read-only** setelah Item punya riwayat stok (§2.1 no. 3). |
| 🟠 | `create_new_batch` | Check | `1` = **setiap baris** transaksi masuk (mis. penerimaan) otomatis **membuat 1 Batch** baru — nomornya di-*generate* dengan pola bawaan `BATCH-00001` (§2.1 no. 12). Dipakai bersama `has_batch_no = 1`. Bila `0` padahal nomor batch belum ditentukan, backend melempar *"Batch ID is mandatory"*. |
| 🟠 | `batch_number_series` | Data | **Opsional — pola nomor Batch khusus Item ini.** Bila diisi, ia **menang** atas pola `Stock Settings` (mis. `ROLL.-.YY.-.MM.-.#` → `ROLL-26-10-1`). **Kosong = pakai pola bawaan** `BATCH-00001` (§2.1 no. 12). **Jangan pakai placeholder `{...}`** — dokumen `Batch` tidak punya field `item_code`, jadi `{item_code}` menghasilkan kosong. |
| 🟠 | `has_expiry_date` | Check | `1` = produk punya tanggal kedaluwarsa (per batch). **Jika diisi `1`, `has_batch_no` wajib ikut diisi `1`** (lihat baris `has_batch_no`). Memunculkan field `shelf_life_in_days`. |
| 🟠 | `shelf_life_in_days` | Int | **Umur simpan dalam hari** — hanya relevan (muncul) saat `has_expiry_date = 1`. Berfungsi sebagai **preset/template**: ketika **Batch baru dibuat** untuk Item ini, `expiry_date` Batch dihitung **otomatis** oleh backend (mis. dari tanggal produksi/manufaktur + `shelf_life_in_days`). |
| 🟠 | `reorder_levels` | Table (child `Item Reorder`) | **Batas stok per (item, warehouse)** — ambang **min & maks**. Baris: `warehouse` (Link→Warehouse, `reqd`); `warehouse_reorder_level` (Float = **nilai ambang minimum**, satuan `stock_uom`); `material_request_type` (`reqd` — Select `Purchase`/`Transfer`/`Material Issue`/`Manufacture`, menentukan cara restock; `Purchase` = beli ke supplier); `warehouse_reorder_qty` (Float, **opsional** — untuk alur pemesanan nanti, PRD stok §9); `max_stock_level` (Float, **opsional** — **custom field dari app `baseapp`**, nilai ambang maksimum, satuan `stock_uom`; hanya ada bila `baseapp` terpasang, lihat §4.1). Hanya relevan untuk item stok (`is_stock_item=1`). Cara simpan: §4.1 (contoh CREATE). |
| 🟠 | `brand` | Link → Brand | Merek produk (opsional; bisa jadi sumber default via `brand_defaults`). |
| 🟠 | `description` | TextEditor | Deskripsi produk (HTML). Backend membersihkan HTML bila kosong/rapi. |
| 🟠 | `hashtags` | Table (child `Item Hashtag`) | **Hashtag produk** — satu baris = satu hashtag, tiap baris **Link ke master `Hashtag`**, ditulis **tanpa `#`** dan huruf kecil (mis. `promo`, `best-seller`). **Custom field dari app `baseapp`** (fieldname tanpa prefix `custom_`, posisi setelah `description`); hanya ada bila app tersebut terpasang. **Inilah field untuk tag/hashtag — bukan `_user_tags`.** Berbeda dari `_user_tags` (satu kolom teks dipisah koma), `hashtags` bisa dicari **persis**: filter `= promo` tidak ikut menarik `promo2026`. Nilai yang belum terdaftar di master **ditolak backend**. Master `Hashtag` (daftar pilihan & CRUD): §4.12f. Format, normalisasi, & cara filter: §2.8. CRUD + pola satu halaman (template & semua varian): §4.12. |
| ⛔ **Jangan dipakai** | `_user_tags` | Tags (kolom sistem) | Kolom sistem Frappe yang otomatis ada di semua tabel — **bukan tempat menyimpan hashtag produk**, karena isinya satu kolom teks dipisah koma sehingga pencarian persis tidak mungkin (§2.6). Pakai `hashtags` di atas. |
| 🟠 | `valuation_rate` | Currency | **Nilai persediaan per unit** (harga modal / biaya masuk stok; dipakai hitung `stock_value` di `tabBin`). Bisa diisi 0 untuk item baru / zero valuation. Saat Item dibuat, baseapp menyinkronkannya ke `price_list_rate` pada **`Standard Buying`**; perubahan harga umum UOM stok di daftar tersebut juga memperbarui field ini (detail: [prd_item_price.md §4.9](./prd_item_price.md)). |
| 🟠 | `standard_rate` | Currency | **Standard Selling Rate** — label field ERPNext; ini **harga jual**. Saat Item dibuat, baseapp menyinkronkannya ke `price_list_rate` pada **`Standard Selling`**; perubahan harga umum UOM stok di daftar tersebut juga memperbarui field ini. ERPNext sendiri juga membuat Item Price dari `standard_rate` pada Price List selling default (`Item.after_insert → add_price`). Detail: [prd_item_price.md §4.9](./prd_item_price.md). |
| ⚪ **Otomatis — jangan dikirim** | `name` | — | `name = item_code`. |
| ⚪ **Dikelola sistem** | `valuation_rate` (stok berjalan), `last_purchase_rate`, `total_projected_qty` | — | Nilai dihitung/di-update dari transaksi stok. Selain itu, hook baseapp menyinkronkan `valuation_rate` dengan Item Price `Standard Buying` sesuai aturan di atas. |
| ⚪ **Set saat varian** | `variant_of`, `variant_based_on`, `attributes` | — | Lihat §4.2. |
| ✖️ **Bukan bagian scope** | `opening_stock`, `taxes`, `item_defaults`, dst. | — | Field lanjutan boleh dipakai, tetapi mekanisme **stok** (opening stock, auto-reorder) dan **price list** didokumentasikan di PRD terpisah (§9). Penyimpanan ambang `reorder_levels` sudah dicakup di dokumen ini (lihat baris `reorder_levels` di atas & §4.1). |

> Catatan: daftar di atas adalah **data yang wajib/diperlukan** menurut kebutuhan aplikasi
> (per permintaan tim produk), bukan seluruh field Item. Field `reqd` sebenarnya oleh doctype hanya
> `item_group` dan `stock_uom` — `item_code` **dibuat backend** (tidak wajib di form karena
> `hidden`), sedangkan `item_name` menjadi **praktis wajib** karena kode produk berupa angka seri
> (§2.1 no. 1). Sisanya opsional di sisi backend, namun **frontend tetap disarankan mengirim**
> sesuai tabel di atas agar data konsisten.

### 2.1 Catatan penting

1. **`item_code` dibuat backend lewat naming series `YY.MM.######` — jangan kirim dari frontend.**
   Item dikonfigurasi memakai **Naming Series** (diatur app `baseapp`:
   `Stock Settings → Item Naming By = "Naming Series"` + opsi field `Item.naming_series`), sehingga
   backend **selalu** membuat `item_code` sendiri dengan pola:

   - `YY` = 2 digit tahun, `MM` = 2 digit bulan, `######` = counter 6 digit (**reset setiap bulan**).
   - Item pertama bulan Sep 2026 → **`2609000001`**, berikutnya `2609000002`, dst.

   **Yang harus dipahami frontend:**

   - **`item_code` yang dikirim frontend ditimpa** (tidak berpengaruh). Karena itu **pre-check
     duplikat tidak lagi diperlukan** (§4.1).
   - **Termasuk varian — tidak ada pengecualian.** Item dengan `variant_of` terisi **juga** mendapat
     kode seri sendiri, **bukan** `{kode template}-{abbr}` (perilaku bawaan ERPNext). Jadi item
     tunggal, template dan varian semuanya memakai **satu counter bulanan yang sama** dan kodenya
     selalu **10 digit**. Detail: §4.2.
   - **Tidak ada duplikat.** Counter dikelola tabel `Series` dan dikunci saat dipakai, sehingga
     `DuplicateEntryError` untuk `item_code` praktis tidak mungkin muncul dari sisi klien.
   - **Nomor bisa melompat.** `set_new_name()` dijalankan **sebelum** validasi
     (`frappe/model/document.py:479` vs `485`), jadi Item yang gagal disimpan tetap **memakai** satu
     nomor seri. Lompatan nomor bukan error — jangan mengasumsikan nomor selalu berurutan tanpa celah.
   - **`name` = `item_code`** → **simpan `name` dari respons CREATE**; itulah identitas item untuk
     semua operasi berikutnya.
   - Error *"Item Code is required"* **tidak akan muncul lagi** — field `item_code` kini `hidden` dan
     tidak wajib di form (§7).
   - Kode bisa **bertambah 1 karakter** saat counter melewati 6 digit (setelah `2609999999` menjadi
     `26091000000`) — bukan error, hanya perlu diperhatikan bila kode dicetak sebagai barcode.
   - Mengubah kode pada item yang sudah ada **tidak mengubah `name`**; bila benar-benar perlu,
     gunakan `frappe.client.rename_doc`.
   - **Item lama tidak berubah.** Aturan ini hanya berlaku untuk Item yang dibuat **setelah** versi
     app `baseapp` ini terpasang. Item yang sudah ada tetap memakai kodenya semula — termasuk yang
     berbentuk `{template}-{abbr}` seperti `BUB-PB-BLU-BIG`. Jadi frontend **tidak boleh** menyimpulkan
     bentuk apa pun dari `item_code`; selalu pakai nilainya apa adanya dari `name`.
   - **Konsekuensi penting:** karena kode produk kini berupa angka seri yang tidak deskriptif,
     **`item_name` wajib dikirim** — kalau kosong, nama produk akan tersimpan sebagai `2609000001`.
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
7. **Barcode default = `item_code` — dibuat otomatis backend saat CREATE.**
   Diatur app `baseapp` (hook `Item.before_validate`), sehingga **tidak ada Item baru tanpa
   barcode**.

   | Kondisi saat CREATE | Hasil |
   |---|---|
   | `barcodes` **tidak dikirim** | backend menambah 1 baris: `barcode = item_code`, `uom = stock_uom`, `barcode_type` **kosong** |
   | `"barcodes": []` (list kosong) | sama seperti di atas — list kosong dianggap "tidak ada barcode" |
   | `barcodes` **dikirim** | payload dipakai apa adanya; **tidak** ditambahi baris `item_code` |

   **Yang harus dipahami frontend:**

   - **Hanya saat CREATE.** Item yang sudah ada **tidak** diperbaiki. `save` bersifat replace-all
     (§4.6), jadi `save` tanpa key `barcodes` akan **menghapus** barisnya dan backend **tidak**
     menambahkannya kembali — saat update, kirim seluruh baris yang diinginkan (§4.7).
   - **`barcode_type` sengaja dikosongkan.** Bila diisi, backend menjalankan uji check digit
     (`barcodenumber`) dan menolak semua nilai yang bukan tepat 8/12/13 digit **dengan check digit
     yang benar**. `2609000001` (10 digit) akan gagal `InvalidBarcode` dan **seluruh insert Item ikut
     gagal** — bukan hanya barcodenya. Kalau frontend perlu mencetak EAN-13, hitung dulu check
     digit-nya: `899123456789` → `8991234567891` (sedangkan `8991234567890` **tidak valid**).
   - **Cetak pakai Code 128 subset C.** Untuk 10 digit hasilnya ±90 modul, jauh lebih tipis dari
     Code 39 (±156 modul, dan `CODE-39` memang lolos validasi backend). EAN-13 (95 modul) **tidak**
     lebih hemat untuk panjang ini — hanya EAN-8 (67 modul) yang lebih tipis, tapi menuntut kode
     7 digit.
   - **Lebar barcode default seragam.** Karena varian juga memakai kode seri 10 digit
     (§2.1 no. 1), semua barcode default berukuran sama. Barcode kiriman frontend boleh berapa pun
     panjangnya — backend tidak membatasi — dan hanya item itulah yang lebar labelnya berbeda.
   - **Barcode unik global.** Karena setiap Item otomatis punya barcode = `item_code`, mengirim
     barcode manual yang sama dengan `item_code` Item lain ditolak: *"Barcode X already used in
     Item Y"* (§7).
   - **Sudah bisa langsung dipindai.** `scan_barcode()` mencari di `Item Barcode` lebih dulu
     (`erpnext/stock/utils.py`), jadi memindai `2609000001` mengembalikan Item yang tepat.
   - **Mengubah `abbr` tidak lagi me-rename varian.** ERPNext aslinya me-rename `item_code` varian
     kembali ke bentuk `{kode template}-{abbr}` (`rename_variant_item_code`) setiap kali `abbr`
     sebuah `Item Attribute Value` diubah; app `baseapp` mematikan jalur itu lewat override
     controller `ItemAttribute.on_update` (`baseapp/overrides/item_attribute.py`). Jadi baris
     `Item Barcode` tetap cocok dengan kode serinya — kode seri tidak pernah dihitung ulang. Catatan:
     `abbr` kini selalu sama dengan `attribute_value` (§4.2 Langkah 1).
8. **`item_name` harus unik di antara Item yang aktif — ERPNext tidak memeriksanya sendiri.**
   app `baseapp` menambahkan aturan ini (hook `Item.validate` + endpoint pre-check §4.11), dengan
   dua cabang:

   | Kondisi saat CREATE/UPDATE | Hasil |
   |---|---|
   | Nama dipakai Item **aktif** (`disabled=0`) | **Ditolak** 417 — *"Item Name X is already used by active Item(s) Y"* |
   | Nama hanya dipakai Item **non-aktif** (`disabled=1`) | **Diizinkan** — backend tidak menghalangi, keputusan lewat pertanyaan reaktifasi (§4.11) |
   | Nama bebas | Lanjut seperti biasa |

   **Yang harus dipahami frontend:**

   - **Selalu pre-check dulu** (§4.11) sebelum CREATE/UPDATE. Untuk kasus "hanya non-aktif", backend
     **sengaja tidak menolak** — jadi kalau frontend tidak bertanya, akan muncul dua Item bernama
     sama (satu non-aktif, satu baru).
   - **Perbandingan case-insensitive & mengabaikan spasi tepi.** `"Air Mineral 600ml"`,
     `"air mineral 600ml"`, dan `"  Air Mineral 600ml  "` dianggap nama yang sama. Spasi **di tengah**
     tetap dihitung: `"Air  Mineral"` ≠ `"Air Mineral"`.
   - **Ini bukan aturan ERPNext bawaan.** Di ERPNext aslinya **tidak ada pengecekan sama sekali**:
     hanya `item_code` yang `unique` (`item.json`), `Item.validate()` tidak melihat duplikat, dan
     `item.js` juga tidak. Aturan ini hilang bila app `baseapp` tidak terpasang.
   - **Item lama tidak dibersihkan.** Nama kembar yang sudah ada sebelum aturan ini dipasang tetap
     dibiarkan; yang dicegah hanya penambahan baru.
   - **`save` dan `set_value` dua-duanya menjalankan aturan ini.** `frappe.client.set_value`
     memuat dokumen lalu memanggil `doc.save()` (`frappe/client.py:215`), sehingga `validate()`
     tetap berjalan — **terverifikasi di site dev 2026-10-03**: mengubah `item_name` ke nama yang
     sudah dipakai Item aktif lewat `set_value` ditolak 417. Yang benar-benar melewati hook
     hanyalah **`frappe.db.set_value`** (level DB, dan tidak dapat dipanggil dari REST).
9. **Menyaring daftar produk — paket dinamis / paket statis / produk manufaktur / aset — cukup
   **satu** panggilan `frappe.client.get_list`.** Semuanya kolom **Check** yang bisa difilter
   langsung:

   | Jenis produk | Cara dideteksi |
   |---|---|
   | Paket dinamis | `is_dynamic_product_bundle = 1` — custom field `baseapp`, **diisi manual** |
   | Paket statis (`Product Bundle`) | `is_product_bundle = 1` — custom field `baseapp`, **diisi manual** |
   | Produk manufaktur | `is_manufactured_item = 1` — custom field `baseapp`, **diisi manual** (bukan turunan BOM) |
   | Aset tetap | `is_fixed_asset = 1` — field standar ERPNext |

   Keempatnya kolom **Check**, jadi bisa digabung dengan **`filters` + `or_filters`** dalam satu
   request (contoh di bawah memakai 3 jenis):

   ```bash
   curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
     -H 'Authorization: Bearer <access_token>' \
     -H 'Content-Type: application/json' \
     -d '{
       "doctype": "Item",
       "fields": ["name","item_name","is_dynamic_product_bundle","is_product_bundle","is_fixed_asset"],
       "filters": [["disabled","=",0]],
       "or_filters": [
         ["is_dynamic_product_bundle","=",1],
         ["is_product_bundle","=",1],
         ["is_fixed_asset","=",1]
       ],
       "order_by": "name asc",
       "limit_start": 0,
       "limit_page_length": 50
     }'
   ```

   **Yang harus dipahami frontend:**

   - **`filters` di-AND, `or_filters` di-OR** — hasil OR lalu di-AND dengan `filters`.
   - **`is_product_bundle` diisi manual.** Dulu baseapp menurunkannya dari doctype `Product Bundle`
     (hook `on_update`/`on_trash`); logika itu sudah **dihapus** — membuat, mengubah,
     mengaktifkan/menonaktifkan, atau menghapus `Product Bundle` **tidak lagi mengubah** flag ini.
     Nilainya kini ditentukan user/frontend, jadi field ini **boleh dikirim** saat
     `insert`/`save`/`set_value` Item, sama seperti `is_dynamic_product_bundle` dan
     `is_manufactured_item`.
   - **Kenapa tetap perlu field tersimpan.** `frappe.client.get_list` **tidak bisa** menjangkau tabel
     lain — DSL filter Frappe hanya menyediakan `=`, `!=`, `<`, `>`, `<=`, `>=`, `in`, `not in`,
     `like`, `ilike`, `not like`, `regex`, `between`, `is`, `timespan`
     (`frappe/database/operator_map.py:139-159`), tanpa subquery/join. Jadi "Item yang punya baris di
     `tabProduct Bundle`" tidak bisa diungkapkan dari record Product Bundle-nya. ERPNext sendiri juga
     **tidak** menandai Item sebagai bundle (`item.json` tidak punya field bundle sama sekali, dan
     `product_bundle.py` tidak pernah menulis balik ke Item — `validate_main_item()` hanya membaca
     `is_stock_item`/`is_fixed_asset`), dan `is_stock_item = 0` **bukan** pengganti: di site dev ada
     12 item non-stok, 11 di antaranya bundle dan 1 bukan. Karena itu penandanya disimpan di kolom
     Item — hanya saja kini **diisi user**, bukan dihitung backend.
   - **Keempatnya independen.** `is_dynamic_product_bundle`, `is_product_bundle`, dan
     `is_manufactured_item` diisi manual; `is_fixed_asset` adalah field standar ERPNext. Tidak ada
     yang dihitung ulang backend, jadi jangan mengasumsikan satu produk hanya masuk satu kategori —
     dan nilai yang salah **tidak** akan "sembuh sendiri"; perbaiki lewat `save`/`set_value`.
   - **Pagination: kirim `or_filters` juga ke `frappe.client.get_count`.** Parameter itu memang
     **tidak terdaftar** di signature-nya (`frappe/client.py:79` hanya `doctype, filters, debug,
     cache`), tapi tetap dibaca — karena `reportview.get_count()` membaca `frappe.form_dict`, yaitu
     **body request itu sendiri**, sementara `get_count` hanya menimpa `doctype`/`filters`/`debug` dan
     tidak membersihkan sisanya. **Terverifikasi di site dev 2026-10-03:** `get_list` dengan 3 kondisi
     OR mengembalikan **6** baris, dan `get_count` dengan body yang sama juga mengembalikan **6**
     (bukan 52 = jumlah seluruh item aktif).

     ```bash
     curl -X POST https://site-anda.com/api/method/frappe.client.get_count \
       -H 'Authorization: Bearer <access_token>' \
       -H 'Content-Type: application/json' \
       -d '{
         "doctype": "Item",
         "filters": [["disabled","=",0]],
         "or_filters": [
           ["is_dynamic_product_bundle","=",1],
           ["is_product_bundle","=",1],
           ["is_fixed_asset","=",1]
         ]
       }'
     ```

     > ⚠️ Perilaku ini **tidak terdokumentasi** (efek samping `form_dict`, bukan API resmi), jadi
     > jangan jadikan satu-satunya sandaran: **selalu kirim `filters` bersamaan** (boleh `[]`) dan
     > siapkan fallback bila total halaman terasa janggal — `get_list` dengan
     > `limit_page_length: 0` + `fields: ["name"]`, lalu hitung panjang array.
     > Alternatif paling tahan lama: pisah jadi **tiga tab** di UI — tiap tab cuma butuh `filters`
     > biasa, sehingga `frappe.client.get_count` standar sudah akurat tanpa bergantung pada perilaku
     > di atas.
10. **Mengganti nama template (produk) otomatis mengganti nama semua variannya.**
    Diatur app `baseapp` (hook `Item.on_update` → `baseapp.utils.sync_variant_item_names`). Ini
    **melengkapi** ERPNext, yang sengaja tidak melakukannya: `copy_attributes_to_variant()`
    (`erpnext/controllers/item_variant.py:449`) mencantumkan `item_name` di `exclude_fields`,
    sehingga template bisa berganti nama sementara variannya tetap memakai nama template yang lama
    (`KAOS-M` di bawah produk `T-SHIRT`).

    | Yang diubah pada **template** | Efek ke `item_name` varian |
    |---|---|
    | `item_name` (mis. `KAOS` → `T-SHIRT`) | ✅ dihitung ulang: `KAOS-M` → **`T-SHIRT-M`** |
    | `item_code`/`name` (rename dokumen) | ❌ tidak ada — nama varian tidak diturunkan dari kode |
    | `item_group`, `stock_uom`, dst. | lewat mekanisme bawaan ERPNext (§4.6), bukan dari dokumen ini |

    **Yang harus dipahami frontend:**

    - **Pola nama sama dengan saat varian dibuat** — `{item_name template}-{abbr}`. Dihitung ulang
      dengan fungsi yang sama (`make_variant_item_code`), jadi hasilnya identik dengan varian baru
      (termasuk atribut numerik yang memakai nilainya, bukan abbr).
    - **Berlaku untuk semua varian**, termasuk yang `item_name`-nya pernah diisi manual — nama
      varian selalu diturunkan dari template.
    - **Hanya saat nama berubah.** Varian yang sudah ada **tidak** dirapikan otomatis: kalau data
      lama telanjur tidak sinkron, itu harus diperbaiki manual (ubah `item_name` variannya).
    - **Bentrok nama membatalkan seluruh proses.** Bila nama hasil hitungan sudah dipakai Item
      **aktif** lain — atau dua varian akan menghasilkan nama yang sama — backend mengembalikan
      **417** (*"Variant Name Clash"*) dan **tidak ada** yang berubah, agar aturan unik `item_name`
      (no. 8) tetap terjaga. Frontend sebaiknya menampilkan pesan error apa adanya, karena pesannya
      menyebut varian mana yang bentrok.
    - **`item_code` dan barcode varian tidak tersentuh** — hanya `item_name` yang ditulis
      (`update_modified` Item juga tidak berubah).
    - **Setting `Item Variant Settings → Do not update variants on save` tidak berlaku di sini** —
      setting itu mengatur penyalinan *field* template ke varian, bukan penamaan.
    - **Terverifikasi di site dev 2026-10-03:** template `ZZT KAOS` dengan 2 varian di-rename ke
      `ZZT T-SHIRT` → varian menjadi `ZZT T-SHIRT-M` dan `ZZT T-SHIRT-B`; saat Item aktif bernama
      `ZZT SHIRT-B` sudah ada, rename berikutnya ditolak dan seluruh data kembali utuh.
11. **`min_sales_qty`, `max_sales_qty`, dan `sales_qty_multiple` adalah custom field bawaan app `baseapp`.**
    Ketiganya bertipe `Float`, default `0`, dan memakai `stock_uom` sebagai satuan. Field muncul
    setelah `baseapp` dipasang dan tersedia pada REST API Item. Field ini menyimpan konfigurasi
    jumlah jual; validasi batas/kelipatan pada transaksi penjualan tidak dilakukan otomatis hanya
    dengan menambahkan field ini.
12. **Nomor Batch saat penerimaan — default ERPNext vs input user.** Aturannya hanya dua:

    | Kondisi di dokumen penerimaan | `batch_id` yang dipakai |
    |---|---|
    | User **tidak** mengisi nomor batch | dibuat otomatis dengan pola bawaan ERPNext → `BATCH-00001`, `BATCH-00002`, … |
    | User **mengisi** nomor batch | memakai input user apa adanya (mis. `PNM0107-001`) |

    Item harus `has_batch_no = 1` **dan** `create_new_batch = 1` agar nomornya bisa dibuat — bila
    `create_new_batch = 0` dan nomornya kosong, backend menolak *"Batch ID is mandatory"*.

    **Pola otomatis (`BATCH-00001`) datang dari `Stock Settings`, bukan dari Item:**

    | Field `Stock Settings` | Nilai | Arti |
    |---|---|---|
    | `use_naming_series` | **`1`** | saklar *"Have default Naming Series for Batch ID?"* — **wajib `1`** untuk pola ini; kalau `0` nomor batch menjadi **hash acak 7 karakter** (mis. `A3F91C2`) |
    | `naming_series_prefix` | `BATCH-` | awal nomor, ditulis **apa adanya** — `LOT-` → `LOT-00001`, sedang `LOT` → `LOT00001` |

    - Pola efektifnya `{prefix}.#####`, sehingga **satu counter dipakai untuk SEMUA Item** ber-batch
      (bukan per Item).
    - **`use_naming_series` hanya mengatur penamaan Batch** — satu-satunya pembacanya `Batch.autoname()`;
      tidak memengaruhi penamaan Item, Serial No, Serial and Batch Bundle, atau dokumen lain.
    - **`batch_number_series` di Item bersifat opsional** dan **menang** atas `Stock Settings`
      (mis. `ROLL.-.YY.-.MM.-.#` → `ROLL-26-10-1`). Isi hanya bila Item tertentu perlu pola sendiri.
      Token yang dikenali: `.YY.`/`.MM.`, `.#`, `.#####`; series wajib memuat titik. **Jangan pakai
      placeholder `{...}`** — saat Batch dibuat, Frappe memakai `doc` = dokumen `Batch` yang **tidak
      punya field `item_code`**, sehingga `{item_code}` menghasilkan kosong.
    - Nomor batch yang **diketik user** tidak lewat `batch_number_series` (field itu statis di Item);
      frontend membuat dokumen `Batch`-nya lebih dulu (§4.3 Langkah 3).
    - Nama/nomor batch ada di field **`batch_id`**, dan pada versi ERPNext yang terpasang (`16.32.3`)
      nilai itu **= `name` dokumen** (`Batch.autoname()` menetapkan `name = batch_id`). Pada baris
      transaksi, `batch_no` diisi nilai tersebut.
    - Contoh lengkap kedua aturan: **§4.3**.

### 2.2 Lokasi tabel penyimpanan (data Item & pendukungnya)

| Tabel (`tab...`) | Doctype | Menyimpan |
|---|---|---|
| `tabItem` | `Item` | Master produk — **template & varian** dalam satu tabel (varian ditandai kolom `variant_of`, `has_variants`, `variant_based_on`). |
| `tabItem Variant Attribute` | `Item Variant Attribute` | Child table `attributes` — atribut & nilai per Item (template: tanpa nilai; varian: dengan nilai). |
| `tabItem Attribute` | `Item Attribute` | Master atribut (mis. `Ukuran`, `Warna`) — parent dari `tabItem Attribute Value`. |
| `tabItem Attribute Value` | `Item Attribute Value` | Child table nilai atribut pada master (mis. `Small`, `Medium`, `Large`; kolom `abbr` diisi otomatis **= `attribute_value`**, §4.2 Langkah 1). |
| `tabUOM Conversion Detail` | `UOM Conversion Detail` | Child table `uoms` pada Item — faktor konversi UOM lain → `stock_uom` (§2.3, §5). |
| `tabUOM Conversion Factor` | `UOM Conversion Factor` | Master global pasangan konversi antar-UOM (mis. `Gram`→`Kg`) — dipakai `get_uom_conv_factor` (§5). |
| `tabItem Barcode` | `Item Barcode` | Child table `barcodes` — daftar barcode produk (§2.4, §4.7). |
| `tabItem Supplier` | `Item Supplier` | Child table `supplier_items` — daftar pemasok untuk Item (§2.7, §4.1/§4.4/§4.6). |
| `tabItem Hashtag` | `Item Hashtag` | Child table `hashtags` — **hashtag produk**, satu baris per hashtag (kolom `hashtag`, **Link → `Hashtag`** + **index**). Dipakai untuk pencarian **persis** (§2.8, §4.12). |
| `tabHashtag` | `Hashtag` | **Master hashtag** — `name` = hashtag itu sendiri (mis. `promo`). Sumber dropdown hashtag dan target Link baris `Item Hashtag` serta `Dynamic Product Bundle Hashtag` (§2.8, §4.12f). |
| `tabFile` | `File` | **Semua attachment** (foto multi produk), ter-link ke Item via `attached_to_doctype` + `attached_to_name` (§4.8). |
| `tabBin` | `Bin` | **Stok per (item, warehouse)** — `actual_qty`, `projected_qty`, `stock_value`, dsb. (§4.10). |
| `tabBatch` | `Batch` | Master **nomor batch** — field `batch_id` (nomor yang dibaca user; pada versi terpasang = `name` dokumen, §2.1 no. 12) + `item` pemiliknya. Stok per batch tidak di sini, melainkan di Stock Ledger Entry / Serial and Batch Bundle. |
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
- **Barcode default = `item_code`** — app `baseapp` menambahkannya otomatis saat CREATE bila
  frontend tidak mengirim `barcodes` (§2.1 no. 7).
- **Barcode bersifat unik global** — backend menolak barcode yang sudah dipakai Item lain
  (`DuplicateEntryError`/`InvalidBarcode`).
- **Panjang tidak dibatasi** — `barcode` bertipe `varchar(140)`, dan `barcode_type` yang dikosongkan
  mematikan uji check digit.
- CRUD lengkap: **§4.7**.

### 2.5 Child table `attributes` — `Item Variant Attribute` (+ master `Item Attribute`)

- Pada **template** (`has_variants=1`): baris `attributes` hanya berisi `attribute` (Link → Item
  Attribute) **tanpa nilai**.
- Pada **varian** (`variant_of` terisi): baris berisi `attribute` + `attribute_value` (nilai konkret),
  dan `attribute_value` tidak bisa diedit setelah varian dibuat.
- Master `Item Attribute` (doctype terpisah) menyimpan daftar nilai lewat child table
  `item_attribute_values` (kolom `attribute_value` + `abbr`). `abbr` dipakai membangun `item_name`
  varian otomatis (§4.2) — bukan `item_code` lagi, karena kode varian kini dibuat backend.
- **`abbr` selalu sama dengan `attribute_value` dan diisi otomatis backend** (app `baseapp`, hook
  `Item Attribute.before_validate` → `baseapp.utils.sync_attribute_value_and_abbr`): frontend
  **cukup mengirim `attribute_value`**. Di form Desk kolom `abbr` di-hide & tidak wajib (property
  setter dari `baseapp`). Karena `abbr` = nilai atribut, `item_name` varian mengikuti **nilai**
  tersebut (`Medium` → `Kaos Polos-Medium`). Detail & contoh: §4.2 Langkah 1.
- Alur & contoh kasus lengkap: **§4.2**.

### 2.6 `_user_tags` (Tags) — ⛔ **bukan** tempat menyimpan hashtag

> **Peringatan.** `_user_tags` **jangan** dipakai untuk menyimpan hashtag/tag produk.
> Gunakan child table **`hashtags`** (§2.8).

- `_user_tags` adalah **kolom sistem** yang otomatis ada di setiap tabel Frappe (bukan field yang
  didefinisikan di `item.json`). Tipe tersimpan `Data`, isi berupa **tag dipisah koma**, mis.
  `"best seller,baru,diskon"`.
- Di form Desk dikelola lewat kontrol "Tags"; lewat API cukup dikirim sebagai field biasa pada
  `doc` (CREATE/UPDATE). Tidak ada enumerasi khusus — bebas teks.
- **Kenapa dilarang untuk hashtag — pencarian persis tidak mungkin.** Semua tag berada di **satu
  kolom teks** yang digabung koma:
  - Filter `[["_user_tags","like","%promo%"]]` **ikut menarik `promo2026`** (juga
    `promo-diskon`, `promoakhir-tahun`, dst.). Padahal kebutuhan user: cari yang bertag
    **`#promo` saja** — `#promo2026` tidak boleh muncul. Hasilnya salah.
  - Filter yang benar untuk satu kolom teks gabungan hanya `regex`
    (mis. `rlike '(^|,| )promo(,| |$)'`), dan **`regex` tidak bisa memakai index** → setiap
    pencarian memindai **seluruh** tabel Item.
- Karena itu hashtag disimpan di **child table `hashtags`** (§2.8): satu baris = satu hashtag,
  sehingga filter **`=`** bisa dipakai, kolomnya **ter-index**, dan hasilnya tepat.
- `_user_tags` tetap boleh dipakai untuk kebutuhan lain yang tidak menuntut pencarian persis
  (mis. catatan/label internal yang jarang difilter), tetapi **bukan** untuk fitur hashtag produk.
- Catatan teknis: setiap nilai di `_user_tags` juga membuat dokumen `Tag` + baris `Tag Link`,
  jadi menyimpan banyak hashtag di sana menambah dua tabel lain yang harus ikut dibersihkan.

### 2.7 Pemasok per Item — child table `supplier_items` (`Item Supplier`)

- Pemasok untuk suatu Item disimpan sebagai child table **`supplier_items`**, dengan doctype
  **`Item Supplier`** (`tabItem Supplier`). Satu Item dapat memiliki beberapa pemasok.
- Setiap baris wajib menunjuk `supplier` yang sudah ada di master **Supplier**. Field
  `supplier_part_no` (opsional) menyimpan kode Item menurut pemasok.
- Kelola daftar ini melalui dokumen induk `Item` (`insert`, `get`, dan `save`), bukan dengan
  membuat atau menghapus `Item Supplier` sebagai dokumen mandiri. GET Item mengembalikan seluruh
  baris `supplier_items`; saat UPDATE, kirim daftar lengkap yang ingin dipertahankan (replace-all).
- Contoh CRUD lengkap: **§4.1, §4.4, dan §4.6**.

### 2.8 Child table `hashtags` — `Item Hashtag` (hashtag produk)

Untuk kebutuhan pencarian yang **persis** (mis. "hanya Item bertag `#promo`", tanpa ikut
menampilkan `#promo2026`), hashtag disimpan di **child table**, bukan di kolom teks (§2.6).
**Satu baris = satu hashtag.**

**Bentuk field**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `hashtag` | **Link → `Hashtag`** (+ **index**) | ya (`reqd=1`) | Menunjuk satu dokumen master `Hashtag`. Nilainya adalah **nama dokumen** master tsb: huruf kecil, tanpa `#`, tanpa spasi. Contoh: `promo`, `best-seller`, `new_arrival`. |

Field pada Item bernama `hashtags` — tipe **Table → `Item Hashtag`**, custom field dari app
`baseapp` (fieldname tanpa prefix `custom_`, posisi setelah `description`; lihat
[prd_baseapp.md §4.1/§4.5](../baseapp/prd_baseapp.md)). Yang punya daftar sendiri: **template
maupun setiap varian**.

**Master `Hashtag` — sumber daftar nilainya.** `Item Hashtag.hashtag` **bukan** teks bebas lagi,
melainkan **Link ke master `Hashtag`** (doctype biasa, `name` = hashtag itu sendiri, pola sama
seperti `Brand`). Konsekuensinya untuk frontend:

- **Dropdown diambil dari master**, bukan dari nilai yang kebetulan sudah dipakai Item —
  `frappe.client.get_list` doctype `Hashtag`, `fields: ["name"]` (§4.12f, §6.8).
- **User hanya boleh memilih yang sudah terdaftar.** Mengirim hashtag yang belum ada di master
  **ditolak** (lihat tabel di bawah), kecuali pada jalur instalasi/migrasi/patch.
- **Menambah/mengganti/menghapus hashtag master** memakai CRUD biasa di doctype `Hashtag`
  (`frappe.client.insert`/`save`/`rename_doc`/`delete`) — §4.12f.
- `rename_doc` pada master ikut memperbaiki semua baris Item yang menunjuk hashtag tsb; menghapus
  master **diblokir** (`LinkExistsError`) selama masih dirujuk Item (atau
  `Dynamic Product Bundle Hashtag`).

**Aturan isi — dirapikan backend otomatis**

Hook `Item.validate` → `baseapp.utils.normalize_item_hashtags` berjalan **sebelum** data
disimpan, jadi yang tersimpan selalu bentuk kanonik:

| Dikirim frontend | Tersimpan | Catatan |
|---|---|---|
| `"#Promo"` / `" Promo "` / `"promo"` | `promo` | tanda `#` dibuang, spasi dipangkas, huruf kecil |
| `""` atau `"#"` | *(baris dibuang)* | baris kosong tidak disimpan |
| `"promo"` dikirim 2× | `promo` (1 baris) | duplikat dibuang, yang pertama dipertahankan |
| `"promo 2026"` / `"promo!"` / `"Promo#2026"` | **ditolak (417)** | nilai valid harus cocok pola `^[a-z0-9_-]{1,50}$` |
| `"#Promo"` (belum ada di master `Hashtag`) | **ditolak (417)** | nilai sudah bersih (`promo`) tetapi belum terdaftar → buat dulu di master (§4.12f) |

Karena itu frontend **boleh** mengirim apa adanya dari input user (dengan/tanpa `#`, huruf besar),
selama karakternya termasuk yang diizinkan **dan** hashtag tsb sudah ada di master `Hashtag`. Pesan
error untuk nilai yang tidak valid:

```
ValidationError: Row #1: hashtag promo 2026 is not valid.
Use lowercase letters, digits, - and _ only (max 50 characters).
```

```
ValidationError: Row #1: hashtag promo belum terdaftar. Buat dulu di master Hashtag.
```

Nomor baris pada pesan mengikuti **urutan baris pada payload yang dikirim**, sehingga frontend bisa
langsung menunjuk baris yang salah.

**Perilaku saat simpan — replace-all**

- Seperti child table lain (`uoms`, `barcodes`), `hashtags` bersifat **replace-all**: kirim
  **seluruh daftar** yang ingin dipertahankan; baris yang tidak ikut terkirim **terhapus**.
- Mengirim `"hashtags": []` **menghapus semua** hashtag Item tsb.
- Menambah/menghapus hashtag **harus lewat dokumen Item** (`frappe.client.save` atau `insert`),
  karena `frappe.client.set_value` tidak bisa menyentuh child table (§4.6).

**Template & varian — tidak ada pewarisan otomatis**

- Setiap Item (template maupun tiap varian) punya daftar hashtag **sendiri**.
- **Template tidak mewariskan hashtagnya ke varian.** `copy_attributes_to_variant()` hanya
  menyalin field `reqd` atau field yang terdaftar di doctype `Variant Field`, dan `hashtags` bukan
  salah satunya. Hasil uji: template bertag `promo` → varian yang baru dibuat **tetap tanpa
  hashtag**.
- **Pewarisan (bila diinginkan) diatur frontend.** Satu halaman yang mengedit template + semua
  varian cukup memanggil `frappe.client.save` **per dokumen varian** dengan hashtag masing-masing.
  Pola lengkap baca & tulis: **§4.12**.

**Cara mencari — perilaku filter yang sudah diuji**

| Maksud | Filter | Hasil |
|---|---|---|
| Item bertag **persis** `promo` | `[["hashtags.hashtag","=","promo"]]` | hanya Item yang punya baris `promo` — **`promo2026` tidak ikut** ✅ |
| Item punya salah satu dari beberapa tag | `[["hashtags.hashtag","in",["promo","baru"]]]` | benar, tetapi **satu Item muncul berulang** (1 baris hasil per baris child yang cocok) → dedupe di frontend atau pakai `group_by` |
| Tag yang **mengandung** `promo` | `[["hashtags.hashtag","like","%promo%"]]` | `promo` **dan** `promo2026` — untuk pencarian persis jangan pakai `like` |
| Item punya tag `promo` **dan** `baru` | dua filter `=` pada `hashtags.hashtag` | **tidak bisa** — Frappe memakai `JOIN`, bukan `EXISTS`, sehingga dua kondisi pada fieldname yang sama menghasilkan **0 baris**. Lakukan 2 request lalu iris hasilnya di frontend |
| Jumlah Item bertag `promo` | `frappe.client.get_count` + filter `=` | akurat; **jangan** pakai `in` (baris hasil JOIN dihitung ganda) — contoh di §4.12 |
| Daftar hashtag unik (autocomplete) | `fields=["hashtags.hashtag"]` + `group_by=hashtags.hashtag` | lihat **§6.8** |

> Contoh request lengkap beserta angkanya: **§4.12** (baca/tulis per Item & per varian) dan
> **§6.8** (autocomplete).

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

> **Tidak ada pre-check duplikat.** `item_code` dibuat backend lewat naming series `YY.MM.######`
> (§2.1 no. 1) dengan counter yang dikelola sistem, sehingga kode **tidak mungkin duplikat**.
> Frontend cukup mengirim data produk — tidak perlu mengirim `item_code` maupun mengecek keberadaan
> kode terlebih dahulu.

**Payload minimum (data wajib) — `frappe.client.insert`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_name": "Air Mineral 600ml",
      "item_group": "Minuman",
      "stock_uom": "Pcs"
    }
  }'
```

> **Tidak ada `item_code`** — backend membuatnya (§2.1 no. 1) dan mengembalikannya di respons.
> `is_stock_item` / `is_sales_item` / `is_purchase_item` default `0`.
>
> ⚠️ Meski secara teknis opsional, **selalu kirim `item_name`**: karena `item_code` sekarang berupa
> angka seri, item tanpa `item_name` akan tampil sebagai `2609000001` di semua dropdown & transaksi.

**Contoh request (lengkap — produk stok yang dibeli & dijual):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_name": "Air Mineral 600ml",
      "item_group": "Minuman",
      "is_stock_item": 1,
      "is_sales_item": 1,
      "is_purchase_item": 1,
      "stock_uom": "Pcs",
      "min_sales_qty": 6,
      "max_sales_qty": 24,
      "sales_qty_multiple": 6,
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
      "supplier_items": [
        { "supplier": "PT Distributor Utama", "supplier_part_no": "AQUA-600" }
      ],
      "description": "Air mineral kemasan botol 600ml",
      "hashtags": [
        { "hashtag": "promo" },
        { "hashtag": "best-seller" }
      ],
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
    "name": "2609000001",
    "owner": "Administrator",
    "creation": "2026-09-01 09:20:00.000000",
    "item_code": "2609000001",
    "item_name": "Air Mineral 600ml",
    "item_group": "Minuman",
    "is_stock_item": 1,
    "is_sales_item": 1,
    "is_purchase_item": 1,
    "stock_uom": "Pcs",
    "min_sales_qty": 6,
    "max_sales_qty": 24,
    "sales_qty_multiple": 6,
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
    "supplier_items": [
      { "name": "sup001", "supplier": "PT Distributor Utama", "supplier_part_no": "AQUA-600" }
    ],
    "hashtags": [
      { "name": "hsh001", "hashtag": "promo" },
      { "name": "hsh002", "hashtag": "best-seller" }
    ],
    "valuation_rate": 3500,
    "standard_rate": 5000,
    "item_defaults": [
      { "name": "def001", "company": "PT Maju Jaya", "default_warehouse": "Toko Cikarang - PTMJ" }
    ]
  }
}
```

> `name` = `2609000001` (= `item_code`) → **simpan nilai ini dari respons**; dipakai sebagai `name`
> pada semua operasi berikutnya (dikirim di body) dan untuk menghubungkannya ke dokumen lain
> (stok, harga, transaksi).

> **Catatan kode contoh:** contoh di §4.4–§4.10 memakai kode hasil generate yang sama dengan
> §4.1/§4.2 — `2609000001` untuk Air Mineral (item tunggal), `2609000002` untuk template Kaos Polos,
> dan `2609000003`–`2609000007` untuk variannya. Di implementasi nyata kode selalu diambil dari
> `name` pada respons CREATE, bukan ditentukan frontend (§2.1 no. 1).

> **Catatan `has_batch_no` / `has_expiry_date` / `shelf_life_in_days`:** ketiga field ini opsional
> di sisi backend, tetapi bila `has_expiry_date` dikirim `1`, maka `has_batch_no` **wajib** ikut `1`
> (backend menolak bila tidak). `shelf_life_in_days` (mis. `180`) berfungsi sebagai **preset** — saat
> Batch baru dibuat untuk Item ini, `expiry_date` Batch dihitung otomatis dari tanggal produksi +
> `shelf_life_in_days`. Setelah Item punya riwayat stok, `has_batch_no` tidak bisa diubah (§2.1 no. 3).
> Untuk **penomoran Batch saat penerimaan** (otomatis `BATCH-00001` vs input user) lihat **§2.1 no. 12**,
> dengan contoh lengkap di **§4.3**.

> **Catatan harga:** saat CREATE, ERPNext dapat membuat Item Price selling dari `standard_rate` pada
> Price List default selling (`after_insert → add_price`). Selain itu, baseapp menyinkronkan
> `standard_rate` (**harga jual**) ke harga umum UOM stok pada `Standard Selling`, dan `valuation_rate`
> (**harga modal**) ke harga umum UOM stok pada `Standard Buying`. Perubahan `price_list_rate` pada
> baris umum yang masih berlaku di kedua daftar tersebut memperbarui field Item terkait. Perubahan
> `standard_rate` pada Item yang sudah ada tidak otomatis mengubah Item Price. `Standard Selling` dan
> `Standard Buying` tidak boleh dihapus karena diperlukan sinkronisasi baseapp; nonaktifkan jika tidak
> ingin dipakai. Detail pengelolaan harga ada di **[prd_item_price.md](./prd_item_price.md)**.

**Varian A — Produk jasa (non-stok, tidak dibeli):**

```json
{
  "doctype": "Item",
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
>   (replace-all); baris ini ikut terbaca pada `frappe.client.get` (§4.4).
> - Menyimpan nilai di sini **belum memicu apa pun** di backend (tidak membuat Material Request
>   otomatis) — mekanisme reorder & notifikasi diatur belakangan (PRD stok, §9).

### 4.2 CREATE — Item dengan varian

> ⚠️ **Kode template juga dibuat backend — frontend tidak bisa menentukannya.** Template adalah item
> non-varian, jadi ia **kena naming series `YY.MM.######`** seperti item biasa. `name`/`item_code`
> template harus diambil dari **respons CREATE** (§2.1 no. 1); kode inilah yang dipakai di langkah
> berikutnya (`item` pada `create_variant`, `variant_of` pada insert varian).
>
> **Varian juga dapat kode seri — `item_code` kiriman frontend tetap ditimpa.** Ini berbeda dari
> perilaku bawaan ERPNext (yang membuat `{kode template}-{abbr}`, mis. `2609000004-Large`): app
> `baseapp` menggantinya lewat hook `Item.before_insert` supaya lebar kode — dan karena itu lebar
> **barcode yang dicetak** — selalu sama (10 digit).
>
> | Yang dilakukan frontend | Hasil |
> |---|---|
> | Tidak mengirim `item_code` | kode seri sendiri, mis. `2609000005` |
> | Mengirim `item_code` sendiri (mis. `ABC-123`) | **tetap ditimpa** kode seri |
> | `create_variant` menyarankan `{kode template}-{abbr}` | **diabaikan** — pakai `name` dari respons `insert` |
>
> `item_name` **tidak** ikut berubah: varian tetap bernama `{item_name template}-{abbr}` (mis.
> `Kaos Polos-Large`), dan relasi ke template tetap tersimpan di `variant_of`.

> **Konsep:** Satu **template** (`has_variants=1`, berisi daftar atribut tanpa nilai) + beberapa
> **varian** (`variant_of=<template>`, berisi atribut + nilai konkret). Template & varian **disimpan
> di tabel yang sama** `tabItem`; atribut tiap Item di `tabItem Variant Attribute`; master daftar
> atribut/nilai di `tabItem Attribute` + `tabItem Attribute Value` (§2.2). Hanya **varian** yang bisa
> ditransaksikan stok/jual — template hanya sebagai kerangka.

**Contoh kasus:** Produk "Kaos Polos" punya atribut `Ukuran` (nilai `Small`/`Medium`/`Large`).
Dibutuhkan 3 varian. Misalkan backend mengembalikan **`2609000002`** sebagai kode template, maka
tiap varian mendapat kode seri **berikutnya dari counter yang sama**: `2609000003` (Small),
`2609000004` (Medium), `2609000005` (Large) — **bukan** `2609000002-Small`. Yang tetap enak dibaca
adalah `item_name`-nya: `Kaos Polos-Small`, `Kaos Polos-Medium`, `Kaos Polos-Large`.

> **Penting — `abbr` = `attribute_value`.** Sejak app `baseapp` terpasang, kolom `abbr` pada child
> table `item_attribute_values` **diisi otomatis backend** dan nilainya **selalu sama dengan
> `attribute_value`** (Langkah 1 di bawah). Karena `abbr` yang menyusun `item_name` varian, nama
> varian mengikuti **nilai** atributnya (`Small` → `Kaos Polos-Small`, bukan `Kaos Polos-S`).
> `item_code` varian tidak terpengaruh — tetap kode seri (§2.1 no. 1).

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
        { "attribute_value": "Small" },
        { "attribute_value": "Medium" },
        { "attribute_value": "Large" }
      ]
    }
  }'
```

> **`abbr` tidak perlu dikirim.** app `baseapp` mengisinya otomatis **sama dengan `attribute_value`**
> (hook `Item Attribute.before_validate` → `baseapp.utils.sync_attribute_value_and_abbr`). Di form
> Desk kolom `abbr` juga **di-hide** dan **tidak wajib** (property setter dari `baseapp`), sehingga
> user cukup mengisi Attribute Value. Bila `abbr` tetap dikirim dengan nilai berbeda, nilai kiriman
> **ditimpa** oleh `attribute_value`.
>
> `abbr` tinggal dipakai menyusun `item_name` varian (mis. `Medium` → `Kaos Polos-Medium`);
> `item_code` varian sendiri dibuat backend (§2.1 no. 1). Tersimpan di `tabItem Attribute`
> (+ `tabItem Attribute Value`).

**Langkah 2 — Buat template (`has_variants=1`, `attributes` tanpa nilai):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
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
>
> **Tidak ada `item_code`** di payload — respons CREATE mengembalikan kode template hasil generate
> (mis. `2609000002`). **Simpan nilai ini**; langkah 3–4 di bawah memakainya.

**Langkah 3a — Buat varian via method resmi `create_variant`:**

```bash
curl -X POST https://site-anda.com/api/method/erpnext.controllers.item_variant.create_variant \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "item": "2609000002",
    "args": { "Ukuran": "Medium" },
    "use_template_image": false
  }'
```

**Contoh respons (HTTP 200) — dokumen varian (BELUM tersimpan):**

```json
{
  "message": {
    "doctype": "Item",
    "item_code": "2609000002-Medium",
    "item_name": "Kaos Polos-Medium",
    "variant_of": "2609000002",
    "variant_based_on": "Item Attribute",
    "item_group": "Pakaian",
    "is_stock_item": 1,
    "is_sales_item": 1,
    "is_purchase_item": 1,
    "stock_uom": "Pcs",
    "attributes": [ { "attribute": "Ukuran", "attribute_value": "Medium" } ]
  }
}
```

> `create_variant` mengembalikan dokumen **belum disimpan** dengan `item_code`/`item_name` otomatis
> `{kode template}-{abbr}` dan field `reqd`/field terpilih (Item Variant Settings) tersalin dari
> template.
>
> ⚠️ **`item_code` di atas hanya usulan — akan diganti saat disimpan.** app `baseapp` menimpanya
> dengan kode seri (§2.1 no. 1), jadi **jangan** bersandar pada `item_code` dari respons ini maupun
> mengirimkannya kembali; ambil `name` dari respons **insert**. Sebaliknya `item_name` (`Kaos Polos-Medium`)
> dipertahankan apa adanya.
>
> **Simpan** dokumen hasil `create_variant` dengan `frappe.client.insert` (tambahkan
> `"doctype": "Item"`). Varian Medium ini akan tersimpan sebagai **`2609000004`**:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_name": "Kaos Polos-Medium",
      "variant_of": "2609000002",
      "variant_based_on": "Item Attribute",
      "item_group": "Pakaian",
      "is_stock_item": 1,
      "is_sales_item": 1,
      "is_purchase_item": 1,
      "stock_uom": "Pcs",
      "attributes": [ { "attribute": "Ukuran", "attribute_value": "Medium" } ]
    }
  }'
```

> **Contoh respons (HTTP 200):**
>
> ```json
> { "message": { "name": "2609000004", "item_code": "2609000004",
>   "item_name": "Kaos Polos-Medium", "variant_of": "2609000002",
>   "attributes": [ { "attribute": "Ukuran", "attribute_value": "Medium" } ] } }
> ```
>
> **Simpan `name` = `2609000004`** — inilah identitas varian untuk semua operasi berikutnya.
> Bila `item_code` tetap dikirim walau bertentangan, nilainya **tetap ditimpa** (§2.1 no. 1).

> **Pre-check opsional:** gunakan `erpnext.controllers.item_variant.get_variant` dengan
> `{"template": "2609000002", "args": {"Ukuran": "Medium"}}` — `message` berisi `name` varian yang sudah
> ada (kalau belum ada → `null`/kosong), untuk menghindari `ItemVariantExistsError`.

**Langkah 3b (alternatif) — Buat varian manual (`insert` dengan `variant_of`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_name": "Kaos Polos-Large",
      "item_group": "Pakaian",
      "is_stock_item": 1,
      "is_sales_item": 1,
      "is_purchase_item": 1,
      "stock_uom": "Pcs",
      "variant_of": "2609000002",
      "attributes": [ { "attribute": "Ukuran", "attribute_value": "Large" } ]
    }
  }'
```

> Backend memvalidasi: template harus `has_variants=1`, atribut harus valid untuk template, dan
> kombinasi atribut tidak boleh menghasilkan varian yang sudah ada
> (*"Item variant {x} exists with same attributes"* — `ItemVariantExistsError`).
>
> Karena `item_code` tidak dikirim, varian ini mendapat kode seri **otomatis** (mis. `2609000005`).
> `item_name` tetap `Kaos Polos-Large` karena `abbr` (= `attribute_value`) dipakai menyusunnya —
> tapi **tetap kirim `item_name`**: kalau kosong, `Item.validate()` menyalin `item_code`, sehingga
> nama produk jadi `2609000005` (§2.1 no. 1).

**Langkah 4 — Verifikasi daftar varian dari template:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","variant_of"],
    "filters": [["variant_of","=","2609000002"]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "name": "2609000005", "item_name": "Kaos Polos-Large", "variant_of": "2609000002" },
    { "name": "2609000004", "item_name": "Kaos Polos-Medium", "variant_of": "2609000002" },
    { "name": "2609000003", "item_name": "Kaos Polos-Small", "variant_of": "2609000002" }
  ]
}
```

> **Aturan penting varian:**
> - `stock_uom` varian harus sama dengan template (kecuali `Item Variant Settings → Allow Different UOM`).
> - `item_code` varian **selalu** kode seri buatan backend; `item_name` dibangun dari `abbr`
>   (mis. `Medium` → `Kaos Polos-Medium`) — dan `abbr` itu sendiri **selalu sama dengan
>   `attribute_value`** (Langkah 1).
> - **Mengganti nama template ikut mengganti `item_name` semua variannya** (`Kaos Polos-Medium` →
>   `T-Shirt-Medium`); mengganti `item_code` template **tidak**. Aturan lengkap: §2.1 no. 10.
> - **Mengubah `abbr`/`attribute_value` tidak lagi me-rename varian.** ERPNext aslinya me-rename
>   `item_code` varian kembali ke bentuk `{kode template}-{abbr}` (`rename_variant_item_code`);
>   app `baseapp` mematikan jalur itu lewat override controller `ItemAttribute.on_update`
>   (§2.1 no. 7), sehingga kode seri & baris `Item Barcode` tetap valid.
> - Mengubah `attribute_value` yang **sedang dipakai varian** tetap **ditolak** ERPNext
>   (*"The value X is already assigned to an existing Item Y…"*) kecuali
>   `Item Variant Settings → Allow Rename Attribute Value` diaktifkan.
> - Template **tidak boleh punya stok/transaksi** — semua transaksi memakai varian.
> - `has_variants` pada template yang sudah dipakai varian **tidak boleh di-nonaktifkan** (ada
>   validasi terkait).

### 4.3 Item ber-batch — nomor otomatis vs input user (contoh dual satuan roll/Kg)

**Aturan ringkas** (latar lengkap: §2.1 no. 12):

| Kondisi di dokumen penerimaan | `batch_id` yang dipakai |
|---|---|
| User **tidak** mengisi nomor batch | otomatis, pola bawaan `Stock Settings` → `BATCH-00001`, `BATCH-00002`, … |
| User **mengisi** nomor batch | input user apa adanya (mis. `PNM0107-001`) |

**Konsep dual satuan.** Produk seperti tekstil punya **dua satuan sekaligus**: jumlah **roll** dan
berat **Kg**. ERPNext hanya menyimpan **satu** `stock_uom`, jadi polanya:

| Yang dilihat user | Yang disimpan backend |
|---|---|
| "5 roll" | **5 Batch** (1 roll = 1 Batch) |
| "roll 1 = 10 Kg, roll 4 = 20 Kg" | **qty Batch** = berat roll tersebut (10 / 20 Kg) |
| "total stok 70 Kg" | jumlah `actual_qty` seluruh Batch (`tabBin`) |

- `stock_uom = Kg` → semua angka stok (masuk, keluar, sisa) dalam Kg.
- **Jumlah roll = banyaknya Batch** yang stoknya masih `> 0`; **berat roll = qty Batch** itu sendiri.
- Tidak perlu field berat per roll — berat sudah melekat pada Batch.

> Contoh di seksi ini memakai Item `2609000008` (*Kain Roll Premium*) — kode tetap hasil *generate*
> backend, bukan ditentukan frontend (§2.1 no. 1).

**Field Item yang dipakai:**

| Field | Nilai | Fungsi |
|---|---|---|
| `stock_uom` | `Kg` | satuan stok (berat) — satu-satunya satuan stok |
| `has_batch_no` | `1` | aktifkan pelacakan Batch (= roll) |
| `create_new_batch` | `1` | **wajib** — setiap baris transaksi masuk otomatis membuat 1 Batch |
| `batch_number_series` | *(dikosongkan)* | biarkan kosong agar nomor memakai pola bawaan `BATCH-00001` (§2.1 no. 12) |

**Langkah 1 — CREATE Item (`frappe.client.insert`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_name": "Kain Roll Premium",
      "item_group": "Tekstil",
      "stock_uom": "Kg",
      "is_stock_item": 1,
      "is_sales_item": 1,
      "is_purchase_item": 1,
      "has_batch_no": 1,
      "create_new_batch": 1
    }
  }'
```

> **`batch_number_series` sengaja tidak dikirim** → nomor batch memakai pola bawaan `BATCH-00001`
> dari `Stock Settings` (§2.1 no. 12). Kirim field itu **hanya** bila Item ini butuh pola sendiri.
>
> **Tidak perlu `uoms` tambahan** — `Kg` sudah menjadi `stock_uom`, jadi tidak ada konversi yang harus
> didaftarkan. Bila produk juga dijual per satuan lain (mis. `Meter`), tambahkan di `uoms` (§5).

**Langkah 2 — Aturan 1: user TIDAK mengisi nomor batch (otomatis).**

`create_new_batch = 1` membuat **setiap baris** penerimaan menghasilkan **tepat 1 Batch**, dan
nomornya di-*generate* backend memakai pola `Stock Settings` (`BATCH-00001`, `BATCH-00002`, …).
Menerima 3 roll @10 Kg + 2 roll @20 Kg = **5 baris**, `qty` = berat masing-masing roll — **tanpa**
key `batch_no`:

```json
"items": [
  { "item_code": "2609000008", "qty": 10, "warehouse": "Gudang Pusat - PTMJ" },
  { "item_code": "2609000008", "qty": 10, "warehouse": "Gudang Pusat - PTMJ" },
  { "item_code": "2609000008", "qty": 10, "warehouse": "Gudang Pusat - PTMJ" },
  { "item_code": "2609000008", "qty": 20, "warehouse": "Gudang Pusat - PTMJ" },
  { "item_code": "2609000008", "qty": 20, "warehouse": "Gudang Pusat - PTMJ" }
]
```

Hasil: **5 Batch** (`BATCH-00001` … `BATCH-00005`) dengan qty 10 / 10 / 10 / 20 / 20 Kg →
`tabBin` mencatat **70 Kg**.

> **Terverifikasi di site dev (2026-10-06):** penerimaan tanpa `batch_no` pada Item ber-
> `create_new_batch = 1` menghasilkan Batch `BATCH-00001` dengan `name` = `batch_id` = `BATCH-00001`
> dan qty sesuai baris.
>
> Ilustrasi di atas hanya memuat array `items`. Alur lengkap dokumen penerimaan (Purchase Receipt),
> submit, dan jurnalnya ada di PRD tersendiri (menyusul).

**Langkah 3 — Aturan 2: user mengisi nomor batch (input user).**

**3a. Pastikan Batch-nya sudah ada.** ERPNext **tidak** menerima `batch_no` yang belum terdaftar —
submit akan gagal dengan `LinkValidationError: Could not find Row #1: Batch No: <nomor>`. Jadi buat
dulu lewat dua panggilan:

```bash
# cek — nomornya sudah ada?
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Batch",
    "fields": ["name","batch_id","item"],
    "filters": [["batch_id","=","PNM0107-001"]],
    "limit_page_length": 1
  }'
```

Belum ada → buat (`frappe.client.insert`):

```json
{
  "doc": {
    "doctype": "Batch",
    "batch_id": "PNM0107-001",
    "item": "2609000008"
  }
}
```

> - Cukup kirim `batch_id` + `item` — `name` dokumen otomatis = `batch_id` (§2.1 no. 12).
> - **Sudah ada & `item` = item kita** → tidak perlu apa-apa; Batch lama dipakai (mis. melanjutkan
>   roll yang belum habis).
> - **Sudah ada tapi `item` ≠ item kita** → **jangan dipakai**; backend akan menolak saat submit
>   dengan *"Batch Nos X does not belong to Item Y"*. Minta user mengganti nomornya.
> - Insert diulang untuk nomor yang sama → balasan `DuplicateEntryError`; perlakukan sebagai
>   **sukses** (artinya Batch sudah terbuat dari percobaan sebelumnya).

**3b. Kirim baris penerimaan dengan `batch_no`.**

```json
"items": [
  {
    "item_code": "2609000008",
    "qty": 10,
    "uom": "Kg",
    "warehouse": "Gudang Pusat - PTMJ",
    "use_serial_batch_fields": 1,
    "batch_no": "PNM0107-001"
  }
]
```

> `use_serial_batch_fields = 1` membuat rincian batch dibaca langsung dari field `batch_no` pada baris
> (ERPNext lalu membuat Serial and Batch Bundle saat submit). Field ini **sudah aktif** di
> `Stock Settings` site dev. Alternatifnya kirim `serial_and_batch_bundle` bila field itu dimatikan.

**Aman untuk dicoba ulang (koneksi putus / error saat posting).** Urutan di atas sengaja idempoten:

| Langkah | Kalau diulang |
|---|---|
| `get_list` cek nomor | aman (read-only) |
| `insert` Batch | balasan `DuplicateEntryError` → anggap **sukses**, lanjut |
| Kirim dokumen penerimaan | **belum idempoten** — pastikan dokumen belum pernah masuk sebelum mengirim ulang (cek daftar penerimaan supplier / no. surat jalan), karena ERPNext tidak punya *idempotency key* |

> Batch yang sudah terbuat tetapi penerimaannya batal **tidak masalah**: Batch tanpa stok boleh ada,
> dan saat dicoba ulang nomor yang sama dipakai kembali (aturan "sudah ada → pakai ulang").

**Penjualan (sebagian) roll.** Jual 1 roll utuh (10 Kg) → pilih Batch roll itu, `qty = 10`. Jual
sebagian (mis. 6 Kg dari roll 10 Kg) → Batch yang sama, `qty = 6`; sisa 4 Kg tetap di Batch itu.
Payload barisnya sama seperti 3b (kirim `batch_no`), hanya dokumennya dokumen penjualan — detail ada
di PRD penjualan tersendiri (menyusul).

**Cara user memilih roll:** tampilkan daftar Batch milik Item ini (`batch_id` + sisa qty); Batch
dengan sisa `> 0` = roll yang masih ada.

**Validasi backend yang perlu diketahui frontend:**

| Kondisi | Hasil |
|---|---|
| `batch_no` belum ada di master `Batch` | ❌ `LinkValidationError: Could not find Row #1: Batch No: <nomor>` |
| `batch_no` milik Item lain | ❌ `Batch Nos <nomor> does not belong to Item <item>` |
| Transaksi **keluar** dengan nomor yang belum ada | ❌ `Batch No <nomor> does not exists` — Batch hanya boleh dibuat pada transaksi **masuk** |
| `create_new_batch = 0` & `batch_no` kosong | ❌ `Batch ID is mandatory` |

> Alternatif satu panggilan untuk 3a: `erpnext.stock.doctype.serial_and_batch_bundle.serial_and_batch_bundle.is_serial_batch_no_exists`
> dengan `{item_code, type_of_transaction: "Inward", batch_no}` — persis yang dipakai Desk. Ia
> idempoten (nomor yang sudah ada tidak error), tetapi **responsnya kosong** sehingga frontend tidak
> bisa memverifikasi `item`-nya; kalau nomor itu milik Item lain, kegagalan baru muncul saat submit.

**Batasan penting:**

- **Satu `stock_uom` saja.** "Roll" bukan satuan stok, melainkan **jumlah Batch** — tidak ada field
  "jumlah roll" di `tabItem`.
- **Batch yang stoknya habis tetap ada** (riwayat). Jumlah roll aktif = jumlah Batch dengan sisa
  `> 0`, bukan `frappe.client.get_count` atas seluruh dokumen `Batch`.
- Menambah roll = menambah baris (1 baris = 1 Batch). Untuk **satu baris berisi beberapa roll
  sekaligus**, Batch harus dibuat/diisi manual (tidak dibahas di sini).

### 4.4 READ (satu record) & total count

**Langkah 1 — Ambil detail Item (`frappe.client.get`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "name": "2609000001"
  }'
```

> Respons `message` berisi seluruh field Item (seperti respons CREATE), termasuk child table
> `uoms`, `barcodes`, `supplier_items`, `attributes`, `reorder_levels`, `item_defaults`, `taxes`.
> (Catatan: `uoms` pada respons menampilkan baris yang tersimpan — baris `stock_uom` dengan
> `conversion_factor=1`
> otomatis ditambahkan backend bila belum ada.)

**Contoh potongan respons — pemasok Item:**

```json
{
  "message": {
    "name": "2609000001",
    "supplier_items": [
      { "name": "sup001", "supplier": "PT Distributor Utama", "supplier_part_no": "AQUA-600" }
    ]
  }
}
```

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

> `message` = jumlah record yang cocok → dipakai menghitung total halaman pada lazy loading (§4.5).
> Bila query-nya memakai `or_filters`, **kirim juga key `or_filters` di body** pada request yang sama —
> `frappe.client.get_count` membacanya dari body walau tidak ada di signature (§2.1 no. 9).

### 4.5 READ (daftar) — `frappe.client.get_list`

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
    { "name": "2609000001", "item_name": "Air Mineral 600ml", "item_group": "Minuman",
      "stock_uom": "Pcs", "image": "/files/min-001.jpg", "standard_rate": 5000, "disabled": 0 }
  ]
}
```

> Halaman berikutnya naikkan `limit_start` kelipatan `50`; `limit_page_length=0` = ambil **semua**
> record. Filter umum: `disabled=0` (aktif), `is_sales_item=1`, `is_stock_item=1`, `has_variants=0`
> (non-template), `has_variants=1` (template), `variant_of is not set` (item non-varian — item tunggal
> **dan** template; lihat §4.5.1), `variant_of=<template>` (daftar varian), `item_group=<leaf>`,
> dan **jenis produk**: `is_dynamic_product_bundle=1` (paket dinamis), `is_product_bundle=1`
> (paket statis), `is_manufactured_item=1` (produk manufaktur), `is_fixed_asset=1` (aset) —
> gabungkan dengan `or_filters` (§2.1 no. 9).

#### 4.5.1 Daftar item + seluruh variannya (2 request + merge di frontend)

**Kebutuhan:** menampilkan daftar produk di mana tiap **template** ikut membawa daftar variannya —
mis. "Kaos Polos" beserta variannya. Karena `item_code` sekarang angka seri (`2609000006`), yang
enak ditampilkan ke user adalah **`item_name`**-nya (`Kaos Polos-Biru`, `Kaos Polos-Merah`).

**Kenapa harus 2 request:** `frappe.client.get_list` **tidak bisa** mengembalikan array varian
bersarang. Varian **bukan child table** (§2.2) — varian adalah **record `Item` terpisah** di tabel
yang sama, dihubungkan lewat kolom `variant_of`. Jadi polanya: **request 1** mengambil semua item
non-varian (item tunggal **+** template), **request 2** mengambil semua varian dari template pada
halaman itu, lalu **frontend** menggabungkan keduanya (`group by variant_of`). Ini generalisasi dari
§4.2 Langkah 4 (yang baru menangani satu template).

> **Request 1 — semua item non-varian (item tunggal + template).**
> Kunci filter: **`["variant_of","is","not set"]`** → menangkap `variant_of` yang NULL **maupun** `''`.
> Item tunggal dan template keduanya tidak punya `variant_of`, jadi keduanya ikut terambil. Sertakan
> `has_variants` di `fields` agar frontend bisa membedakan **template** (`has_variants=1`) dari
> **item tunggal** (`has_variants=0`).

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","item_group","stock_uom","image","standard_rate","disabled","has_variants","variant_of"],
    "filters": [["disabled","=",0],["variant_of","is","not set"]],
    "order_by": "name asc",
    "limit_start": 0,
    "limit_page_length": 50
  }'
```

> Filter tambahan disesuaikan konteks UI — mis. `["is_sales_item","=",1]` (katalog jual/POS) atau
> `["is_stock_item","=",1]` (hanya produk stok). Contoh di atas sengaja netral: **semua item aktif
> yang bukan varian**.

**Contoh respons (HTTP 200) — item tunggal + template bercampur:**

```json
{
  "message": [
    { "name": "2609000008", "item_name": "Biaya Instalasi", "item_group": "Jasa",
      "stock_uom": "Nos", "image": "", "standard_rate": 150000, "disabled": 0,
      "has_variants": 0, "variant_of": "" },

    { "name": "2609000002", "item_name": "Kaos Polos", "item_group": "Pakaian",
      "stock_uom": "Pcs", "image": "/files/kaos.jpg", "standard_rate": 50000, "disabled": 0,
      "has_variants": 1, "variant_of": "" },

    { "name": "2609000001", "item_name": "Air Mineral 600ml", "item_group": "Minuman",
      "stock_uom": "Pcs", "image": "/files/min-001.jpg", "standard_rate": 5000, "disabled": 0,
      "has_variants": 0, "variant_of": "" }
  ]
}
```

> **Request 2 — semua varian dari template di halaman tersebut.**
> Kirim **hanya `name` yang `has_variants=1`** (lihat catatan penting no. 1). Pakai
> `limit_page_length: 0` karena jumlah varian per halaman kecil; jika template-nya sangat banyak,
> pecah daftar `in` menjadi beberapa batch (lihat catatan penting no. 3).

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","variant_of","stock_uom","image","standard_rate","disabled"],
    "filters": [["variant_of","in",["2609000002"]],["disabled","=",0]],
    "order_by": "variant_of asc, name asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200) — daftar varian (flat, `variant_of` = template-nya):**

```json
{
  "message": [
    { "name": "2609000006", "item_name": "Kaos Polos-Biru", "variant_of": "2609000002",
      "stock_uom": "Pcs", "image": "/files/kaos-biru.jpg", "standard_rate": 50000, "disabled": 0 },
    { "name": "2609000007", "item_name": "Kaos Polos-Merah", "variant_of": "2609000002",
      "stock_uom": "Pcs", "image": "/files/kaos-merah.jpg", "standard_rate": 50000, "disabled": 0 }
  ]
}
```

> **Opsional — child table (nilai atribut & hashtag) tanpa request ketiga.** Karena `attributes`
> dan `hashtags` **keduanya** child table, request 1 maupun request 2 bisa menambahkan nested child
> query pada `fields` — **boleh lebih dari satu sekaligus**:
>
> ```json
> "fields": ["name","item_name","variant_of","image","standard_rate",
>            { "attributes": ["attribute","attribute_value"] },
>            { "hashtags":   ["hashtag"] }]
> ```
>
> → tiap baris ikut membawa `"attributes": [ ... ]` **dan** `"hashtags": [ ... ]`.
> **Terverifikasi di site dev (Frappe v16, 2026-10-08)** — 2 nested child query dalam 1 request:
>
> ```json
> { "name": "2610000034", "item_name": "ZZT HT Template",
>   "attributes": [ { "attribute": "ZZT HT Color", "attribute_value": null } ],
>   "hashtags":   [ { "hashtag": "promo" } ] }
> { "name": "2610000035", "item_name": "ZZT HT Template-Red",
>   "attributes": [ { "attribute": "ZZT HT Color", "attribute_value": "Red" } ],
>   "hashtags":   [ { "hashtag": "varian-merah" } ] }
> ```
>
> Karena itu, **satu halaman yang sekaligus menampilkan template + semua varian + hashtag tiap
> varian tetap cukup 2 request** (§4.12). Contoh respons nyata satu varian (uji 2026-09-26):
>
> ```json
> { "name": "BUB-PB-BLU-BIG", "item_name": "Produk Bervarian - Blue - Big", "variant_of": "BUB-PB",
>   "attributes": [ { "attribute": "Color", "attribute_value": "Blue" },
>                   { "attribute": "Size",  "attribute_value": "Big" } ] }
> ```
>
> Catatan: (a) pada baris **template**, `attribute_value` bernilai `null` (§2.5 — template hanya
> mendaftar atribut); (b) sintaks `{ "<child_table>": [ ... ] }` hanya berlaku untuk **child table
> asli** — di Item: `attributes`, `uoms`, `barcodes`, `reorder_levels`, `item_defaults`, `taxes`,
> **`hashtags`**; **tidak** untuk varian (varian bukan child table); (c) child table yang belum
> pernah diisi tidak muncul di respons (mis. `hashtags` tidak ada key-nya); (d) alternatif lain:
> `fields` string bertitik (`"attributes.attribute_value"`) juga jalan, tetapi hasilnya **flat** —
> satu baris per baris child sehingga item bisa muncul berulang (perilaku sama seperti
> `hashtags.hashtag`, §2.8); (e) memperbanyak child query **tidak** menggandakan baris Item selama
> dipakai sintaks `{ ... }` — pada uji 2026-09-26 varian dengan **2 baris** `attributes` tetap keluar
> sebagai **1 baris** respons.

> **Merge di frontend** — gabungkan respons request 2 ke respons request 1 dengan `group by variant_of`.

```js
// 1) Request 1
const items = (await api1({ limit_start: 0, limit_page_length: 50 })).message;

// 2) Ambil nama template di halaman ini saja
const templateNames = items.filter(i => i.has_variants == 1).map(i => i.name);

// 3) Request 2 — WAJIB di-skip kalau daftar template kosong (lihat catatan penting no. 1)
const variantRows = templateNames.length ? (await api2(templateNames)).message : [];

// 4) Group by variant_of
const byTemplate = new Map();
for (const v of variantRows) {
  if (!byTemplate.has(v.variant_of)) byTemplate.set(v.variant_of, []);
  byTemplate.get(v.variant_of).push(v);
}

// 5) Tempelkan ke baris request 1 (nama key hasil gabungan bebas — ditentukan UI, mis. "variants")
const result = items.map(i => ({
  ...i,
  variants: i.has_variants == 1 ? (byTemplate.get(i.name) || []) : null,
}));
```

**Contoh hasil gabungan yang dipakai UI:**

```json
[
  { "name": "2609000008", "item_name": "Biaya Instalasi", "has_variants": 0, "variants": null },
  {
    "name": "2609000002", "item_name": "Kaos Polos", "item_group": "Pakaian",
    "stock_uom": "Pcs", "image": "/files/kaos.jpg", "standard_rate": 50000,
    "has_variants": 1,
    "variants": [
      { "name": "2609000006", "item_name": "Kaos Polos-Biru", "variant_of": "2609000002",
        "image": "/files/kaos-biru.jpg", "standard_rate": 50000 },
      { "name": "2609000007", "item_name": "Kaos Polos-Merah", "variant_of": "2609000002",
        "image": "/files/kaos-merah.jpg", "standard_rate": 50000 }
    ]
  },
  { "name": "2609000001", "item_name": "Air Mineral 600ml", "has_variants": 0, "variants": null }
]
```

**Kontrak untuk UI:**

| Kondisi | Arti |
|---|---|
| `variants: null` | Item tunggal (`has_variants=0`) — memang tidak punya varian. |
| `variants: []` | Template (`has_variants=1`) yang **belum punya varian** atau semua variannya nonaktif. |
| `variants: [ ... ]` | Daftar varian template tersebut (objek `Item` utuh, minimal `name` + `item_name`). |

**Catatan penting:**

1. **Jangan pernah mengirim `["variant_of","in",[]]` (list kosong).** Di Frappe, list kosong pada
   operator `in` **tidak** berarti "tidak ada hasil", melainkan diubah menjadi `IN ('')` →
   `variant_of IN ('') OR variant_of IS NULL`, sehingga request 2 akan **mengembalikan item
   non-varian** (item tunggal + template) seolah-olah varian. Selalu **lewati request 2** bila daftar
   template kosong. (Terverifikasi di site dev 2026-09-26: `["variant_of","in",[]]` mengembalikan
   item dengan `variant_of: null` — bukan hasil kosong.)
2. **`variant_of is not set` menangkap NULL dan `''`.** Template juga tidak punya `variant_of`, jadi
   ikut terambil di request 1 — ini yang diinginkan. (Padanan `has_variants=0` **bukan** penggantinya:
   filter itu **mengecualikan** template.)
3. **Batasi request 2 per halaman, jangan seluruh dataset.** Kalau request 1 tidak dipaginasi
   (`limit_page_length=0`) dan template-nya banyak, pecah `in` menjadi batch ±100–200 nama per request
   agar query & payload tetap wajar.
4. **Urutan stabil.** Request 1 pakai `order_by: "name asc"` (agar pagination tidak melompat/duplikat),
   request 2 pakai `order_by: "variant_of asc, name asc"` agar pengelompokan deterministik.
5. **Varian nonaktif.** Dengan `["disabled","=",0]` di request 2, template yang seluruh variannya
   dinonaktifkan akan tampil dengan `variants: []`. Sesuai §4.9, menonaktifkan template juga
   menonaktifkan seluruh variannya, sehingga filter ini konsisten.
6. **Total halaman (pagination).** Pakai `frappe.client.get_count` (§4.4) dengan `filters` yang
   **identik** dengan request 1.
7. **Alternatif tanpa merge di frontend:** bila UI tidak ingin menggabungkan sendiri, pola ini perlu
   dijadikan satu endpoint custom (whitelisted, mis. di app `baseapp`) yang mengembalikan template +
   array varian dalam satu respons — di luar cakupan dokumen ini karena bukan `frappe.client.*`.

### 4.6 UPDATE — `frappe.client.save` & `frappe.client.set_value`

Update memakai `save`: kirim dokumen (hasil `frappe.client.get` yang dimodifikasi); `name` ada di
body. Child table (`uoms`, `barcodes`, `supplier_items`, `attributes`, `reorder_levels`,
`item_defaults`, `taxes`) berlaku **replace-all** — kirim seluruh baris yang diinginkan.

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "name": "2609000001",
      "item_code": "2609000001",
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
      "supplier_items": [
        { "supplier": "PT Distributor Utama", "supplier_part_no": "AQUA-600" }
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
> - `supplier_items` dikelola dengan cara yang sama: pertahankan baris yang masih berlaku dan kirim
>   seluruh daftar pemasok yang diinginkan. Untuk menghapus satu pemasok dari Item, hilangkan baris
>   tersebut dari daftar sebelum `save`; untuk menghapus semua pemasok, kirim `"supplier_items": []`.
> - Ubah `item_name` pada **template** (`has_variants=1`) → **semua varian ikut berganti nama**
>   (`KAOS-M` → `T-SHIRT-M`), baik lewat `save` maupun `set_value`. Bila salah satu nama varian
>   bentrok dengan Item aktif lain, seluruh permintaan ditolak 417 dan tidak ada yang berubah
>   (§2.1 no. 10).
> - Ubah `standard_rate` pada Item yang sudah ada **tidak** otomatis mengubah Item Price. Perubahan
>   harga yang akan disinkronkan ke `standard_rate` (harga jual) atau `valuation_rate` (harga modal)
>   dilakukan melalui baris Item Price umum UOM stok pada `Standard Selling` atau `Standard Buying`.
>   Pengelolaan harga:
>   [prd_item_price.md](./prd_item_price.md).
> - Ubah `item_group` → pindah kategori produk (tidak ada efek samping stok).
> - Ubah `stock_uom` pada Item ber-stok **ditolak backend**.

**DELETE pemasok dari Item — `frappe.client.save`:**

Tidak ada delete mandiri untuk baris child `Item Supplier`. Ambil dokumen Item dengan GET, hapus
dari `doc.supplier_items` baris pemasok yang ingin dilepas, lalu kirim kembali **dokumen lengkap**
tersebut ke `frappe.client.save`. Contoh, bila hasil GET berisi dua pemasok dan
`PT Distributor Utama` akan dihapus, pertahankan semua field Item dari respons GET dan sisakan hanya
pemasok yang tidak dihapus. Berikut **potongan** field `doc` yang berubah:

```json
{
  "doctype": "Item",
  "name": "2609000001",
  "item_code": "2609000001",
  "supplier_items": [
    { "supplier": "PT Pemasok Kedua", "supplier_part_no": "SKU-002" }
  ]
}
```

Kirim objek tersebut sebagai `"doc"` ke `frappe.client.save`, bersama field lain dari dokumen hasil
GET. Untuk melepas **semua** pemasok, set `"supplier_items": []`. Baris lain yang tidak ingin diubah
harus tetap disertakan karena seluruh child table diganti sesuai payload.

**Perubahan kecil — `frappe.client.set_value`** (lebih aman untuk satu-dua field):

```bash
# Non-aktifkan item
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "name": "2609000001",
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
    "name": "2609000001",
    "fieldname": { "item_group": "Air Mineral", "standard_rate": 6000 }
  }'
```

### 4.7 Barcode — CRUD

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
      "item_name": "Air Mineral 600ml",
      "item_group": "Minuman",
      "stock_uom": "Pcs",
      "barcodes": [
        { "barcode": "8991234567891", "barcode_type": "EAN-13" },
        { "barcode": "8991234567808", "barcode_type": "EAN-13", "uom": "Dus" }
      ]
    }
  }'
```

> **Tanpa `item_code`** — kode dibuat backend (§2.1 no. 1); `barcodes` boleh ikut di payload CREATE yang sama.
>
> **`barcodes` bersifat opsional.** Bila tidak dikirim (atau dikirim `[]`), backend otomatis
> menambahkan satu baris `barcode = item_code` dengan `uom = stock_uom` dan `barcode_type` kosong
> (§2.1 no. 7). Kirim `barcodes` hanya bila produk memang punya barcode pabrik sendiri.
>
> ⚠️ **`barcode_type` yang diisi memicu uji check digit.** `8991234567890` **ditolak** dengan
> `InvalidBarcode` — check digit yang benar untuk `899123456789` adalah `1`, jadi nilai validnya
> `8991234567891`. Contoh di atas sudah memakai nilai yang valid. Karena `item_code` (10 digit) tidak
> mungkin lolos uji EAN/UPC, baris barcode default selalu memakai `barcode_type` kosong.

**READ — `barcodes` ikut dalam respons `frappe.client.get`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Item", "name": "2609000001" }'
```

**Contoh respons (bagian `barcodes`):**

```json
{
  "message": {
    "name": "2609000001",
    "barcodes": [
      { "name": "xyz001", "barcode": "8991234567891", "barcode_type": "EAN-13", "uom": "" },
      { "name": "xyz002", "barcode": "8991234567808", "barcode_type": "EAN-13", "uom": "Dus" }
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
      "name": "2609000001",
      "item_code": "2609000001",
      "item_group": "Minuman",
      "stock_uom": "Pcs",
      "barcodes": [
        { "barcode": "8991234567891", "barcode_type": "EAN-13" },
        { "barcode": "8991234567815", "barcode_type": "EAN-13", "uom": "Pcs" }
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
>     "filters": [["barcodes.barcode","=","8991234567891"]],
>     "limit_page_length": 1
>   }'
> ```
> `barcodes.barcode` adalah **child filter** — didukung `frappe.client.get_list` (join child table).

### 4.8 Foto produk — CRUD (`image` + multi foto di `tabFile`)

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
    "docname": "2609000001",
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

Contoh kasus: 5 foto di-upload ke template `2609000002` → **3 foto** untuk varian `2609000004`,
**2 foto** untuk varian `2609000005`.

**Langkah 1 — Upload 5 foto ke template (`frappe.client.attach_file`, ulangi per foto):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.attach_file \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "docname": "2609000002",
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
      "attached_to_name": "2609000004",
      "is_private": 0
    }
  }'
```

> Lakukan hal yang sama untuk varian 2 (`attached_to_name: "2609000005"`) dengan `file_url` 2 foto
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
      ["attached_to_name","=","2609000004"]
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
      ["attached_to_name","=","2609000001"]
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

### 4.9 Non-aktif / hapus

**Aturan bisnis (wajib diikuti frontend):**

1. **Non-aktif = soft-delete.** Set `disabled=1` via `frappe.client.set_value` (contoh di §4.6).
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
    "filters": [["variant_of","=","2609000002"]],
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
      ["item_code","in",["2609000002","2609000003","2609000004","2609000005"]],
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
    { "item_code": "2609000004", "warehouse": "Toko Cikarang - PTMJ", "actual_qty": 12, "stock_uom": "Pcs" },
    { "item_code": "2609000005", "warehouse": "Gudang Pusat - PTMJ", "actual_qty": 30, "stock_uom": "Pcs" }
  ]
}
```

> - Respons berupa **baris per (item, warehouse)** dengan `actual_qty > 0`. Frontend mengagregasi
>   total per `item_code` untuk dialog konfirmasi (mis. `2609000004` = 12 Pcs, `2609000005` = 30 Pcs).
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
        { "item_code": "2609000004", "warehouse": "Toko Cikarang - PTMJ", "qty": 0 },
        { "item_code": "2609000005", "warehouse": "Gudang Pusat - PTMJ", "qty": 0 }
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
    "name": "2609000004",
    "fieldname": { "disabled": 1 }
  }'
```

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "name": "2609000002",
    "fieldname": { "disabled": 1 }
  }'
```

**Hapus permanen — `frappe.client.delete`** (hanya bila benar-benar diperlukan):
- `name` di body; **diblokir** bila Item sudah dipakai transaksi/stok (`LinkExistsError` /
  `StockExistsForTemplateError`), atau masih punya child table yang di-referensikan.
- Karena data master "jangan hapus" + item bisa punya riwayat, **jangan hapus Item yang sudah
  dipakai** — cukup non-aktifkan.
- Khusus **template** (`has_variants=1`): tidak bisa dihapus bila masih punya varian.

### 4.10 tabBin — info stok per (Item, Warehouse)

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
    "filters": [["item_code","=","2609000001"]],
    "order_by": "warehouse asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "item_code": "2609000001", "warehouse": "Gudang Pusat - PTMJ", "actual_qty": 120,
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

### 4.11 Pre-check `item_name` — `baseapp.api.check_item_name`

> **Endpoint custom dari app `baseapp`** (bukan `frappe.client.*`). Panggil **sebelum** CREATE
> (§4.1) dan sebelum mengubah nama lewat UPDATE, karena backend hanya menolak duplikat yang masih
> **aktif** — keputusan untuk kasus item non-aktif ada di tangan user (§2.1 no. 8).

```bash
curl -X POST https://site-anda.com/api/method/baseapp.api.check_item_name \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "item_name": "Air Mineral 600ml",
    "exclude": "2609000001"
  }'
```

> `exclude` = `name` Item yang sedang diedit, supaya tidak cocok dengan dirinya sendiri.
> **Omit saat CREATE.**

**Contoh respons (HTTP 200):**

```json
{
  "message": {
    "item_name": "Air Mineral 600ml",
    "normalized": "air mineral 600ml",
    "status": "duplicate_inactive",
    "active": [],
    "inactive": [ { "name": "2609000005", "item_name": "Air Mineral 600ml", "disabled": 1 } ]
  }
}
```

**Arti `status` & tindakan frontend:**

| `status` | Arti | Tindakan |
|---|---|---|
| `ok` | Nama bebas | Lanjut CREATE/UPDATE |
| `duplicate_active` | Ada Item **aktif** dengan nama itu (`active` tidak kosong) | Tampilkan alert — *"Nama X sudah dipakai Item Y"* — dan minta user mengganti nama. Kalau tetap dikirim, backend menolak 417 (§7) |
| `duplicate_inactive` | Hanya Item **non-aktif** yang memakainya (`inactive` tidak kosong) | Tanya user: *"Item Y (non-aktif) sudah memakai nama ini. Reaktifkan item lama, atau buat item baru?"* — lihat di bawah |

> Kalau keduanya ada, `status` = `duplicate_active` (yang memblokir menang), tapi `inactive` tetap
> diisi sehingga frontend masih bisa menawarkan reaktifasi.

**Alur untuk `duplicate_inactive`:**

1. **User memilih "Reaktifkan item lama"** → **jangan** CREATE. Cukup aktifkan kembali Item lama:

   ```bash
   curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
     -H 'Authorization: Bearer <access_token>' \
     -H 'Content-Type: application/json' \
     -d '{ "doctype": "Item", "name": "2609000005", "fieldname": { "disabled": 0 } }'
   ```

   Selanjutnya pakai `name` Item lama tersebut sebagai identitas produk.

2. **User memilih "Buat item baru"** → lanjut CREATE seperti biasa (§4.1). Backend **mengizinkan**
   karena duplikatnya non-aktif, sehingga akan ada dua Item bernama sama (satu non-aktif, satu aktif).

> **Catatan:** `normalized` di respons hanya untuk debug — frontend **tidak perlu** menghitungnya
> sendiri. Perbandingan dijamin sama dengan yang dipakai validator backend (§2.1 no. 8).

---

### 4.12 Hashtag — CRUD & pola satu halaman (template + semua varian)

Bentuk field & aturannya di **§2.8**. Ringkas: `hashtags` adalah **child table** (satu baris = satu
hashtag) yang tiap barisnya **Link ke master `Hashtag`**, nilainya dirapikan backend (tanpa `#`,
huruf kecil), bersifat **replace-all** saat simpan, dan **tidak diwariskan** dari template ke varian.
Hashtag yang belum terdaftar di master **ditolak** (§2.8, §4.12f).

#### a. Tulis hashtag saat CREATE

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item",
      "item_name": "Air Mineral 600ml",
      "item_group": "Minuman",
      "stock_uom": "Pcs",
      "hashtags": [
        { "hashtag": "promo" },
        { "hashtag": "best-seller" },
        { "hashtag": "minuman" }
      ]
    }
  }'
```

> Frontend **boleh** mengirim apa adanya seperti yang diketik user (`"#Promo"`, `" Promo "`) —
> backend merapikannya. Yang **tidak boleh**: spasi di tengah, `!`, `#` di tengah, karakter selain
> `a-z 0-9 - _`, atau lebih dari 50 karakter → ditolak `417` (§7).

#### b. Baca hashtag satu Item

Sudah ikut di respons `frappe.client.get` (§4.4) sebagai key `hashtags`:

```json
{
  "message": {
    "name": "2609000001",
    "item_name": "Air Mineral 600ml",
    "hashtags": [
      { "name": "hsh001", "hashtag": "promo" },
      { "name": "hsh002", "hashtag": "best-seller" },
      { "name": "hsh003", "hashtag": "minuman" }
    ]
  }
}
```

> Urutan baris mengikuti `idx` = **urutan baris yang dikirim saat menyimpan**, jadi frontend boleh
> mengandalkannya untuk menampilkan urutan hashtag (tidak perlu sort sendiri). **Terverifikasi di
> site dev (2026-10-08)**: kirim `promo, baru, minuman` → respons `insert`, `frappe.client.get`, dan
> nested `{ "hashtags": [...] }` pada `get_list` semuanya mengembalikan urutan yang sama.

#### c. Ganti hashtag satu Item (UPDATE)

`hashtags` **replace-all** — kirim seluruh daftar yang ingin dipertahankan (§2.8):

```bash
# pertahankan hanya "promo", hapus sisanya
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "name": "2609000001",
      "doctype": "Item",
      "hashtags": [ { "hashtag": "promo" } ]
    }
  }'

# hapus semua hashtag Item tsb:
#   "hashtags": []
```

> **Jangan** pakai `frappe.client.set_value` untuk hashtag — method itu hanya menyentuh field biasa,
> bukan child table (§4.6). Untuk sekadar **menambah 1 hashtag**, ambil dulu daftar lama (§4.4),
> tambahkan di frontend, lalu kirim **seluruh** daftar.

#### d. Pola satu halaman: template + semua varian

Skenario: form produk menampilkan **template** beserta **seluruh variannya** dalam satu halaman.
Hashtag bersifat **per Item** — tiap varian punya daftar sendiri dan **tidak ikut** template (§2.8).

**Baca — cukup 2 request:**

1. **Request 1** — template (1 dokumen penuh, §4.4): `frappe.client.get`
   `{ "doctype": "Item", "name": "2609000002" }` → `message.hashtags` = hashtag template.
2. **Request 2** — semua varian **beserta atribut & hashtag-nya, satu request** (§4.5.1):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","variant_of","stock_uom","image",
               { "attributes": ["attribute","attribute_value"] },
               { "hashtags":   ["hashtag"] }],
    "filters": [["variant_of","=","2609000002"]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200) — tiap varian membawa hashtagnya sendiri:**

```json
{
  "message": [
    { "name": "2609000006", "item_name": "Kaos Polos-Biru", "variant_of": "2609000002",
      "attributes": [ { "attribute": "Colour", "attribute_value": "Biru" } ],
      "hashtags":   [ { "hashtag": "promo" }, { "hashtag": "kaos" } ] },
    { "name": "2609000007", "item_name": "Kaos Polos-Merah", "variant_of": "2609000002",
      "attributes": [ { "attribute": "Colour", "attribute_value": "Merah" } ],
      "hashtags":   [ { "hashtag": "kaos" } ] }
  ]
}
```

> Varian yang belum pernah diisi hashtag **tidak punya key `hashtags`** di respons — perlakukan
> sebagai daftar kosong. Baris **template** juga tidak punya key itu bila belum diisi.
> **Terverifikasi (2026-10-08):** satu `get_list` dengan **dua** nested child query (`attributes`
> + `hashtags`) mengembalikan keduanya terisi, tanpa menggandakan baris Item.

**Tulis — N request, satu per dokumen.** Child table tidak bisa disimpan lewat `set_value`, dan
`save` hanya memproses **satu** dokumen. Jadi halaman yang mengedit template + 2 varian mengirim
**3** `frappe.client.save`:

```bash
# 1 dokumen = 1 request; cukup kirim dokumen yang berubah
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "name": "2609000006",
      "doctype": "Item",
      "hashtags": [ { "hashtag": "promo" }, { "hashtag": "kaos" } ]
    }
  }'
# ulangi untuk 2609000007 (varian lain) dan 2609000002 (template)
```

> **Pewarisan template → varian ditangani frontend.** Backend **tidak** menyalin hashtag template ke
> varian (§2.8). Bila user ingin varian mewarisi, frontend cukup memanggil `save` untuk **tiap
> varian** dengan daftar yang diinginkan (mis. hashtag template + tambahan khusus varian) — kirim
> seluruh daftar, bukan hanya tambahannya.
>
> Urutan yang disarankan: **template dulu, baru varian**. Tidak ada transaksi lintas-dokumen di
> level API — bila salah satu request gagal (`417`), tampilkan error beserta nama Item-nya, dan
> biarkan user memperbaiki lalu simpan ulang (dokumen lain yang sudah berhasil tetap tersimpan).

#### e. Menyaring daftar Item per hashtag

```bash
# hanya Item bertag "promo" — "promo2026" TIDAK ikut (inti fiturnya)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","item_group","image"],
    "filters": [["hashtags.hashtag","=","promo"],["disabled","=",0]],
    "order_by": "name asc",
    "limit_start": 0,
    "limit_page_length": 50
  }'
```

```bash
# jumlahnya (untuk pagination / lazy loading)
#   POST frappe.client.get_count
#   { "doctype":"Item", "filters":[["hashtags.hashtag","=","promo"],["disabled","=",0]] }
```

**Hasil uji di site dev (2026-10-08)** — 3 Item uji: A = `promo`, B = `promo2026`, C = `promo` + `baru`:

| Filter | Hasil | Catatan |
|---|---|---|
| `[["hashtags.hashtag","like","%promo%"]]` | A, B, C | **salah untuk kebutuhan ini** — `promo2026` ikut terbawa |
| `[["hashtags.hashtag","=","promo"]]` | A, C | **benar** — inilah yang dipakai |
| `[["hashtags.hashtag","in",["promo","baru"]]]` | A, C — **C muncul 2×** | dedupe di frontend, atau pakai `group_by` |
| `[["hashtags.hashtag","=","promo"],["hashtags.hashtag","=","baru"]]` | **kosong** | 2 kondisi pada fieldname yang sama **bukan** AND (§2.8) |
| `frappe.client.get_count` + filter `=` | `2` | akurat |

> **Mencari Item yang punya DUA tag sekaligus** (mis. `promo` **dan** `baru`): jalankan 2 request
> (masing-masing 1 filter `=`, `fields: ["name"]`), lalu iris (`intersect`) daftar `name`-nya di
> frontend. Ini konsekuensi dari child table yang di-JOIN (§2.8).
>
> **Daftar hashtag yang sudah pernah dipakai** (untuk autocomplete / saran filter): **§6.8**.

#### f. Master `Hashtag` — CRUD (daftar pilihan hashtag)

Sejak `Item Hashtag.hashtag` menjadi **Link ke master `Hashtag`**, nilai yang boleh dipakai tidak lagi
"apa pun yang pernah ditulis" — melainkan **isi master**. Frontend mengambil dropdown dari sini, dan
user menambah/mengubah/menghapus hashtag lewat CRUD di bawah.

```bash
# daftar (dropdown) — name = hashtag itu sendiri
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Hashtag",
    "fields": ["name"],
    "order_by": "name asc",
    "limit_page_length": 0
  }'

# tambah hashtag baru
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "doc": { "doctype": "Hashtag", "hashtag": "promohariini" } }'

# ubah nama hashtag — ikut memperbaiki semua Item / DPB Hashtag yang menunjuknya
curl -X POST https://site-anda.com/api/method/frappe.client.rename_doc \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Hashtag", "old_name": "promo", "new_name": "promo-baru" }'

# hapus hashtag — DIBLOKIR selama masih dirujuk Item / Dynamic Product Bundle Hashtag
curl -X POST https://site-anda.com/api/method/frappe.client.delete \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{ "doctype": "Hashtag", "name": "promo-baru" }'
```

- **`name` = hashtag itu sendiri** (`autoname: field:hashtag`), jadi nilai yang dipakai di baris
  `hashtags` = nama dokumen master — pola sama seperti `Brand`.
- Nilai wajib huruf kecil, tanpa `#`, cocok pola `^[a-z0-9_-]{1,50}$`. Normalisasi otomatis hanya
  berlaku pada input yang dikirim lewat **dokumen Item** (§2.8); membuat master baru berarti
  mengirim nilainya sudah bersih.
- `Hashtag` adalah doctype **normal** (bukan child) — jadi `get_list`, `get_count`, `rename_doc`,
  `Frappe` desk, dan permission-nya berjalan seperti master lain.
- Role: **`Item Manager`** dan **`System Manager`** (selaras dengan doctype Dynamic Product Bundle —
  [prd_baseapp.md §4.5](../baseapp/prd_baseapp.md)).

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
      "item_name": "Gula Pasir 1kg",
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

### 6.8 Daftar hashtag untuk dropdown — master `Hashtag`

Sejak `Item Hashtag.hashtag` menjadi **Link ke master `Hashtag`** (§2.8), sumber daftar hashtag adalah
**master-nya**, bukan lagi kumpulan nilai yang kebetulan dipakai Item. Cukup satu `get_list` biasa —
`Hashtag` adalah doctype normal yang punya permission sendiri (tidak seperti child doctype):

```bash
# dropdown semua hashtag
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Hashtag",
    "fields": ["name"],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{ "message": [ { "name": "baru" }, { "name": "minuman" }, { "name": "promo" } ] }
```

```bash
# autocomplete saat user mengetik "prom"
#   "filters": [["name","like","%prom%"]]

# jumlah hashtag (pagination) — doctype normal, jadi get_count akurat
#   { "doctype": "Hashtag", "filters": [] }
```

> **Tidak perlu lagi** trik lama `fields: ["hashtags.hashtag"]` + `group_by: "hashtags.hashtag"`
> pada doctype `Item`. Trik itu dipakai waktu hashtag masih berupa teks bebas di child table; sekarang
> daftar resminya ada di master, sehingga satu query sederhana sudah cukup.
>
> ⛔ **Jangan** query langsung ke child doctype `Item Hashtag` (`frappe.client.get_list` mengembalikan
> `name` saja karena child doctype tidak punya definisi permission) — tetap berlaku.

---

## 7. Penanganan error umum

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | Role tidak punya akses (bukan `Item Manager` utk tulis) | `{"message": "Not permitted"}` |
| 404 | Resource tidak ditemukan | `{"exc_type":"DoesNotExistError","message":"Resource Not Found"}` |
| 417 | Field wajib kosong (`reqd`) | `{"exc_type":"MandatoryError","message":"item_group is mandatory"}` / `"stock_uom is mandatory"` |
| 417 | Duplikat `item_code` (`unique`) | **Tidak lagi relevan** — kode dibuat backend dari counter naming series, jadi tidak mungkin duplikat (§2.1 no. 1). |
| 417 | `item_name` sudah dipakai Item **aktif** | `{"exc_type":"ValidationError","message":"Item Name Air Mineral 600ml is already used by active Item(s) 2609000001."}` — pre-check dulu (§4.11). Duplikat dengan Item **non-aktif** **tidak** diblokir (§2.1 no. 8) |
| 417 | Ganti nama **template** yang membuat nama varian bentrok | `{"exc_type":"ValidationError","message":"Cannot rename variant 2609000004 to T-SHIRT-M: that Item Name is already used by active Item(s) 2609000009."}` — varian yang bentrok disebutkan di pesan; **rename template dibatalkan**, tidak ada yang berubah (§2.1 no. 10) |
| 417 | Barcode duplikat global | `{"exc_type":"ValidationError","message":"Barcode 2609000003 already used in Item 2609000003"}` — keunikan dicek lintas **semua** Item (§2.4) |
| 417 | `barcode_type` diisi tapi nilainya bukan 8/12/13 digit dengan check digit valid | `{"exc_type":"InvalidBarcode","message":"Barcode 8991234567890 is not a valid EAN-13 code"}` — **seluruh insert Item gagal**; untuk kode `item_code` kosongkan `barcode_type` (§2.1 no. 7) |
| 417 | Varian sudah ada dgn kombinasi atribut sama | `{"exc_type":"ItemVariantExistsError","message":"Item variant 2609000004 exists with same attributes"}` — pesan memakai `name` varian yang sudah ada |
| 417 | Nilai atribut tidak valid utk template | `{"exc_type":"InvalidItemAttributeValueError","message":"..."}` |
| 417 | Template tidak boleh punya stok / `has_variants` salah | `{"exc_type":"ValidationError","message":"..."}` |
| 417 | Ubah `stock_uom` pada item ber-stok | `{"exc_type":"ValidationError","message":"..."}` |
| 417 | `conversion_factor` stock_uom ≠ 1 / `uoms` duplikat | `{"exc_type":"ValidationError","message":"..."}` (lihat §5.4) |
| 417 | Hapus Item yang sudah dipakai transaksi/stok | `{"exc_type":"LinkExistsError","message":"Cannot delete because of linked records"}` |
| 417 | `attributes` kosong saat `has_variants=1` | `{"exc_type":"ValidationError","message":"Attribute table is mandatory"}` |
| 417 | Hashtag tidak sesuai pola | `{"exc_type":"ValidationError","message":"Row #1: hashtag promo 2026 is not valid. Use lowercase letters, digits, - and _ only (max 50 characters)."}` — nomor baris mengikuti urutan baris pada payload; yang boleh hanya `a-z 0-9 - _` (maks 50 karakter), tanpa `#` dan tanpa spasi (§2.8) |

> **Catatan `item_code` (naming series):** karena `item_code` dibuat backend (§2.1 no. 1), error
> **`"Item Code is required"`** (`ValidationError` dari `field:item_code` saat kode kosong) **tidak
> akan muncul lagi** di konfigurasi ini — dulu bisa terjadi karena kode wajib dikirim frontend.
> `MandatoryError` yang masih relevan hanya `item_group` dan `stock_uom`.

> **Catatan:** karena seluruh pemanggilan memakai `/api/method/...`, hasil sukses dibungkus di
> `message` (bukan `data`). Body error tetap berbentuk `{ "exc_type": ..., "exception": ..., "message": ... }`.
> Method `frappe.client.get` dengan `name` yang tidak ada → 404 `DoesNotExistError`.

---

## 8. Koleksi Postman

Spesifikasi di atas **direncanakan** masuk ke satu koleksi Postman yang siap import:
`docs/postman/postman_erpnext_api.json` (Collection v2.1). Koleksi itu saat ini berisi seluruh modul
API ERPNext yang sudah jadi (OAuth 2.0 + Supplier + Customer + Contact + Address + Lead + Employee
+ User + Warehouse + POS Profile + Item Group) — **folder `11. Item` belum dibuat**.

> **Status: folder `11. Item` belum ada di koleksi.** Koleksi saat ini melompat dari `10. Item Group`
> ke `12. Stock Reconciliation`; daftar di bawah adalah **rencana** isinya, sudah disesuaikan dengan
> naming series (§2.1 no. 1).

Folder **`11. Item`** direncanakan berisi **±52 request** yang mencakup:
- Pre-check nama Item — `baseapp.api.check_item_name` (§4.11): kasus `ok` / `duplicate_active` /
  `duplicate_inactive` + reaktifasi item lama lewat `set_value` — `11.0`, `11.0b`
- CREATE (minimum, lengkap, jasa, bahan baku) — `11.1`–`11.3b`. **Tanpa pre-check duplikat
  `item_code` & tanpa `item_code`** di payload (kode dibuat backend — lihat §2.1 no. 1 dan §4.1)
- CREATE varian: master Item Attribute, template (**tanpa `item_code`**), `create_variant` + simpan,
  pre-check `get_variant`, manual, daftar varian — `11.4`–`11.8`
- READ single / count / list — `11.9`–`11.11`
- UPDATE (`save`) & `set_value` — `11.12`–`11.13`
- Barcode: create / read / update / scan — `11.14`–`11.17`
- Foto: upload (`attach_file`), tag ke varian (`insert` File), GET per varian, hapus — `11.18`–`11.21`
- Non-aktif (§4.9): GET varian template, cek stok > 0 (`tabBin`), zero stok (insert + submit Stock
  Reconciliation), non-aktif varian & template — `11.22`–`11.25b`
- tabBin (stok) — `11.26`
- UOM: `uoms`, `get_uom_conv_factor`, `UOM Conversion Factor` global — `11.27`–`11.29`
- GET pendukung UI: Item Group leaf, UOM, Brand, Item Attribute, Warehouse/Company, Item Tax
  Template — `11.30`–`11.35`
- Ganti nama template → verifikasi `item_name` varian ikut berubah, plus kasus bentrok yang ditolak
  (§2.1 no. 10) — `11.36`, `11.36b`
- Batch (§2.1 no. 12, §4.3): penerimaan 5 roll **tanpa** nomor (auto `BATCH-00001`…), penerimaan
  **dengan** nomor dari user (cek `get_list` → `insert` Batch → kirim `batch_no` di baris), penjualan
  sebagian per batch — `11.37`–`11.40`
- Hashtag produk (§2.8, §4.12): CREATE dengan hashtag (`"#Promo"` → tersimpan `promo`), UPDATE
  replace-all (kirim 1 hashtag → sisanya terhapus; kirim `[]` → semua terhapus), hashtag tidak valid
  → `417` beserta nomor barisnya, filter **`=`** yang **tidak** menarik `promo2026` (dibandingkan
  `like`), filter `in` yang menduplikasi baris, pencarian **2 tag sekaligus** (2 request + irisan di
  frontend), hashtag **per varian** (template & varian berbeda) + fetch template & semua varian dalam
  1 request, dan daftar hashtag unik (autocomplete) — `11.41`–`11.48`

**Variabel yang perlu diisi** (Collection Variables):
- `item_id` / `item_code` — `name`/`item_code` **hasil CREATE**, diambil dari respons (mis. `2609000001`) — **bukan** dikirim frontend
- `item_name` — **wajib dikirim** saat CREATE (identitas produk yang dibaca user)
- `item_template` — kode template varian **hasil CREATE** (mis. `2609000002`)
- `item_variant` / `item_variant_2` — varian **hasil CREATE**, masing-masing kode seri sendiri (mis. `2609000005`, `2609000006`) — **bukan** bentuk `{template}-{abbr}`
- `item_attribute` / `item_attribute_value` — master atribut & nilainya (mis. `Ukuran`, `M`)
- `item_group` — leaf Item Group (mis. `Minuman`)
- `item_barcode` — barcode hasil CREATE (default = `item_code`, mis. `2609000001`; atau barcode kiriman frontend, mis. `8991234567891`)
- `item_hashtag` / `item_hashtag_2` — hashtag untuk uji filter (mis. `promo`, `promo2026`; lihat §2.8/§4.12)
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
     `valuation_rate`, atau via Stock Entry). Di file ini: info baca `tabBin` (§4.10), **alur zero
     stok via Stock Reconciliation saat non-aktif** (§4.9), dan **penyimpanan ambang stok minimum
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
   Item Group [prd_item_group.md](./prd_item_group.md), **Stock Entry (transfer & pemakaian stok)
   [prd_stock_entry.md](./prd_stock_entry.md)**, Warehouse
   [prd_warehouse.md](../setup/prd_warehouse.md), OAuth [prd_oauth.md](../prd_oauth.md).
   > Field Item yang dipakai Pricing Rule: `max_discount` (batas diskon per item) dan
   > `item_group`/`brand` (dasar `apply_on` pada Pricing Rule).
   > Field Item untuk paket dinamis: `is_dynamic_product_bundle` (custom field `baseapp`) —
   > lihat [prd_item_dynamic_product_bundle.md](./prd_item_dynamic_product_bundle.md).
   > Field Item untuk paket statis: `is_product_bundle` (custom field `baseapp`, Check, diisi manual
   > — tidak lagi diturunkan dari doctype `Product Bundle`) — lihat §2.1 no. 9 dan
   > [prd_item_product_bundle.md](./prd_item_product_bundle.md).
   > Field Item untuk produk manufaktur: `is_manufactured_item` (custom field `baseapp`, Check,
   > diisi manual — bukan turunan BOM) — lihat §2.1 no. 9.
