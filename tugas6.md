# Laporan Tugas 6 — Endpoint Collection yang Dapat Dicari, Dipaginasikan, dan Disortir

## 1. Tujuan
Menerapkan fungsionalitas lanjutan pada endpoint collection dari resource `books` agar mendukung:
- Paginasi data dinamis (`page` dan `per_page` dengan batas maksimum 50).
- Pencarian data (*search*) pada dua field utama (`title` dan `isbn`).
- Pengurutan data (*sorting*) terkontrol menggunakan *allowlist*.
- Validasi query parameter (*query validation*) yang mengembalikan status `422 Unprocessable Content` jika input invalid.
- Response collection berbasis `BookResource` yang memuat `data`, `links`, dan `meta`.

---

## 2. Endpoint dan Query Contract
- **Endpoint Utama**: `GET http://127.0.0.1:8000/api/books`
- **Query Contract Parameter**:
  - `search`: Kata kunci pencarian berdasarkan `title` atau `isbn` (string, max 100 karakter, opsional).
  - `sort`: Field acuan pengurutan data (`id`, `title`, `isbn`, `created_at`). Default: `id`.
  - `direction`: Arah pengurutan data (`asc` atau `desc`). Default: `asc`.
  - `page`: Nomor halaman yang diakses (integer, min 1). Default: 1.
  - `per_page`: Jumlah data per halaman (integer, min 1, max 50). Default: 5.

---

## 3. Daftar Field Search dan Sort Allowlist
- **Field Search**:
  1. `title` (pencarian teks pada judul buku)
  2. `isbn` (pencarian teks pada nomor ISBN buku)
- **Sort Allowlist**:
  - `id`
  - `title`
  - `isbn`
  - `created_at`

---

## 4. Validation Rules
Aturan validasi yang diterapkan pada method `index()` controller:
```php
$validated =$request->validate([
    'search' => ['sometimes', 'nullable', 'string', 'max:100'],
    'sort' => ['sometimes', Rule::in(['id', 'title', 'isbn', 'created_at'])],
    'direction' => ['sometimes', Rule::in(['asc', 'desc'])],
    'page' => ['sometimes', 'integer', 'min:1'],
    'per_page' => ['sometimes', 'integer', 'min:1', 'max:50'],
]);
```

---

## 5. Implementasi Query Search dan Sorting
Implementasi logika query pada controller:

### 5.1. Filter Pencarian
Jika parameter `search` diberikan, query akan memfilter berdasarkan `title` atau `isbn`:
```php
if (! empty($validated['search'])) {
    $search = $validated['search'];

    $query->where(function ($builder) use ($search) {
        $builder
            ->where('title', 'like', "%{$search}%")
            ->orWhere('isbn', 'like', "%{$search}%");
    });
}
```

### 5.2. Pengurutan dan Paginasi
Parameter `sort`, `direction`, `page`, dan `per_page` diterapkan sebagai berikut:
```php
$sort = $validated['sort'] ?? 'id';
$direction = $validated['direction'] ?? 'asc';
$perPage = $validated['per_page'] ?? 5;

$books = $query
    ->orderBy($sort, $direction)
    ->paginate($perPage)
    ->withQueryString();
```

---

## 6. Sample Response

### 6.1. Request Contoh
GET http://127.0.0.1:8000/api/books?search=Code&sort=title&direction=asc&per_page=2&page=1
```json
{
    "data": [
        {
            "id": 1,
            "title": "Clean Code",
            "isbn": "9780132350884",
            "available": true,
            "created_at": "2026-10-08T07:05:47.000000Z"
        }
    ],
    "links": {
        "first": "http://127.0.0.1:8000/api/books?search=Code&sort=title&direction=asc&per_page=2&page=1",
        "last": "http://127.0.0.1:8000/api/books?search=Code&sort=title&direction=asc&per_page=2&page=1",
        "prev": null,
        "next": null
    },
    "meta": {
        "current_page": 1,
        "from": 1,
        "last_page": 1,
        "links": [
            {
                "url": null,
                "label": "&laquo; Previous",
                "page": null,
                "active": false
            },
            {
                "url": "http://127.0.0.1:8000/api/books?search=Code&sort=title&direction=asc&per_page=2&page=1",
                "label": "1",
                "page": 1,
                "active": true
            },
            {
                "url": null,
                "label": "Next &raquo;",
                "page": null,
                "active": false
            }
        ],
        "path": "http://127.0.0.1:8000/api/books",
        "per_page": 2,
        "to": 1,
        "total": 1
    }
}
### 6.2. Response JSON (Status `200 OK`)
GET http://127.0.0.1:8000/api/books?search=Code&sort=title&direction=asc&per_page=2&page=1
```json
{
    "data": [
        {
            "id": 1,
            "title": "Clean Code",
            "isbn": "9780132350884",
            "available": true,
            "created_at": "2026-10-08T07:05:47.000000Z"
        }
    ],
    "links": {
        "first": "http://127.0.0.1:8000/api/books?search=Code&sort=title&direction=asc&per_page=2&page=1",
        "last": "http://127.0.0.1:8000/api/books?search=Code&sort=title&direction=asc&per_page=2&page=1",
        "prev": null,
        "next": null
    },
    "meta": {
        "current_page": 1,
        "from": 1,
        "last_page": 1,
        "links": [
            {
                "url": null,
                "label": "&laquo; Previous",
                "page": null,
                "active": false
            },
            {
                "url": "http://127.0.0.1:8000/api/books?search=Code&sort=title&direction=asc&per_page=2&page=1",
                "label": "1",
                "page": 1,
                "active": true
            },
            {
                "url": null,
                "label": "Next &raquo;",
                "page": null,
                "active": false
            }
        ],
        "path": "http://127.0.0.1:8000/api/books",
        "per_page": 2,
        "to": 1,
        "total": 1
    }
}
```

### 6.3. Response JSON (Status `422 Unprocessable Content`)
Terjadi jika parameter `sort` memiliki nilai yang tidak ada dalam *allowlist*:
```http
GET http://127.0.0.1:8000/api/books?sort=invalid_field
```
```json
{
    "message": "The selected sort is invalid.",
    "errors": {
        "sort": [
            "The selected sort is invalid."
        ]
    }
}
```

---

## 7. Kesimpulan
Fungsionalitas pencarian, pengurutan, dan paginasi pada endpoint `GET /api/books` berhasil diterapkan menggunakan kombinasi parameter query dan builder pattern. Validasi input memastikan robustness terhadap parameter yang tidak valid, sementara format response berbasis API Resource memastikan konsistensi data untuk klien API.