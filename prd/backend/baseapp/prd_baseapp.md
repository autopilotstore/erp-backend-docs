# PRD — Base App

## 1. Ringkasan

`baseapp` adalah aplikasi kustom Frappe/ERPNext untuk pengaturan awal dan aturan bisnis lintas modul. Aplikasi ini menambahkan metadata dan DocType untuk kebutuhan lokal, menyelaraskan master data, serta mengubah alur penomoran dan validasi Item.

Dokumen ini mencatat perilaku yang ditemukan pada implementasi aplikasi saat ini. `baseapp` tidak menyediakan alur UI khusus; antarmuka Desk standar, REST API Frappe, dan aplikasi pemanggil menggunakan field, DocType, hook, serta endpoint yang disediakan.

## 2. Tujuan

- Menyeragamkan konfigurasi ERPNext dan data master awal pada setiap site.
- Menetapkan kebijakan password awal: policy aktif dengan skor minimum `1`.
- Mendukung kode Item berurutan yang juga dapat dipakai sebagai barcode.
- Menjaga nama Item aktif tetap unik dan membantu UI menangani duplikasi.
- Menyinkronkan rate Item dengan Item Price pada `Standard Selling` dan `Standard Buying`.
- Melindungi Price List standar yang dibutuhkan sinkronisasi agar tidak dapat dihapus.
- Menyediakan penanda manual Item untuk Product Bundle statis, Product Bundle dinamis, dan produk manufaktur.
- Menyediakan struktur konfigurasi untuk pilihan komponen paket dinamis.
- Menambahkan data pendukung untuk negara, tarif pajak, dan reorder Item.
- Menambahkan konfigurasi batas kuantitas penjualan pada Item.
- Menyediakan penyimpanan hashtag produk yang dapat difilter secara persis (child table `Item Hashtag`).

## 3. Ruang Lingkup

### Termasuk

- Custom field dan Property Setter pada DocType ERPNext.
- Tiga DocType untuk konfigurasi Dynamic Product Bundle.
- Satu DocType child table (`Item Hashtag`) beserta field `Item.hashtags` dan normalisasi isinya.
- Hook dokumen untuk Contact, Item, Item Price, Price List, dan Item Attribute.
- Override controller Item Attribute untuk mencegah perubahan kode varian ketika singkatan atribut diubah.
- Endpoint pemeriksaan nama Item.
- Penetapan nilai Single pada `System Settings` untuk kebijakan password saat instalasi.
- Sinkronisasi dan seed konfigurasi/master data saat instalasi dan migrasi.
- Patch untuk site yang sudah memasang aplikasi.

### Tidak termasuk

- UI kasir atau UI konfigurasi paket di frontend.
- Logika transaksi untuk memilih/mengemas komponen Dynamic Product Bundle.
- Validasi batas minimum, maksimum, dan kelipatan kuantitas pada transaksi penjualan.
- Perhitungan stok/reorder dari `max_stock_level`.
- Sinkronisasi data negara selain `phone_code` dan `flag_emoji`.
- Penanganan seluruh alur duplikasi nama di UI; endpoint hanya menyediakan hasil pemeriksaan bagi pemanggil.
- Pewarisan/penyalinan hashtag dari Item template ke variannya; hal itu diatur pemanggil (frontend) dengan menyimpan hashtag tiap varian.
- Migrasi data tag lama dari `_user_tags` ke `hashtags`.

## 4. Data dan Konfigurasi

### 4.1 Custom field

| DocType | Field | Tipe | Perilaku |
|---|---|---|---|
| `Country` | `phone_code` | Data | Kode telepon negara; diisi dari `data/country.json` berdasarkan kode ISO-2. |
| `Country` | `flag_emoji` | Data | Emoji bendera negara; diisi dari `data/country.json` berdasarkan kode ISO-2. |
| `Sales Taxes and Charges` | `custom_rates` | JSON | Daftar tarif alternatif pada baris pajak, diletakkan setelah `rate`. Nilai REST harus berupa string JSON, misalnya `"[5, 10, 15]"`; saat dibaca nilainya juga berupa string. |
| `Item` | `is_dynamic_product_bundle` | Check | Default `0`; penanda bahwa Item adalah paket dinamis. Tersedia sebagai filter standar. Konfigurasi komponennya ada pada tiga DocType Dynamic Product Bundle. |
| `Item` | `is_product_bundle` | Check | Default `0`; penanda paket statis (doctype `Product Bundle`) yang **diisi manual**. Tersedia sebagai filter standar. Tidak ada hook/turunan yang mengubahnya mengikuti status aktif/non-aktif Product Bundle. |
| `Item` | `is_manufactured_item` | Check | Default `0`; penanda produk manufaktur (diproduksi sendiri). Tersedia sebagai filter standar. Diisi manual oleh pengguna/frontend — tidak diturunkan dari BOM. |
| `Item` | `min_sales_qty` | Float | Batas kuantitas jual minimum per Item dalam Stock UOM; default `0` berarti tidak ada batas minimum. |
| `Item` | `max_sales_qty` | Float | Batas kuantitas jual maksimum per Item dalam Stock UOM; default `0` berarti tidak ada batas maksimum. |
| `Item` | `sales_qty_multiple` | Float | Kelipatan kuantitas jual per Item dalam Stock UOM; default `0` berarti aturan kelipatan tidak digunakan. |
| `Item` | `hashtags` | Table → `Item Hashtag` | Daftar hashtag produk — **satu baris = satu hashtag**, ditulis tanpa `#` dan huruf kecil. Bukan field `reqd` (boleh kosong) dan isinya dirapikan otomatis saat `validate` (§5.8). Setiap Item — template maupun varian — punya daftar sendiri. |
| `Item Reorder` | `max_stock_level` | Float | Ambang stok maksimum per Item dan gudang; non-negatif. Informasional/notifikasi saja dan tidak mengubah logika stok ERPNext. |

Field-field tersebut dibuat secara idempoten saat `after_migrate` dan instalasi. Tiga field kuantitas
penjualan Item merupakan custom field bawaan `baseapp` dan tersimpan sebagai kolom di `tabItem`.
Field tersebut hanya menyimpan konfigurasi; validasi transaksi penjualan untuk menegakkan batas atau
kelipatannya belum termasuk dalam cakupan aplikasi.

### 4.2 Property Setter

- `Contact.status`: nilai default menjadi `Open`; hook validasi juga mengisi `Open` bila status kosong.
- `Item.naming_series`: opsi penomoran menjadi `YY.MM.######`.
- `Item Attribute Value.abbr`: tidak wajib dan disembunyikan di Desk; nilainya diturunkan dari `attribute_value`.
- Aktivasi mode penamaan berbasis Naming Series pada `Stock Settings` memicu Property Setter bawaan ERPNext untuk menampilkan `naming_series` dan menyembunyikan/menjadikan `item_code` tidak wajib.

Perubahan `Contact.status` dan visibilitas/keharusan `abbr` ditegakkan kembali pada setiap migrasi. Pengaturan Item Naming Series diterapkan satu kali saat instalasi atau melalui patch, sehingga perubahan yang disengaja setelah go-live tidak ditimpa setiap migrasi.

### 4.3 DocType Dynamic Product Bundle

Ketiga DocType berada di modul `Base App`. Hak akses Create, Read, Write, Delete, Export, Print, dan Share diberikan kepada `System Manager` dan `Item Manager`; `System Manager` juga memiliki Import dan Email. Perubahan dilacak (`track_changes`).

#### `Dynamic Product Bundle Option`

Mendefinisikan satu pilihan komponen untuk sebuah Item paket.

| Field | Tipe | Keterangan |
|---|---|---|
| `bundle_item` | Link → Item | Item paket pemilik opsi; wajib dan tersedia di filter standar. |
| `option_name` | Data | Nama pilihan; wajib dan menjadi bagian nama dokumen. |
| `seq` | Int | Urutan tampilan, default `0`. |
| `min_qty` | Float | Jumlah total minimum pilihan yang wajib diambil; `0` berarti tidak wajib; non-negatif. |
| `max_qty` | Float | Jumlah total maksimum yang boleh diambil; `0` berarti tanpa batas; non-negatif. |
| `filter_type` | Select | Cara membatasi pilihan: `Item` atau `Item Group`; wajib, default `Item`. |

Nama dokumen dibentuk sebagai `{option_name} - {bundle_item}` dan urutan daftar berdasarkan `seq` menaik.

#### `Dynamic Product Bundle Item`

Mendefinisikan komponen tetap atau Item yang dapat dipilih pada suatu opsi.

| Field | Tipe | Keterangan |
|---|---|---|
| `bundle_item` | Link → Item | Item paket pemilik baris; wajib dan tersedia di filter standar. |
| `bundle_option` | Link → Dynamic Product Bundle Option | Opsi pemilik baris. Kosong berarti komponen tetap; dipakai untuk komponen pilihan dengan `filter_type = Item`. |
| `item` | Link → Item | Item komponen; wajib. |
| `min_qty` | Float | Minimum jumlah komponen; `0` berarti opsional; non-negatif. |
| `max_qty` | Float | Maksimum jumlah komponen; `0` berarti tanpa batas; non-negatif. |
| `uom` | Link → UOM | Satuan, wajib, otomatis diambil dari `item.stock_uom` jika kosong. |

#### `Dynamic Product Bundle Item Group`

Membatasi pilihan opsi bertipe `Item Group` pada grup Item tertentu beserta subgrup turunannya.

| Field | Tipe | Keterangan |
|---|---|---|
| `bundle_item` | Link → Item | Item paket pemilik baris; wajib dan tersedia di filter standar. |
| `bundle_option` | Link → Dynamic Product Bundle Option | Opsi pilihan; wajib. |
| `item_group` | Link → Item Group | Grup yang anggotanya dan subgrup turunannya boleh dipilih; wajib dan tersedia di filter standar. |

Validasi relasi tambahan di luar field wajib/Link tersebut dan eksekusi pilihan komponen pada transaksi tidak ditemukan di aplikasi ini.

### 4.4 DocType Item Hashtag

Child table untuk menyimpan hashtag produk. Berada di modul `Base App`, dan **hanya bermakna sebagai
child table** (`istable = 1`): tidak punya definisi permission sendiri sehingga tidak dapat dicari
langsung lewat `frappe.client.get_list` dengan `doctype` = `Item Hashtag` (Frappe hanya mengembalikan
`name`). Seluruh akses lewat dokumen `Item`.

| Field | Tipe | Keterangan |
|---|---|---|
| `hashtag` | Data | Satu hashtag; wajib, terindeks (`search_index`), tampil di list view. Ditulis tanpa `#`, huruf kecil, cocok pola `^[a-z0-9_-]{1,50}$`. |

Nama dokumen acak (`hash`). Karena `hashtags` adalah child table pada Item, daftar hashtag sebuah
Item disimpan lewat dokumen Item (`insert`/`save`) dan bersifat **replace-all**: baris yang tidak ikut
terkirim akan terhapus. Filter memakai operator `=` pada field child-nya, mis.
`[["hashtags.hashtag","=","promo"]]`.

## 5. Perilaku Bisnis

### 5.1 Contact dan Lead

- Saat `Contact.validate`, status yang kosong diisi menjadi `Open`.
- Default field `Contact.status` juga ditetapkan ke `Open`.
- `CRM Settings.auto_creation_of_contact` dinonaktifkan agar pembuatan Lead tidak otomatis membuat Contact.

### 5.2 Item dan penomoran

- Item baru menggunakan Naming Series `YY.MM.######` (contoh format kode: `2609000001`, dengan prefix tahun-bulan dan penghitung enam digit).
- Mode Naming Series diaktifkan melalui `Stock Settings`. Dalam mode ini ERPNext menghasilkan kode Item dari server.
- Item varian juga mendapat kode dari penghitung bulanan yang sama, bukan kode `{template}-{abbr}`. Nama yang dapat dibaca tetap dibentuk dari nama template dan singkatan atribut; jika Item varian dibuat manual tanpa nama, nama tersebut dihasilkan dari logika ERPNext.
- Pada Item baru yang tidak mengirim barcode, baseapp menambahkan barcode sama dengan `item_code` memakai `stock_uom`. Barcode yang sudah dikirim tidak ditimpa. Tipe barcode dibiarkan kosong agar pemeriksaan check digit ERPNext tidak menolak kode serial.
- Nama Item dibandingkan tanpa membedakan kapitalisasi dan mengabaikan spasi di awal/akhir. Nama yang sudah dipakai Item aktif menolak penyimpanan; nama yang hanya dipakai Item nonaktif tetap diperbolehkan.
- Saat nama Item template varian berubah, nama varian dihitung ulang dengan algoritme ERPNext. Kode Item/barcode varian tidak berubah. Bila ditemukan benturan nama aktif atau dua varian baru akan mendapat nama sama, perubahan template ditolak sebelum nama varian ditulis.
- Sinkronisasi nama varian hanya berjalan saat nama template berubah dan berlangsung sinkron. Varian yang sudah tidak konsisten tidak diperbaiki otomatis kecuali template-nya diubah.
- Saat Item baru dibuat, `standard_rate` (harga jual) disinkronkan ke `price_list_rate` di `Standard Selling`, sedangkan `valuation_rate` (harga modal) disinkronkan ke `price_list_rate` di `Standard Buying`. Baris yang dipakai adalah harga umum untuk `stock_uom`; harga khusus customer/supplier, batch, UOM lain, atau yang belum/sudah tidak berlaku tidak dipakai.
- Ketika `price_list_rate` berubah pada Item Price umum dengan UOM stok yang masih berlaku di `Standard Selling` atau `Standard Buying`, baseapp memperbarui `Item.standard_rate` atau `Item.valuation_rate` sesuai pemetaan tersebut. Mengubah rate pada Item yang sudah ada tidak menjalankan sinkronisasi balik ke Item Price.
- Price List bernama `Standard Selling` dan `Standard Buying` ditolak saat dihapus karena dibutuhkan sinkronisasi rate. Daftar tersebut dapat dinonaktifkan bila tidak ingin dipakai.

### 5.3 Atribut varian

- Untuk atribut non-numerik, `attribute_value` menjadi nilai kanonis dan `abbr` diselaraskan ke nilai yang sama setelah spasi luar dipangkas.
- Pengguna cukup mengisi `attribute_value`; `abbr` disembunyikan dan tidak wajib.
- Baris tanpa keduanya ditolak dengan pesan validasi.
- Atribut numerik mengikuti perilaku ERPNext; sinkronisasi ini tidak mengubah tabel nilai numerik.
- Perubahan `abbr` tidak boleh memicu ERPNext mengganti kode Item varian. Override menjalankan `ItemAttribute.on_update()` asli dan sementara menonaktifkan hanya fungsi cascade pengubahan kode varian. Propagasi perubahan `attribute_value` tetap berjalan.

### 5.4 Product Bundle statis

- Doctype `Product Bundle` bawaan ERPNext dipakai apa adanya; baseapp tidak mengubah perilakunya.
- `Item.is_product_bundle` adalah **penanda manual**: user/frontend menentukan nilai `0`/`1`. Tidak ada hook, perhitungan turunan, atau patch backfill yang mengisinya.
- Membuat, mengubah, mengaktifkan/menonaktifkan, atau menghapus `Product Bundle` **tidak mengubah** flag ini.
- Perubahan flag mengikuti jalur biasa (`insert`/`save`/`set_value`) dan ikut memperbarui timestamp `modified` Item.

### 5.5 Produk manufaktur

- `Item.is_manufactured_item` (Check, default `0`) menandai Item yang merupakan produk manufaktur (diproduksi sendiri).
- Field ini **tidak diturunkan** dari dokumen lain: ERPNext tidak menyimpan penanda "manufactured" pada Item (yang tersedia hanya `default_bom`, `include_item_in_manufacturing`, dan `is_sub_contracted_item`), sehingga nilainya diisi pengguna/frontend dan disimpan apa adanya. Tidak ada hook atau patch yang menulis ulang flag ini.
- Tersedia sebagai filter standar sehingga dapat dipakai pada `frappe.client.get_list`/`get_count` bersama penanda produk lain.

### 5.6 Master data dan akun awal

Saat pengaturan ditegakkan pada instalasi/migrasi:

- `Country.phone_code` dan `Country.flag_emoji` diperbarui dari `data/country.json` melalui pencocokan ISO-2 (`Country.code`).
- Master Gender dipertahankan sebagai `Male`, `Female`, dan `Prefer not to say`; nilai lain dihapus jika tidak sedang direferensikan.
- Role `All` dinonaktifkan agar tidak muncul dalam daftar role.
- Warehouse Type `Store` dibuat jika belum tersedia.
- Untuk setiap Company, akun ledger berikut dibuat bila parent account bernomor terkait ditemukan dan akun belum ada:
  - `1120.001` — `Rekening Bank Utama`, tipe akun `Bank`, di bawah grup `1120.000`.
  - `1132.002` — `Piutang Payment Gateway`, akun ledger generik tanpa tipe akun, di bawah grup `1132.000`.
- Akun tidak dibuat bila parent group tidak ada; kegagalan penciptaan dicatat di error log dan tidak menghentikan pemrosesan perusahaan lain.

### 5.6 Penyederhanaan Item Group

Saat instalasi baru atau patch satu kali, master Item Group dikonsolidasikan menjadi:

- `All Item Groups` sebagai root.
- `Non Category` sebagai satu-satunya grup daun yang dapat dipilih.

Referensi `item_group` dan `other_item_group` pada seluruh tabel DocType yang terdeteksi di skema dialihkan ke `Non Category`, termasuk data historis dan konfigurasi. `Stock Settings.item_group` juga diubah ke `Non Category`; grup lainnya dihapus.

**Peringatan:** proses ini menghapus seluruh Item Group lain, termasuk grup buatan pengguna. Proses sengaja tidak dijalankan setiap migrasi untuk mencegah penghapusan grup baru setelah go-live.

### 5.7 Kebijakan password

Saat instalasi — dan lewat patch untuk site yang sudah memasang aplikasi — `System Settings`
disetel menjadi:

| Field | Nilai | Bawaan Frappe |
|---|---|---|
| `enable_password_policy` | `1` | `1` |
| `minimum_password_score` | **`1`** | `2` |

- Nilai ditulis langsung ke `tabSingles` (`frappe.db.set_single_value`), bukan lewat `save()`,
  sehingga `SystemSettings.validate()` tidak dijalankan.
- **Keduanya ditulis bersamaan karena skor hanya berlaku bila policy aktif.** `User.test_password_strength()`
  berhenti lebih awal (`return {}`) saat `enable_password_policy` nonaktif
  (`frappe/core/doctype/user/user.py:995-998`), dan `SystemSettings.validate()` mengosongkan skor
  pada **setiap** penyimpanan selama policy nonaktif
  (`frappe/core/doctype/system_settings/system_settings.py:122-127`) — termasuk penyimpanan lewat REST
  API, karena `frappe.client.save` dan `frappe.client.set_value` sama-sama menjalankan `validate()`.
  Menulis skor saja akan sia-sia dan cepat hilang.
- `enable_password_policy` ditulis eksplisit meski bawaan Frappe sudah `1`, agar skor tidak
  kosong lagi bila ada yang pernah menonaktifkan policy.
- Cache dibersihkan setelah penulisan supaya policy langsung berlaku tanpa perlu restart;
  pembacaan runtime lewat `get_system_settings()` dapat masih memakai salinan lama selama cache
  belum dibersihkan.
- **Skor `1` berarti hanya password paling lemah yang ditolak** (skor `0`). Password yang sudah ada
  tidak diubah dan tidak ada pengguna yang dipaksa mengganti password.
- **Satu kali saja.** Tidak dijalankan ulang setiap migrasi, sehingga perubahan policy yang
  disengaja setelah go-live tidak ditimpa.

### 5.8 Hashtag Item

- Hashtag produk disimpan di child table `hashtags` (`Item Hashtag`) — **bukan** `_user_tags`.
  `_user_tags` adalah satu kolom teks dipisah koma, sehingga filter `like` untuk `promo` ikut
  menarik `promo2026`; filter yang tepat untuk kolom seperti itu hanya `regex`, yang tidak dapat
  memakai indeks. Dengan child table, satu hashtag = satu baris, sehingga filter `= promo` tepat dan
  terindeks.
- `Item.validate` menjalankan `normalize_item_hashtags`: spasi luar dipangkas, `#` di depan dibuang,
  huruf diubah ke kecil, baris kosong dibuang, dan duplikat di dalam satu Item dibuang (yang pertama
  dipertahankan). Normalisasi berjalan **sebelum** data ditulis (`validate` dijalankan sebelum
  `update_children()`), jadi nilai yang tersimpan selalu kanonik.
- Nilai yang tidak cocok pola `^[a-z0-9_-]{1,50}$` **ditolak** dengan pesan yang menyebut nomor
  barisnya, mis. `Row #1: hashtag promo 2026 is not valid. Use lowercase letters, digits, - and _ only
  (max 50 characters).` Tolakan ini **tidak** berlaku selama install, migrate, atau patch — pada
  kondisi tersebut nilai hanya dinormalkan (dan baris tidak valid dibuang) agar proses seeding tidak
  gagal.
- Hashtag bersifat per Item: **template dan setiap varian punya daftar sendiri**, dan template
  **tidak** mewariskan hashtagnya ke varian. Sebabnya `copy_attributes_to_variant()` hanya menyalin
  field `reqd` atau field yang terdaftar di doctype `Variant Field`, dan `hashtags` bukan keduanya.
  Pewarisan, bila diinginkan, diatur pemanggil (frontend) dengan menyimpan hashtag tiap varian.
- Mengganti/menghapus hashtag dilakukan dengan mengirim **seluruh daftar** yang diinginkan
  (replace-all); mengirim daftar kosong menghapus semuanya. `frappe.client.set_value` **tidak** dapat
  menulis child table, jadi perubahan harus lewat `insert`/`save` dokumen Item.
- Tidak ada backfill dari `_user_tags` atau sumber lain: hashtag hanya terisi lewat dokumen Item.

## 6. Hook dan Endpoint

### 6.1 Siklus instalasi/migrasi

- `after_install`: menjalankan `enforce_baseapp_settings`, konsolidasi Item Group, aktivasi Item Naming Series, dan penetapan kebijakan password.
- `after_migrate`: menjalankan `enforce_baseapp_settings` secara idempoten.
- Patch yang tercantum di `patches.txt` menangani site yang sudah memasang aplikasi; patch yang terkait instalasi fresh tidak mengulang perubahan destruktif/perubahan satu kali pada setiap migrasi.

### 6.2 Document event

| DocType | Event | Handler | Fungsi |
|---|---|---|---|
| Contact | `validate` | `set_contact_status_open` | Mengisi status `Open` jika kosong. |
| Item | `before_insert` | `assign_variant_item_code` | Memberi kode serial pada varian sebelum proses autoname. |
| Item | `after_insert` | `sync_standard_item_prices` | Menyinkronkan rate Item ke Item Price umum UOM stok di `Standard Selling` dan `Standard Buying`. |
| Item | `before_validate` | `set_default_item_barcode` | Menambahkan barcode default saat Item baru belum memilikinya. |
| Item | `validate` | `normalize_item_hashtags` | Menormalkan hashtag Item (buang `#`, spasi luar, duplikat, dan baris kosong) dan menolak nilai yang tidak sesuai pola beserta nomor barisnya. Tolakan dilewati selama install, migrate, dan patch. |
| Item | `validate` | `prevent_duplicate_item_name` | Menolak nama Item yang sudah digunakan Item aktif. Dilewati selama install, migrate, dan patch. |
| Item | `on_update` | `sync_variant_item_names` | Memperbarui nama varian setelah nama template berubah. |
| Item Price | `on_update` | `sync_item_rate_from_standard_price` | Menyinkronkan perubahan harga umum UOM stok pada Price List standar ke field rate Item yang dipetakan. |
| Price List | `on_trash` | `prevent_standard_price_list_deletion` | Menolak penghapusan `Standard Selling` dan `Standard Buying`. |
| Item Attribute | `before_validate` | `sync_attribute_value_and_abbr` | Memvalidasi nilai atribut dan menyelaraskan `abbr`. |
### 6.3 Override controller

`extend_doctype_class` menambahkan `CustomItemAttribute` pada `Item Attribute`. Override tidak menyalin seluruh implementasi ERPNext; method `on_update` ERPNext tetap dipanggil, dengan cascade pengubahan kode varian akibat perubahan singkatan dinonaktifkan hanya selama pemanggilan tersebut.

**Catatan upgrade ERPNext:** override bergantung pada nama fungsi internal `update_variant_item_codes_for_abbr_renames` di modul ERPNext. Jika nama/fungsi tersebut berubah, override dapat menjadi tidak efektif dan perilaku cascade ERPNext dapat kembali. Perlu pemeriksaan regresi setelah upgrade ERPNext.

### 6.4 API pemeriksaan nama Item

- Endpoint whitelist: `baseapp.api.item_name.check_item_name`.
- Memerlukan izin baca pada DocType `Item`.
- Parameter: `item_name` dan `exclude` opsional (nama Item yang sedang diedit).
- Respons: `item_name`, `normalized`, `status`, `active`, dan `inactive`.
- Nilai `status`: `duplicate_active`, `duplicate_inactive`, atau `ok`.
- Pencocokan mengabaikan kapitalisasi dan spasi luar. Daftar hasil dibatasi maksimal 50 Item.
- Pemanggil dapat menggunakan hasil untuk meminta nama lain atau menawarkan aktivasi ulang Item nonaktif. API ini hanya memeriksa; tidak mengubah Item atau mengaktifkan ulang Item.

## 7. Patch dan Migrasi

Patch yang terdaftar di `baseapp/patches.txt`:

| Patch | Tujuan |
|---|---|
| `change_contact_default_status` | Menetapkan Property Setter default `Contact.status` menjadi `Open`. |
| `disable_lead_contact_auto_creation` | Menonaktifkan auto-creation Contact dari Lead. |
| `collapse_item_groups` | Menjalankan konsolidasi Item Group satu kali. |
| `enable_item_naming_series` | Mengaktifkan format dan mode Naming Series Item satu kali. |
| `set_minimum_password_score` | Mengaktifkan kebijakan password dan menetapkan skor minimum `1` satu kali. |
Field kustom dan pengaturan yang memang perlu selalu ditegakkan diselaraskan oleh `after_migrate`. Pengaturan Naming Series dan konsolidasi Item Group sengaja tidak ditegakkan terus-menerus.

Field `Item.hashtags` dan DocType `Item Hashtag` **tidak** memerlukan patch tersendiri: keduanya
dibuat ulang secara idempoten oleh `enforce_baseapp_settings()` pada setiap `after_migrate`, dan tidak
ada data lama yang perlu di-backfill. Site yang sudah memasang aplikasi cukup menjalankan migrasi
biasa untuk mendapatkan field tersebut.

> **Catatan penomoran dokumen:** §5 punya dua sub-bab bernomor `5.6` (`Master data dan akun awal` dan
> `Penyederhanaan Item Group`). Penomoran itu dibiarkan apa adanya agar referensi lama tidak patah;
> sub-bab baru memakai nomor berikutnya (`5.8`).

## 8. Kriteria Penerimaan

1. Instalasi/migrasi membuat field kustom yang tercantum tanpa duplikasi dan mengisi data negara yang cocok dengan ISO-2.
2. Contact baru tanpa status tersimpan dengan status `Open`; pembuatan Lead tidak otomatis menghasilkan Contact.
3. Item baru tanpa barcode mendapat barcode sama dengan kode Item; barcode yang diberikan pengguna tetap dipertahankan.
4. Kode Item baru dan varian mengikuti Naming Series bulanan; nama varian tetap terbaca dan perubahan nama template menyinkronkan nama seluruh variannya tanpa mengganti kode/barcode.
5. Penyimpanan Item ditolak jika nama yang dinormalisasi telah digunakan Item aktif, tetapi nama milik Item nonaktif dapat digunakan.
6. Endpoint pemeriksaan nama membedakan duplikasi aktif, nonaktif, dan nama bebas serta mengecualikan Item yang sedang diedit.
7. Atribut non-numerik tidak memerlukan input `abbr`; `abbr` tersimpan sama dengan `attribute_value`, dan perubahan singkatan tidak mengganti kode Item varian.
8. Flag `Item.is_product_bundle` tersimpan apa adanya dari input pengguna/frontend dan tidak berubah ketika Product Bundle dibuat, diubah, dinonaktifkan, atau dihapus.
9. Pembuatan Item menyinkronkan `standard_rate` ke `Standard Selling` dan `valuation_rate` ke `Standard Buying` untuk harga umum UOM stok; perubahan harga yang berlaku pada Item Price kedua daftar memperbarui field Item terkait.
10. Penghapusan `Standard Selling` atau `Standard Buying` ditolak; keduanya dapat dinonaktifkan.
11. Dynamic Product Bundle dapat dikonfigurasi dengan opsi, Item komponen, dan pembatasan Item Group sesuai field dan hak akses yang ditentukan.
12. Batas `max_stock_level` tersimpan sebagai data informasional tanpa mengubah perhitungan stok ERPNext.
13. Konsolidasi Item Group memindahkan referensi ke `Non Category`, mengubah default Stock Settings, dan tidak berjalan berulang pada migrasi berikutnya.
14. Seed akun hanya dilakukan saat parent account tersedia; site/company lain tetap dapat diproses jika satu seed gagal.
15. Instalasi menetapkan `System Settings` menjadi `enable_password_policy = 1` dan `minimum_password_score = 1`; kebijakan langsung berlaku tanpa restart, dan tidak dijalankan ulang pada migrasi berikutnya.
16. Field `Item.is_manufactured_item` dibuat ulang secara idempoten pada instalasi/migrasi, tersedia sebagai filter standar, dan nilainya dipertahankan apa adanya (bukan turunan BOM).
17. DocType `Item Hashtag` dan field `Item.hashtags` dibuat ulang secara idempoten pada instalasi/migrasi; hashtag dengan `#`, huruf besar, baris kosong, dan duplikat tersimpan dalam bentuk kanonik, nilai di luar pola `^[a-z0-9_-]{1,50}$` ditolak beserta nomor barisnya, filter `=` hanya mengembalikan Item dengan hashtag tersebut (bukan yang mengandungnya), dan hashtag varian tidak diwarisi dari template.

## 9. Catatan Implementasi

- Sumber utama konfigurasi: `baseapp/hooks.py`, `baseapp/utils.py`, `baseapp/overrides/item_attribute.py`, `baseapp/api/item_name.py`, metadata DocType pada `baseapp/base_app/doctype/`, dan patch pada `baseapp/patches/`.
- Metadata DocType saja belum mengimplementasikan validasi rentang min/max, pemilihan paket pada transaksi, atau eksekusi harga/stock untuk Dynamic Product Bundle; fungsi-fungsi tersebut harus disediakan oleh komponen pemanggil bila diperlukan.
- `custom_rates` adalah string berisi JSON sesuai perilaku field JSON Frappe; klien perlu melakukan parse saat membaca dan mengirim serialisasi JSON sebagai string.
