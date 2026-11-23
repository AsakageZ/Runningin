# Motion Runner — Sekolah Telkom

Endless runner 3D bertema SMK Telkom yang dikendalikan **gerakan badan** lewat webcam. Lompat, roll, dan pindah lajur dengan menggerakkan tubuh di depan kamera — atau pakai keyboard kalau kamera tidak tersedia.

Semua ada dalam satu file: `index.html`. Tanpa build step, tanpa `npm install`, tanpa aset gambar (semua model 3D, tekstur, dan suara dibuat secara prosedural di browser).

---

## Cara menjalankan di laptop

> **Wajib lewat server lokal.** Membuka `index.html` langsung dengan klik dua kali (`file://`) **tidak bisa** — browser memblokir ES module dari CDN dan kamera tidak diizinkan di `file://`. Jalankan salah satu cara di bawah, lalu buka `http://localhost:8000`.

### 1. Python (paling praktis, hampir semua laptop sudah ada)

```bash
# masuk ke folder project
cd motion-runner-telkom17

# Python 3
python -m http.server 8000

# kalau perintah di atas tidak dikenali, coba:
python3 -m http.server 8000
```

Buka <http://localhost:8000> di Chrome / Edge / Firefox.

### 2. Node.js

```bash
npx serve .
# atau
npx http-server -p 8000
```

### 3. VS Code — ekstensi Live Server

Klik kanan `index.html` → **Open with Live Server**. Otomatis terbuka di browser.

### 4. Tanpa install apa pun (GitHub Pages)

Kalau repo ini diaktifkan GitHub Pages, game bisa dimainkan langsung online lewat HTTPS (kamera butuh HTTPS atau localhost, jadi ini aman).

**Matikan server:** tekan `Ctrl + C` di terminal.

**Butuh internet.** Model pose (`pose_landmarker_lite`) dan library three.js diambil dari CDN saat halaman dibuka (± 3–5 MB pada load pertama, setelah itu ter-cache browser).

---

## Kontrol

| Aksi | Kamera (gerak badan) | Keyboard |
|---|---|---|
| **Lompat** | Angkat badan / bahu naik | `↑` / `W` / `Spasi` |
| **Roll (jongkok)** | Turunkan badan / jongkok | `↓` / `S` / `Panah Bawah` |
| **Pindah lajur** | Miringkan badan ke kiri/kanan | `←` `→` / `A` `D` |
| **Kalibrasi ulang** | Tekan `C` | — |

Berdiri sekitar **2 meter** dari kamera, posisikan kepala dan bahu masuk frame. Saat mulai, game minta kamu berdiri diam **2,5 detik** untuk kalibrasi posisi netral.

Di menu juga ada:
- **Kesulitan** — Mudah / Normal / Sulit / Ekstrem
- **Waktu** — Otomatis / Pagi / Siang / Sore / Malam
- **Cuaca** — Otomatis / Cerah / Berawan / Gerimis / Angin
- **Sensitivitas** — slider untuk lompat, jongkok, dan miring (tersimpan otomatis)

Semua pengaturan, rekor terbaik, dan leaderboard disimpan di `localStorage` browser (per-browser, tidak sinkron antar perangkat).

---

## Cara kerja

### 1. Deteksi gerakan

Frame webcam diproses model estimasi pose, mencoba berurutan sampai satu berhasil:

1. **MediaPipe Tasks Vision 0.10.14** — `pose_landmarker_lite`, mode VIDEO, hingga 5 orang
2. MediaPipe versi alternatif (0.10.3)
3. **MoveNet MultiPose Lightning** (TensorFlow.js 4.22) sebagai fallback

Hanya 3 titik yang dipakai: hidung, bahu kiri, bahu kanan. Dari situ diturunkan:

- `sw` — lebar bahu (dipakai sebagai satuan skala, jadi jarak ke kamera tidak masalah)
- `hx` — posisi horizontal (60 % hidung + 40 % tengah bahu) → untuk lajur
- `cy` — posisi vertikal → untuk lompat/roll

**Semua ambang batas diukur dalam satuan lebar bahu**, bukan piksel. Efeknya: orang tinggi/pendek atau maju/mundur tetap terpicu dengan gerakan yang sama.

### 2. Filter & prediksi

Posisi mentah dihaluskan dengan **One Euro filter** — saat badan diam hasilnya stabil, saat gerakan cepat lag-nya hampir nol. Lalu ditambah kompensasi latensi (`latencyComp`, 70 ms) memakai kecepatan sinyal, supaya lompatan terasa responsif dan tidak "telat satu frame".

Setelah kalibrasi, pose netral dipakai sebagai titik nol, dan **bisa bergeser pelan** (drift) hanya kalau kamu sendirian di frame dan benar-benar diam — jadi badan tidak perlu kembali ke posisi semula persis.

### 3. Pemilihan pemain di antara kerumunan

Kalau ada beberapa orang terdeteksi (misal teman lewat di belakang), game **mengunci satu pemain**: saat kalibrasi dipilih orang terdekat-di-tengah, saat bermain dipilih yang skala bahu dan posisinya paling cocok dengan baseline. Orang lain digambar garis merah tipis dan diabaikan.

### 4. Dunia 3D

Scene dibangun dengan **three.js 0.160**. Enam zona bergiliran membentuk lintasan tak berujung:

| # | Zona |
|---|---|
| 1 | Koridor kantin |
| 2 | Gang samping gedung |
| 3 | Lorong utama · rak sepatu |
| 4 | Lobi piala penghargaan |
| 5 | Koridor kursi tunggu |
| 6 | Halaman · mural canteen |

Di ujung tiap zona ada **belokan 90°** gaya Temple Run: dunia diputar mengelilingi titik pusat radius 6 unit, lalu zona berikutnya digeser masuk. Hanya 3 zona aktif sekaligus (sekarang, sebelumnya, berikutnya) supaya ringan.

Semua tekstur (keramik, paving, beton, rumput, seng, mural, kaca) dan prop (meja, kursi, rak sepatu, bangku tunggu, lemari piala, gerobak, pohon, motor) digambar lewat `canvas` / geometri prosedural, lalu **di-batch** per zona (satu mesh per material) untuk menekan jumlah draw call.

### 5. Rintangan

| Rintangan | Aturan |
|---|---|
| **Meja / kursi sekolah** (rendah) | Harus **lompat** (y < 0,8 = kena) |
| **Papan pengumuman** (tinggi) | Harus **roll** — kalau tidak jongkok, kena |
| **Tumpukan gerobak** (panjang) | Tidak bisa dilompati/di-roll, harus **ganti lajur** |

Jumlah dan kerapatan rintangan mengikuti tingkat kesulitan (`obsMin`–`obsMax`), termasuk peluang 1–3 lajur terblokir dalam satu baris.

### 6. Koin & item istimewa

Koin muncul di jalur aman di antara rintangan (dan kadang melengkung di atas rintangan rendah). Ada 3 item istimewa yang muncul acak:

| Item | Efek |
|---|---|
| 🛡️ **Tameng** | Kebal semua rintangan |
| 🧲 **Magnet** | Menarik koin di sekitar |
| **2x** | Nilai koin jadi dua |

Untuk **klaim** item, game dijeda dan menampilkan kuis tentang Telkom School. Jawaban benar → efek aktif 10 detik. Jawaban salah → **hukuman ngebut** (kecepatan ×1,55) selama 10 detik. Kuis bisa dijawab dengan klik, angka `1`–`4`, atau huruf `A`–`D`.

### 7. Suasana & efek

Siklus siang–malam penuh berjalan tiap 2000 unit jarak: warna langit, kekuatan matahari, kabut, lampu koridor, jendela gedung, dan bintang semuanya diinterpolasi dari keyframe. Cuaca (cerah/berawan/gerimis/angin) mengubah langit, intensitas cahaya, dan memunculkan partikel hujan atau debu angin. Warna aksen UI (`--acc`) ikut berubah mengikuti waktu.

Suara sepenuhnya sintetis lewat Web Audio API (oscillator + noise buffer) — tidak ada file audio.

### 8. Alur status game

```
menu → countdown (3-2-1-LARI) → playing → quiz (kalau kena item) → resume → playing
                              ↓
                           caught (dikejar guru, animasi jumpscare) → over → menu / main lagi
```

Rekor top-3 dengan nama pemain disimpan di `localStorage` (`motionRunnerLB`), begitu juga rekor terbaik, tingkat kesulitan, sensitivitas, dan preferensi waktu/cuaca.

---

## Struktur file

```
.
├── index.html   # seluruh game: HTML + CSS + JS + aset prosedural
└── README.md
```

---

## Tuning cepat

Kumpulan konstanta permainan ada di object `CFG` (bagian atas `<script>`):

```js
const CFG = {
  laneWidth: 2.2,       // jarak antar lajur
  startSpeed: 12,       // kecepatan awal (unit/detik)
  maxSpeed: 16.8,       // batas kecepatan
  jumpThreshold: 0.32,  // bahu naik sekian lebar-bahu = lompat
  squatThreshold: 0.45, // bahu turun sekian lebar-bahu = roll
  laneThreshold: 0.50,  // miring sekian lebar-bahu = pindah lajur
  lostMs: 700,          // tidak ada orang terdeteksi sekian ms = game dijeda
  delegate: 'CPU',      // ganti ke 'GPU' kalau laptop kuat
};
```

Tingkat kesulitan diatur di object `DIFFS`, daftar soal kuis di array `QUIZ`.

---

## Troubleshooting

| Masalah | Penyebab & solusi |
|---|---|
| Kamera tidak menyala | Ada aplikasi lain yang memakai kamera (Zoom, Meet, Discord, tab lain). Tutup dulu, lalu muat ulang. Pastikan popup izin browser diklik **Allow**. |
| `Kamera/model gagal: ...` | Biasanya WebGL2 mati — aktifkan **hardware acceleration** di pengaturan browser, lalu restart browser. Mode keyboard tetap bisa dipakai. |
| Game jalan tapi gerakan tidak terbaca | Berdiri lebih jauh dari kamera supaya bahu terlihat, pastikan pencahayaan cukup, atau naikkan slider sensitivitas. Tekan `C` untuk kalibrasi ulang. |
| Layar hitam / kosong | Halaman dibuka lewat `file://`. Jalankan server lokal (lihat bagian atas) dan buka `http://localhost:8000`. |
| Load pertama lambat | Model pose dan three.js diunduh dari CDN. Butuh internet; load berikutnya lebih cepat karena cache. |
| Port 8000 sudah dipakai | Ganti nomornya: `python -m http.server 8080`. |

---

## Catatan

- Butuh browser modern dengan **WebGL2** dan akses kamera: Chrome, Edge, atau Firefox versi terbaru.
- Performa kamera bergantung perangkat; status FPS dan waktu inferensi model bisa dilihat di pojok kanan bawah.
- Proyek ini dibuat untuk keperluan sekolah (SMK Telkom). Nama sekolah, tokoh guru, dan soal kuis bersifat fiktif/hiburan.
