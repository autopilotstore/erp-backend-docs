# PRD — REST API Doctype Item Group (ERPNext / Frappe)

> Dokumen spesifikasi pemanggilan REST API untuk **Item Group** di ERPNext (Frappe),
> diperuntukkan bagi tim **UI/Frontend**.

- **Modul:** Setup (ERPNext) — path `erpnext.setup.doctype.item_group`
- **Doctype:** `Item Group`
- **Versi API:** `/api/method/...` (API v1) — method whitelisted `frappe.client.*`, `name` dikirim di **body**
- **Autentikasi:** OAuth 2.0 — Authorization Code + Refresh Token
- **Format body:** JSON
- **Base URL:** ganti `https://site-anda.com` dengan alamat site Anda (mis. dev: `https://erpnext.localhost`)

---

## 1. Ruang lingkup

| # | Endpoint (POST `/api/method/...`) | Operasi | `name` di |
|---|---|---|---|
| 1 | `frappe.client.insert` | Buat Item Group baru (CREATE) | body (`doc`) |
| 2 | `frappe.client.get` | Ambil detail 1 Item Group (READ) | body |
| 3 | `frappe.client.get_list` | Daftar Item Group (READ list) | body (filters) |
| 4 | `frappe.client.save` | Ubah Item Group (UPDATE) | body (`doc`) |
| 5 | `frappe.client.set_value` | Ubah field tunggal — pindah `parent_item_group`, ganti `is_group`, dst. | body |
| 6 | `frappe.client.get_list` | Daftar Company (dropdown `item_group_defaults`, §5.3) | body |
| 7 | `frappe.client.get_list` | Daftar Item Tax Template (dropdown `taxes`, §5.4) | body |
| 8 | `frappe.client.get_list` | Daftar Warehouse / Cost Center / Account (dropdown `item_group_defaults`, §5.5) | body |
| 9 | `frappe.client.get_count` | Total record Item Group sesuai filter — pagination (§4.2) | body |
| 10 | `frappe.desk.treeview.get_children` (GET) | Ambil node tree Item Group (§5.1) | query |
| 11 | `frappe.client.get_list` | Daftar Item Group leaf (dropdown UI — POS Profile / Item, §5.2) | body |
| 12 | `erpnext.setup.doctype.item_group.item_group.get_company_resolved_defaults` | Resolve default dari Company utk prefill `item_group_defaults` (opsional, §5.6) | body |
| 13 | `frappe.client.delete` | HAPUS — **diblokir** bila ada child/Item; tanpa soft-delete (§4.5) | body |
| 14 | `frappe.client.get_list` | Daftar Item dalam leaf/group — utk pemindahan saat leaf dipecah (§5.7) | body (filters) |

> **Konvensi pemanggilan (penting):** seluruh operasi memakai method whitelisted **`frappe.client.*`**
> dengan `name` (dan filter) dikirim lewat **body JSON**, bukan di URL path. `name` Item Group bisa
> mengandung spasi / karakter khusus — mengirim `name` di body menghindari masalah URL-encoding
> di belakang nginx/proxy.
> **Format respons:** method `/api/method/...` membungkus hasil di **`"message"`** (bukan `"data"`
> seperti `/api/resource/...`).

> **Perbedaan utama dengan Warehouse:**
> - `name = item_group_name` (**tanpa** suffix `- abbr` company). Contoh: `item_group_name="Minuman"`
>   → `name = "Minuman"`.
> - Pohon Item Group bersifat **global** (tidak per company). Root bawaan install: `All Item Groups`.
> - Item Group **TIDAK punya field `disabled`** → tidak ada operasi non-aktif (soft-delete). Lihat §2.1 no. 4.

> Dokumen ini hanya membahas **data wajib terisi** + field penting. Export/import masal
> **tidak dipakai** di aplikasi ini (tidak didokumentasikan).

---

## 2. Ringkasan field & data

| Status | Field | Tipe | Keterangan |
|---|---|---|---|
| 🔴 **WAJIB** | `item_group_name` | Data | Nama group. `reqd: 1` dan **`unique: 1`** — sekaligus menjadi `name` (autoname `field:item_group_name`, tanpa suffix). |
| 🟠 **DISARANKAN** | `parent_item_group` | Link → Item Group | Node tree. Hanya node `is_group=1`, **bukan dirinya sendiri**. Kosong = root (otomatis `All Item Groups` saat create, §2.1 no. 3). |
| 🟠 **DISARANKAN** | `is_group` | Check | `1` = group induk (node tree); `0` = group daun yang bisa menampung Item. Default `0`. |
| 🟠 | `item_group_defaults` | Table → Item Default | Default per company yang **diwarisi Item** (`default_warehouse`, cost center, akun, dst.). §2.2. |
| 🟠 | `taxes` | Table → Item Tax | Pajak default yang **diwarisi Item** (`item_tax_template` + `tax_category`). §2.3. |
| 🟠 | `image` | Attach Image | Gambar/ikon group (tersembunyi di form v16, tetap valid via API). |
| ⚪ **Otomatis — jangan dikirim** | `name` | — | `name = item_group_name` (tanpa naming series / suffix). |
| ⚪ **Read-only — jangan dikirim** | `lft`, `rgt`, `old_parent` | — | Dikelola sistem tree (NestedSet). |
| ✖️ **Tidak ada** | `disabled` | — | Item Group **tidak punya field `disabled`** — tanpa soft-delete (§2.1 no. 4). |

> Catatan: **field website tidak tersedia** di versi ini (v16). Field `show_in_website`, `route`,
> `website_title`, `slideshow`, `show_in_desktop`, `show_in_purchase`, `show_in_sales` **sudah
> dihapus** dari doctype `Item Group`. Jangan dikirim.

### 2.1 Catatan penting

1. **`name` = `item_group_name` (unique, tanpa suffix)** — bukan naming series dan **tidak** memakai
   abbr company. Karena `unique: 1`, duplikat nama ditolak di level database → cek duplikat sebelum
   CREATE (§4.1 Langkah 0). Mengubah `item_group_name` pada dokumen yang ada **tidak mengubah `name`**
   (autoname hanya saat insert). Bila harus rename, gunakan `frappe.client.rename_doc`
   (whitelisted; doctype ini `allow_rename: 1`).
2. **Pohon global (NestedSet)** — Item Group **tidak per company** (beda dari Warehouse). Satu pohon
   dipakai semua company; root bawaan install = **`All Item Groups`**. `lft/rgt` dikelola otomatis.
3. **`parent_item_group`** harus node `is_group=1` dan tidak boleh menunjuk ke dirinya sendiri
   (backend mencegah loop: *"Item cannot be added to its own descendants"*). Bila kosong saat create,
   sistem otomatis menetapkan root. **Aturan tambahan:** node leaf (`is_group=0`) **tidak boleh punya
   child** — ubah `is_group=1` dulu sebelum menambah child di bawahnya (backend: *"{name} cannot be a
   leaf node as it has children"*).
   **Hanya leaf yang menampung Item yang ditransaksikan** — saat sebuah leaf dipecah menjadi group,
   semua Item di dalamnya harus dipindahkan ke leaf baru (§4.6).
4. **Tidak ada `disabled` — tanpa soft-delete.** Item Group **tidak punya field `disabled`**, sehingga
   pola non-aktif di Warehouse **tidak berlaku**. `frappe.client.delete` **diblokir** bila:
   - masih punya **child node** → `NestedSetChildExistsError` (*"Cannot delete {name} as it has child nodes"*), atau
   - **direferensikan Item / doctype lain** → `LinkExistsError` (*"Cannot delete because of linked records"*).
   Karena aturan master data "jangan hapus" + tanpa soft-delete: **jangan hapus dan jangan non-aktifkan** —
   group yang tidak terpakai cukup dibiarkan.
5. **Role yang dibutuhkan** (v16) — baca: `Stock Manager` / `Stock User` / `Sales User` /
   `Purchase User` / `Accounts User` / `Desk User`; **tulis/buat/hapus: `Item Manager`**.
6. **Item Group bisa dibatasi per user?** Secara teknis bisa via `User Permission`
   (`allow = "Item Group"`, `for_value = <name group>`). Namun karena Item Group adalah master **global**,
   umumnya **tidak dibatasi per user** — alur ini tidak didokumentasikan sebagai alur utama.

### 2.2 `item_group_defaults` — child table `Item Default` (default per company)

- Tabel child `item_group_defaults` (options **`Item Default`**), **satu baris per `company`**
  (`company` = Link → Company, `reqd`).
- Field yang umum dipakai: `default_warehouse`, `default_price_list`, `default_supplier`,
  `buying_cost_center`, `selling_cost_center`, `expense_account`, `income_account`,
  `default_inventory_account`, `default_discount_account`, `default_provisional_account`,
  `purchase_expense_account`, `default_cogs_account`, `deferred_expense_account`,
  `deferred_revenue_account`.
- **Diwarisi ke Item:** saat Item dibuat memakai `item_group` ini dan **belum punya `item_defaults`**,
  sistem menyalin baris `item_group_defaults` → `item_defaults` Item (fungsi
  `update_defaults_from_item_group`). Urutan prioritas default: **Company → Brand → Item Group → Item**
  (nilai yang lebih spesifik menimpa).
- **Untuk klien non-akunting:** biarkan akun kosong. Cukup satu baris per company berisi `company`
  (dan bila perlu `default_warehouse`). Boleh **dikosongkan seluruhnya** — Item akan memakai default
  dari Company / fallback sistem.
- **Peringatan child table:** `save` memperlakukan child table sebagai **replace-all** — kirim seluruh
  baris `item_group_defaults` saat UPDATE (§4.4).

### 2.3 `taxes` — child table `Item Tax` (pajak default)

- Tabel child `taxes` (options **`Item Tax`**): `item_tax_template` (Link → Item Tax Template, `reqd`)
  + `tax_category` (opsional).
- **Diwarisi ke Item:** Item baru yang **belum punya `taxes`** akan menyalin baris `taxes` dari
  Item Group.
- **Validasi backend:** kombinasi `item_tax_template` + `tax_category` **tidak boleh duplikat** dalam
  satu Item Group (*"{0} entered twice {1} in Item Taxes"*).
- `Item Tax Template` adalah doctype terpisah (modul Accounts); `name`-nya = `{title} - {abbr}`.
  Ambil daftarnya via §5.4.

---

## 3. Autentikasi

Seluruh request di dokumen ini wajib menyertakan header:

```text
Authorization: Bearer <access_token>
```

Detail lengkap setup OAuth Client ada di **[`prd_oauth.md`](../prd_oauth.md)**.

---

## 4. CRUD — Doctype Item Group

### 4.1 CREATE — `frappe.client.insert`

**Langkah 0 — Pre-check (wajib sebelum CREATE)**

Karena `name = item_group_name` dan field ini `unique: 1`, duplikat terjadi bila nama sama. Cek
keberadaan lewat `frappe.client.get_list`:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Group",
    "fields": ["name"],
    "filters": [["item_group_name","=","Minuman"]],
    "limit_page_length": 1
  }'
```

**Hasil & aturan:**
- `message` kosong (`[]`) → lanjut ke CREATE.
- `message` terisi → blokir CREATE, tampilkan pesan: *"Item Group {item_group_name} sudah ada."*

> **Catatan backend:** tanpa pre-check, ERPNext tetap melempar `DuplicateEntryError` saat insert
> duplikat (field `unique`). Pre-check memberi pesan ramah & lebih cepat.

**Payload minimum (data wajib) — `frappe.client.insert`:**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item Group",
      "item_group_name": "Minuman"
    }
  }'
```

> `parent_item_group` kosong → otomatis menjadi root `All Item Groups`; `is_group` default `0`.

**Contoh request (lengkap — group daun dengan defaults + pajak):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item Group",
      "item_group_name": "Minuman",
      "parent_item_group": "Products",
      "is_group": 0,
      "item_group_defaults": [
        { "company": "PT Maju Jaya", "default_warehouse": "Toko Cikarang - PTMJ" }
      ],
      "taxes": [
        { "item_tax_template": "PPN 11% - PTMJ", "tax_category": "" }
      ]
    }
  }'
```

**Contoh respons sukses (HTTP 200):**

```json
{
  "message": {
    "name": "Minuman",
    "owner": "Administrator",
    "creation": "2026-09-01 09:15:00.000000",
    "modified": "2026-09-01 09:15:00.000000",
    "modified_by": "Administrator",
    "docstatus": 0,
    "idx": 0,
    "item_group_name": "Minuman",
    "parent_item_group": "Products",
    "is_group": 0,
    "lft": 6,
    "rgt": 7,
    "old_parent": "Products",
    "item_group_defaults": [
      { "name": "abc123", "company": "PT Maju Jaya", "default_warehouse": "Toko Cikarang - PTMJ" }
    ],
    "taxes": [
      { "name": "def456", "item_tax_template": "PPN 11% - PTMJ", "tax_category": "" }
    ]
  }
}
```

> `name` = `Minuman` → simpan nilai ini; dipakai untuk operasi berikutnya (dikirim di body).

> Variants berikut cukup dijadikan nilai **`doc`** pada `frappe.client.insert` di atas
> (contoh request lengkap), lalu baca `name` dari respons `message`.

**Varian A — Group node (induk tree):**

```json
{
  "doctype": "Item Group",
  "item_group_name": "Food & Beverage",
  "parent_item_group": "All Item Groups",
  "is_group": 1
}
```

> `is_group = 1` → node induk. Child group/Item mengacu ke sini via `parent_item_group`.

**Varian B — Group daun sederhana (tanpa defaults/pajak):**

```json
{
  "doctype": "Item Group",
  "item_group_name": "Makanan",
  "parent_item_group": "Food & Beverage",
  "is_group": 0
}
```

**Varian C — Group daun dengan `item_group_defaults` (multi company):**

```json
{
  "doctype": "Item Group",
  "item_group_name": "Elektronik",
  "parent_item_group": "Products",
  "is_group": 0,
  "item_group_defaults": [
    { "company": "PT Maju Jaya", "default_warehouse": "Gudang Pusat - PTMJ" },
    { "company": "PT Sukses Abadi", "default_warehouse": "Gudang Pusat - PSA" }
  ]
}
```

> Satu baris per company. Field `company` wajib di tiap baris. Item yang dibuat di group ini akan
> mewarisi default ini bila belum punya `item_defaults` sendiri (§2.2).

### 4.2 READ (satu record) & total count

**Langkah 1 — Ambil detail Item Group (`frappe.client.get`):**

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Group",
    "name": "Minuman"
  }'
```

Respons `message` berisi seluruh field Item Group (seperti respons CREATE), termasuk `lft`/`rgt`
dan child table `item_group_defaults`/`taxes`.

**Total count — `frappe.client.get_count`** (untuk pagination / lazy loading):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_count \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Group",
    "filters": [["is_group","=",0]]
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": 42
}
```

> Respons berupa `message` = jumlah record yang cocok. Nilai ini dipakai menghitung total halaman
> saat lazy loading di §4.3.

### 4.3 READ (daftar) — `frappe.client.get_list`

```bash
# Daftar — halaman 1, hanya group daun (leaf)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Group",
    "fields": ["name","is_group","parent_item_group"],
    "filters": [["is_group","=",0]],
    "order_by": "name asc",
    "limit_start": 0,
    "limit_page_length": 50
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "name": "Makanan", "is_group": 0, "parent_item_group": "Food & Beverage" },
    { "name": "Minuman", "is_group": 0, "parent_item_group": "Products" }
  ]
}
```

> Halaman berikutnya naikkan `limit_start` kelipatan `50`; `limit_page_length=0` = ambil **semua**
> record. Item Group **tidak punya filter `disabled`** — jangan sertakan `["disabled","=",0]`.

### 4.4 UPDATE — `frappe.client.save`

Update memakai `save`: kirim dokumen (hasil `frappe.client.get` yang dimodifikasi); `name` ada di
dalam body.

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.save \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item Group",
      "name": "Minuman",
      "item_group_name": "Minuman",
      "parent_item_group": "Products",
      "is_group": 0,
      "item_group_defaults": [
        { "company": "PT Maju Jaya", "default_warehouse": "Toko Cikarang - PTMJ" }
      ],
      "taxes": [
        { "item_tax_template": "PPN 11% - PTMJ", "tax_category": "" }
      ]
    }
  }'
```

**Contoh respons (HTTP 200):** objek `message` berisi dokumen terbaru (field berubah).

> **Catatan:**
> - `save` membangun ulang dokumen dari dict — kirim dokumen yang konsisten/lengkap (idealnya hasil
>   GET yang diubah). Child table `item_group_defaults` dan `taxes` berlaku **replace-all** — kirim
>   seluruh baris yang diinginkan.
> - Mengubah `parent_item_group` di sini otomatis memindahkan node di tree (lft/rgt dihitung ulang).
> - Mengubah `item_group_name` pada dokumen yang ada **tidak mengubah `name`** — bila ingin rename,
>   gunakan `frappe.client.rename_doc` (lihat §2.1 no. 1).
> - Jangan set `is_group` dari `1` ke `0` bila masih ada child (backend menolak, §2.1 no. 3).

**Perubahan kecil — `frappe.client.set_value`** (lebih aman untuk satu-dua field):

```bash
# Pindahkan group ke parent lain
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Group",
    "name": "Minuman",
    "fieldname": { "parent_item_group": "Food & Beverage" }
  }'
```

### 4.5 Hapus / non-aktif — **tidak dipakai**

**Aturan:** Item Group **tidak punya field `disabled`** dan data master **tidak boleh dihapus**.
Tidak ada operasi non-aktif (soft-delete).

`frappe.client.delete` **diblokir** backend bila:
- masih punya **child node** → `NestedSetChildExistsError`; atau
- **direferensikan** Item / doctype lain → `LinkExistsError`.

> ⚠️ **Jangan gunakan `frappe.client.delete`** — karena tanpa soft-delete, group yang tidak terpakai
> cukup **dibiarkan** (tidak dihapus, tidak dinon-aktifkan). Untuk master aktif, pastikan posisinya
> di tree sudah benar sejak awal (§4.1), karena koreksi posisi hanya lewat `parent_item_group`.

### 4.6 Menambah leaf baru di bawah leaf — ubah leaf menjadi group

**Aturan produk (penting):** hanya **group daun (`is_group=0`)** yang menampung Item yang
**boleh ditransaksikan** (deskripsi field `is_group`: *"Only leaf nodes are allowed in transaction"*).
Node group (`is_group=1`) hanya untuk organisasi tree. Karena leaf **tidak bisa punya child**, untuk
menambah leaf baru di bawah sebuah leaf, leaf tersebut **wajib diubah menjadi group** (`is_group=1`) —
dan semua Item di dalamnya harus dipindahkan ke leaf baru (atau leaf lain) agar tetap bisa
ditransaksikan.

> **Catatan backend:** aturan di atas adalah **konvensi/model ERPNext** yang dijaga lewat UI
> (dropdown parent dibatasi `is_group=1`, POS memperluas group → child leaf). Backend **tidak**
> memblokir keras item yang group-nya bukan leaf saat transaksi — jadi **frontend wajib menjaga**
> aturan ini: hanya leaf yang boleh dipilih sebagai `item_group` produk yang ditransaksikan.

**Alur lengkap (rekomendasi):**

**Langkah 1 — GET daftar Item di leaf** (untuk ditampilkan & dipilih target pemindahan):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","item_code"],
    "filters": [["item_group","=","Minuman"]],
    "limit_page_length": 0
  }'
```

> Endpoint ini juga didokumentasikan di **§5.7** beserta contoh respons. **Tanpa API ini, Langkah 4
> tidak bisa dijalankan** — hasil Langkah 1 adalah daftar Item yang harus dipindahkan ke leaf baru.

**Langkah 2 — Ubah leaf → group** (`is_group=1`):

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Group",
    "name": "Minuman",
    "fieldname": { "is_group": 1 }
  }'
```

**Langkah 3 — CREATE leaf baru** di bawah group:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.insert \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doc": {
      "doctype": "Item Group",
      "item_group_name": "Air Mineral",
      "parent_item_group": "Minuman",
      "is_group": 0
    }
  }'
```

**Langkah 4 — Pindahkan tiap Item ke leaf baru (manual per Item).** Frontend menampilkan daftar Item
hasil Langkah 1, user memilih target leaf untuk tiap Item (atau sekaligus), lalu update satu per satu:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.set_value \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "name": "Air Mineral 600ml",
    "fieldname": { "item_group": "Air Mineral" }
  }'
```

> Detail lengkap doctype `Item` (termasuk child table `item_defaults`, `taxes`, dsb.) ada di
> **[`prd_item.md`](./prd_item.md)** (folder yang sama).

**Langkah 5 — GET child / subtree** untuk verifikasi:

```bash
# (a) Child langsung (parent_item_group = ...)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Group",
    "fields": ["name","is_group","parent_item_group"],
    "filters": [["parent_item_group","=","Minuman"]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

```bash
# (b) Seluruh keturunan (subtree) via rentang lft/rgt
#     Ambil dulu lft/rgt parent: frappe.client.get { "doctype": "Item Group", "name": "Minuman" }
#     lalu query semua baris dengan lft >= parent.lft AND rgt <= parent.rgt:
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item Group",
    "fields": ["name","is_group","parent_item_group","lft","rgt"],
    "filters": [["lft",">=",4],["rgt","<=",11]],
    "order_by": "lft asc",
    "limit_page_length": 0
  }'
```

> **Catatan:**
> - Mengubah `is_group` dari `1` ke `0` pada node yang masih punya child **ditolak backend**
>   (*"{name} cannot be a leaf node as it has children"*, §6).
> - Selama transisi, Item masih menunjuk ke node yang kini group — pastikan pemindahan (Langkah 4)
>   selesai sebelum transaksi baru dibuat, agar Item tidak tertinggal di node group.

---

## 5. GET pendukung UI

### 5.1 GET node tree Item Group — `frappe.desk.treeview.get_children`

Untuk merender tree Item Group (dipakai juga oleh Tree view Desk). Endpoint generik Frappe
(Item Group tidak meng-override `get_children` sendiri):

```bash
curl -G "https://site-anda.com/api/method/frappe.desk.treeview.get_children" \
  -H 'Authorization: Bearer <access_token>' \
  --data-urlencode 'doctype=Item Group' \
  --data-urlencode 'parent=Products' \
  --data-urlencode 'include_disabled=false'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "value": "Makanan", "title": "Makanan", "expandable": 0 },
    { "value": "Minuman", "title": "Minuman", "expandable": 0 }
  ]
}
```

> - `expandable: 1` = node group (`is_group=1`).
> - `parent` kosong → mengambil node paling atas (root `All Item Groups`). Lalu panggil dengan
>   `parent=<nama node>` untuk child-nya (lazy loading per level).
> - `include_disabled` tidak berpengaruh di sini (Item Group tidak punya kolom `disabled`).

### 5.2 GET Item Group (dropdown leaf) — `frappe.client.get_list`

Untuk dropdown UI yang hanya boleh memilih group daun — mis. mengisi `item_groups` di POS Profile
atau `item_group` saat membuat Item:

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

> Kirim `name` ke field `item_group` (Item) / baris `item_groups[].item_group` (POS Profile).
> Hanya node daun yang menampung Item yang **boleh ditransaksikan** (deskripsi field `is_group`).
> Saat sebuah leaf dipecah menjadi group, Item di dalamnya harus dipindahkan ke leaf baru — lihat §4.6.

### 5.3 GET Company — dropdown `item_group_defaults`

Untuk memilih company per baris `item_group_defaults`:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Company",
    "fields": ["name","abbr"],
    "filters": [["is_group","=",0]],
    "limit_page_length": 0
  }'
```

### 5.4 GET Item Tax Template — dropdown `taxes`

Untuk field `item_tax_template` pada child table `taxes`:

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

### 5.5 GET Warehouse / Cost Center / Account — dropdown `item_group_defaults`

Untuk field link di dalam baris `item_group_defaults` (dibatasi per `company` baris):

```bash
# Warehouse (default_warehouse)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Warehouse",
    "fields": ["name"],
    "filters": [["company","=","PT Maju Jaya"],["disabled","=",0],["is_group","=",0]],
    "limit_page_length": 0
  }'

# Cost Center (buying/selling_cost_center)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Cost Center",
    "fields": ["name"],
    "filters": [["company","=","PT Maju Jaya"],["disabled","=",0],["is_group","=",0]],
    "limit_page_length": 0
  }'

# Account (income/expense_account, dst. sesuai tipe)
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Account",
    "fields": ["name"],
    "filters": [["company","=","PT Maju Jaya"],["is_group","=",0]],
    "limit_page_length": 0
  }'
```

> Detail Warehouse: **[prd_warehouse.md](../setup/prd_warehouse.md)**.

### 5.6 GET resolve default dari Company (opsional) — `get_company_resolved_defaults`

Untuk **prefill** baris `item_group_defaults` saat membuat group baru — ambil nilai default yang
sudah ter-resolve dari Company (bukan menyimpan, hanya membaca):

```bash
curl -X POST https://site-anda.com/api/method/erpnext.setup.doctype.item_group.item_group.get_company_resolved_defaults \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "company": "PT Maju Jaya"
  }'
```

> Endpoint whitelisted bawaan Item Group. Berguna bila frontend ingin menampilkan nilai awal
> `default_warehouse`, cost center, akun, dst. yang berasal dari Company sebelum user mengedit.

### 5.7 GET Item dalam leaf / group — `frappe.client.get_list` (doctype Item)

Ambil **seluruh Item yang `item_group`-nya = group/leaf tertentu**. Endpoint ini yang dipakai pada
**§4.6 Langkah 1** untuk menampilkan Item yang harus dipindahkan saat sebuah leaf dipecah menjadi
group:

```bash
curl -X POST https://site-anda.com/api/method/frappe.client.get_list \
  -H 'Authorization: Bearer <access_token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "doctype": "Item",
    "fields": ["name","item_name","item_code","stock_uom","disabled"],
    "filters": [["item_group","=","Minuman"]],
    "order_by": "name asc",
    "limit_page_length": 0
  }'
```

**Contoh respons (HTTP 200):**

```json
{
  "message": [
    { "name": "Air Mineral 600ml", "item_name": "Air Mineral 600ml", "item_code": "Air Mineral 600ml", "stock_uom": "Nos", "disabled": 0 },
    { "name": "Air Mineral 1500ml", "item_name": "Air Mineral 1500ml", "item_code": "Air Mineral 1500ml", "stock_uom": "Nos", "disabled": 0 }
  ]
}
```

> - `filters` memakai **`name` Item Group** (`Minuman`), sama dengan nilai `item_group` pada Item.
> - **Tanpa API ini, Langkah 4 di §4.6 tidak bisa dijalankan**: frontend harus memanggil endpoint ini
>   dulu (Langkah 1) untuk mendapat daftar Item, lalu user memilih target leaf, baru `set_value` per
>   Item (Langkah 4).
> - Untuk list Item yang besar, kombinasikan dengan `frappe.client.get_count` (doctype `Item`, filter
>   sama) untuk pagination — pola sama seperti §4.2.

---

## 6. Penanganan error umum

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa | `{"message": "Not permitted"}` |
| 403 | Role tidak punya akses (bukan `Item Manager` utk tulis) | `{"message": "Not permitted"}` |
| 404 | Resource tidak ditemukan | `{"exc_type":"DoesNotExistError","message":"Resource Not Found"}` |
| 417 | Field wajib kosong (`reqd`) | `{"exc_type":"MandatoryError","message":"item_group_name is mandatory"}` |
| 417 | Duplikat nama (`unique`) | `{"exc_type":"DuplicateEntryError","message":"... already exists"}` |
| 417 | Parent invalid / loop tree | `{"exc_type":"NestedSetRecursionError","message":"Item cannot be added to its own descendants"}` |
| 417 | Delete diblokir (masih ada child) | `{"exc_type":"NestedSetChildExistsError","message":"Cannot delete Minuman as it has child nodes"}` |
| 417 | Delete diblokir (di-referensikan Item) | `{"exc_type":"LinkExistsError","message":"Cannot delete because of linked records"}` |
| 417 | Leaf node punya child | `{"exc_type":"ValidationError","message":"Minuman cannot be a leaf node as it has children"}` |
| 417 | `taxes` duplikat (template + kategori) | `{"exc_type":"ValidationError","message":"... entered twice ... in Item Taxes"}` |

> **Catatan:** karena seluruh pemanggilan memakai `/api/method/...`, hasil sukses dibungkus di
> `message` (bukan `data`). Body error tetap berbentuk `{ "exc_type": ..., "exception": ..., "message": ... }`.
> Method `frappe.client.get` dengan `name` yang tidak ada → 404 `DoesNotExistError`.

---

## 7. Koleksi Postman

Seluruh pemanggilan di atas tersedia dalam **satu koleksi Postman** yang siap import:
`docs/postman/postman_erpnext_api.json` (Collection v2.1). Koleksi ini berisi seluruh modul API
ERPNext (OAuth 2.0 + Supplier + Customer + Contact + Address + Lead + Employee + User + Warehouse
+ POS Profile + **Item Group**).

Folder **`10. Item Group`** berisi **24 request** yang mencakup:
- Pre-check & CREATE (minimum, lengkap, group node, leaf, multi-company) — `10.1a`–`10.4`
- READ single / count / list — `10.5`–`10.7`
- UPDATE (`save`) & `set_value` (pindah parent, leaf → group) — `10.8`–`10.10`
- Alur pemecahan leaf (§4.6): GET Items, pindah Item, GET child / subtree — `10.11`–`10.14`
- GET pendukung UI: tree, dropdown leaf, Company, Item Tax Template, Warehouse / Cost Center /
  Account, resolve default — `10.15`–`10.22`

**Variabel yang perlu diisi** (Collection Variables):
- `item_group_id` / `item_group_name` — name hasil CREATE (mis. `Minuman`)
- `item_group_parent` — parent group node (mis. `Food & Beverage`)
- `item_group_child` — leaf baru di bawah group (mis. `Air Mineral`)
- `item_code` — Item yang dipindah ke leaf baru (mis. `Air Mineral 600ml`)
- `item_tax_template` — template pajak utk child `taxes` (mis. `PPN 11% - PTMJ`)

Cara pakai sama dengan folder lain: isi variabel di atas, jalankan folder `0. OAuth 2.0` (atau
Get New Access Token), lalu jalankan request pada folder `10. Item Group`. Request `10.1`, `10.1b`,
`10.2`, `10.3`, `10.4` otomatis menyimpan `name` hasil CREATE ke variabel `item_group_id` /
`item_group_parent` / `item_group_child` lewat test script.
