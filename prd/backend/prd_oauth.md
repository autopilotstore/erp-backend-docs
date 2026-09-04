# PRD — Autentikasi OAuth 2.0 (ERPNext / Frappe)

> Dokumen ini **dipakai bersama** oleh seluruh dokumen PRD REST API modul ERPNext
> (Supplier, Buying, Selling, dst.). Detail pembuatan OAuth Client dan alur token
> cukup ditulis **sekali di sini**; dokumen PRD modul lain cukup merujuk file ini.

- **Versi API:** `/api/method/...` (API v1)
- **Grant type yang didukung:** `authorization_code`, `refresh_token`, `password`
- **Grant type yang TIDAK didukung:** `client_credentials`
- **Format body:** `application/x-www-form-urlencoded`

---

## 1. Konsep singkat

Setiap request REST API ERPNext memerlukan header:

```text
Authorization: Bearer <access_token>
```

`access_token` diperoleh lewat alur OAuth 2.0 **Authorization Code**:

1. User diarahkan ke halaman otorisasi Frappe (login + setujui aplikasi).
2. Browser di-redirect kembali dengan `code`.
3. Aplikasi menukar `code` → `access_token` + `refresh_token`.
4. `access_token` (masa berlaku ±1 jam / `expires_in: 3600`) dipakai untuk request.
5. Saat kedaluwarsa, `refresh_token` dipakai untuk mendapat token baru tanpa login ulang.

---

## 2. Buat OAuth Client (sekali saja)

1. Buka **Integrations → OAuth Client → New** di site ERPNext.
2. Isi:

   | Field | Nilai |
   |---|---|
   | `app_name` | mis. `MyApp` |
   | `scopes` | `all openid` (default) |
   | `grant_type` | **Authorization Code** |
   | `response_type` | **Code** |
   | `redirect_uris` | `https://myapp.example.com/callback` (satu URI per baris) |
   | `default_redirect_uri` | `https://myapp.example.com/callback` |
   | `skip_authorization` | centang bila tidak ingin halaman konfirmasi |

3. Simpan → catat **`client_id`** dan **`client_secret`**.

> **Penting:** Frappe **tidak mendukung** grant type `client_credentials`.
> Validator (`frappe/oauth.py`) hanya mengizinkan `authorization_code`, `refresh_token`, dan `password`.

---

## 3. Alur mendapatkan token

### 3.1 Langkah 1 — Arahkan user ke halaman otorisasi (browser)

```text
GET https://site-anda.com/api/method/frappe.integrations.oauth2.authorize?client_id=<client_id>&redirect_uri=https://myapp.example.com/callback&response_type=code&scope=all&state=<random_state>
```

- Jika user belum login → diarahkan ke `/login`, lalu kembali.
- Setelah user menyetujui (kecuali `skip_authorization`), browser di-redirect ke:
  `https://myapp.example.com/callback?code=<AUTHORIZATION_CODE>&state=<random_state>`
- `state` dipakai untuk mencegah CSRF — aplikasi wajib memverifikasinya.

### 3.2 Langkah 2 — Tukar `code` menjadi `access_token` + `refresh_token`

```bash
curl -X POST https://site-anda.com/api/method/frappe.integrations.oauth2.get_token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=authorization_code' \
  -d 'code=<AUTHORIZATION_CODE>' \
  -d 'redirect_uri=https://myapp.example.com/callback' \
  -d 'client_id=<client_id>' \
  -d 'client_secret=<client_secret>'
```

**Contoh respons sukses (token):**

```json
{
  "access_token": "b5c4f6a1d9e2...",
  "refresh_token": "7a8b9c0d1e2f...",
  "expires_in": 3600,
  "token_type": "Bearer",
  "scope": "all"
}
```

### 3.3 Langkah 3 — Gunakan token di setiap pemanggilan CRUD

```bash
-H 'Authorization: Bearer <access_token>'
```

### 3.4 Langkah 4 — Perbarui token (saat `access_token` kedaluwarsa)

```bash
curl -X POST https://site-anda.com/api/method/frappe.integrations.oauth2.get_token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=refresh_token' \
  -d 'refresh_token=<refresh_token>' \
  -d 'client_id=<client_id>' \
  -d 'client_secret=<client_secret>'
```

### 3.5 Langkah 5 — (Opsional) Cabut token

```bash
curl -X POST https://site-anda.com/api/method/frappe.integrations.oauth2.revoke_token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'token=<access_token>' \
  -d 'token_type_hint=access_token'
```

---

## 4. Penanganan error autentikasi

| Kode | Kondisi | Contoh body |
|---|---|---|
| 401 | Token tidak valid / kedaluwarsa / tidak dikirim | `{"message": "Not permitted"}` |
| 403 | Role user tidak punya akses ke resource | `{"message": "Not permitted"}` |
| 400 | `grant_type` / parameter token salah | `{"message": "..."}` |

> **Strategi umum di aplikasi:** saat menerima **401**, otomatis panggil ulang `get_token`
> dengan `grant_type=refresh_token`, lalu ulangi request yang gagal.
