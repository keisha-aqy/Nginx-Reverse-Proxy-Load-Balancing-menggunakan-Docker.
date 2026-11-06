# Sistem Informasi Mahasiswa dengan Nginx Load Balancer

Implementasi **Load Balancer Nginx** menggunakan **Docker** dan **Docker Compose** dengan dua container backend (**POD 1** dan **POD 2**) yang menjalankan aplikasi Flask yang sama. Request dari pengguna dibagi secara bergantian memakai metode **Round Robin**.

> Istilah "POD" di project ini hanyalah nama untuk dua container backend Docker, **bukan** Kubernetes Pod.

## Fitur

- Dashboard dan daftar mahasiswa (NIM, nama, jurusan, fakultas)
- Pencarian berdasarkan NIM atau nama, dengan highlight hasil
- Filter fakultas dan pengurutan data
- Halaman detail mahasiswa
- Identitas POD yang melayani request ditampilkan di website (badge di atas, kotak "Informasi Server" di bawah)
- Riwayat request dan tombol **Tes 10 request** untuk membuktikan Round Robin langsung dari browser
- Mode terang/gelap dan tampilan responsif untuk ponsel
- Dua POD berbagi satu database SQLite sehingga data selalu konsisten

## Arsitektur

```text
 USER / BROWSER
       │
       ▼  http://localhost:8080
 ┌─────────────────┐
 │  NGINX (Load    │
 │  Balancer)      │
 └───────┬─────────┘
   ┌─────┴──────┐
   ▼            ▼
┌────────┐  ┌────────┐
│ POD 1  │  │ POD 2  │   ← aplikasi Flask yang SAMA
└───┬────┘  └───┬────┘
    └─────┬─────┘
          ▼
   SQLite (volume ./data)
```

| Komponen | Fungsi |
|---|---|
| **Nginx** | Menerima request di port 8080 dan meneruskannya ke POD 1 / POD 2 secara bergantian |
| **POD 1 / POD 2** | Container Flask dari image yang sama, dibedakan oleh environment variable `POD_NAME` |
| **SQLite** | Menyimpan data mahasiswa, dibagikan ke kedua POD lewat volume Docker |
| **Docker Compose** | Menjalankan ketiga container dengan satu command |

## Cara Kerja Identifikasi POD

1. `docker-compose.yml` memberi `POD_NAME=POD 1` pada container pertama dan `POD_NAME=POD 2` pada container kedua.
2. Kode Flask yang sama membaca nilai tersebut dengan `os.environ`.
3. Setiap response API menyertakan field `pod` dan `status`, serta header HTTP `X-Served-By`.
4. Frontend hanya menampilkan `data.pod` dari response, tidak menentukan nama POD sendiri.

Nginx juga menambahkan header `X-Upstream` yang berisi alamat IP container yang dipilih, sebagai bukti kedua.

## Teknologi

- HTML, CSS, JavaScript (frontend)
- Python Flask (backend)
- SQLite (database)
- Nginx (load balancer)
- Docker dan Docker Compose

## Struktur Folder

```text
student-lb/
├── docker-compose.yml        # menjalankan Nginx + POD 1 + POD 2
├── nginx/
│   └── nginx.conf            # konfigurasi load balancer (Round Robin)
├── app/                      # satu aplikasi untuk POD 1 dan POD 2
│   ├── Dockerfile            # resep image aplikasi
│   ├── requirements.txt      # library Python
│   ├── app.py                # backend Flask, identitas POD, database
│   └── static/
│       ├── index.html        # dashboard
│       ├── detail.html       # halaman detail mahasiswa
│       ├── style.css         # tampilan
│       └── script.js         # logika frontend
└── data/                     # dibuat otomatis, berisi students.db
```

## Prasyarat

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Windows/macOS) atau Docker Engine (Linux)
- Docker Compose v2 (sudah termasuk di Docker Desktop)

Cek instalasi:

```bash
docker --version
docker compose version
```

Python **tidak perlu** diinstal di komputer Anda karena aplikasi berjalan di dalam container.

## Menjalankan Project

```bash
git clone https://github.com/keisha-aqy/Nginx-Reverse-Proxy-Load-Balancing-menggunakan-Docker.git
cd Nginx-Reverse-Proxy-Load-Balancing-menggunakan-Docker
docker compose up --build -d
```

Cek status container:

```bash
docker compose ps
```

Tiga container (`pod1`, `pod2`, `load-balancer`) harus berstatus **Up**. Lalu buka **http://localhost:8080**.

## Endpoint

| Endpoint | Keterangan |
|---|---|
| `/` | Dashboard |
| `/detail?nim=20260001` | Halaman detail mahasiswa |
| `/api/info` | Info server (JSON: website, pod, status) |
| `/api/students` | Daftar mahasiswa (JSON), mendukung `?q=kata` |
| `/api/students/<nim>` | Detail satu mahasiswa (JSON) |
| `/whoami` | Teks polos berisi nama POD, untuk pengujian terminal |

## Pengujian Round Robin

### Lewat terminal

**Linux / macOS / Git Bash:**

```bash
for i in 1 2 3 4 5 6; do echo "Request $i: $(curl -s http://localhost:8080/whoami)"; done
```

**Windows PowerShell:**

```powershell
1..6 | ForEach-Object { "Request $_`: " + (curl.exe -s http://localhost:8080/whoami) }
```

Penjelasan singkat: perintah mengulang 6 kali, tiap pengulangan mengirim satu request HTTP dengan `curl` ke Load Balancer (`-s` menyembunyikan progress bar). Di PowerShell gunakan `curl.exe`, karena `curl` adalah alias command lain.

Hasil yang diharapkan:

```text
Request 1: Request ini dilayani oleh POD 1
Request 2: Request ini dilayani oleh POD 2
Request 3: Request ini dilayani oleh POD 1
Request 4: Request ini dilayani oleh POD 2
Request 5: Request ini dilayani oleh POD 1
Request 6: Request ini dilayani oleh POD 2
```

Request pertama bisa saja dimulai dari POD 2. Yang penting polanya bergantian.

### Bukti lewat header HTTP

```bash
curl -si http://localhost:8080/whoami | grep -i -E "x-served-by|x-upstream"
```

PowerShell:

```powershell
curl.exe -si http://localhost:8080/whoami | Select-String -Pattern "x-served-by|x-upstream"
```

### Lewat browser

Buka http://localhost:8080, lalu klik **Cek server lagi** atau **Tes 10 request**. Kotak Informasi Server berganti **biru (POD 1)** dan **hijau (POD 2)**.

> Saat halaman dimuat, browser mengirim beberapa request sekaligus (HTML, CSS, JS, API) dan semuanya ikut mengambil giliran. Karena itu pergantian lewat refresh tidak selalu rapi satu per satu. Tombol dan terminal memberi hasil paling presisi.

## Pengujian Saat Salah Satu Container Mati

```bash
docker compose stop pod1
```

Jalankan lagi pengujian terminal. Semua request seharusnya dilayani **POD 2** dan website tetap bisa diakses. Hidupkan kembali:

```bash
docker compose start pod1
```

Pola bergantian akan kembali. Untuk melihat tampilan error, matikan keduanya dengan `docker compose stop pod1 pod2`, lalu hidupkan lagi dengan `docker compose start pod1 pod2`.

## Perintah Berguna

| Perintah | Fungsi |
|---|---|
| `docker compose up --build -d` | Build image dan jalankan semua container di latar belakang |
| `docker compose ps` | Lihat status container |
| `docker compose logs -f` | Lihat log semua container |
| `docker compose stop pod1` | Hentikan POD 1 |
| `docker compose start pod1` | Jalankan kembali POD 1 |
| `docker compose down` | Hentikan dan hapus container |

Setelah mengubah file di `app/`, jalankan ulang `docker compose up --build -d`, lalu tekan **Ctrl + F5** di browser.

## Pemecahan Masalah

| Masalah | Solusi |
|---|---|
| `the Dockerfile cannot be empty` | `app/Dockerfile` kosong atau belum disimpan. Isi dan simpan filenya, pastikan namanya bukan `Dockerfile.txt` |
| `port is already allocated` | Port 8080 sudah dipakai. Ubah `8080:80` di `docker-compose.yml`, misalnya menjadi `8081:80` |
| `cannot connect to the Docker daemon` | Docker Desktop belum berjalan. Buka dan tunggu hingga Engine running |
| `docker compose ps` kosong | Container belum berjalan. Jalankan `docker compose up --build -d` dan baca pesan errornya |
| Tampilan tidak berubah setelah edit | Build ulang image, lalu Ctrl + F5 di browser |
| `NIM tidak diberikan` | Halaman `/detail` dibuka tanpa `?nim=...`. Masuk lewat tombol **Detail** di tabel |
| Pylance: `Import "flask" could not be resolved` | Hanya peringatan editor. Flask terpasang di container. Abaikan atau jalankan `pip install flask` secara lokal |

## Catatan

- Project ini untuk **pembelajaran**. Flask dijalankan dengan server bawaan, bukan server produksi (misalnya Gunicorn).
- SQLite pada volume bersama cocok untuk demo. Untuk beban tulis yang besar, gunakan database server seperti PostgreSQL atau MySQL.
- Folder `data/` berisi file database hasil runtime. Disarankan menambahkannya ke `.gitignore`:

```text
data/
__pycache__/
*.pyc
```

## Lisensi

Tambahkan lisensi sesuai kebutuhan, misalnya MIT.
