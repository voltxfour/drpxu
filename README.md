# drxbox — fondasi port native *Digimon Rumble Arena 2* (Xbox) ke Android

> **Ini kerangka kerja, BUKAN game yang sudah bisa dimainkan.**
> Baca bagian **Status jujur** sebelum berharap apa pun.

Proyek ini melanjutkan pekerjaan sebelumnya (parser XBE + skeleton Android)
dan menambahkan: parser XBE yang **teruji**, port C++ yang **teruji identik**
dengan versi Python, serta **guest memory map** yang benar untuk alamat
32-bit Xbox.

---

## 1. Apa yang sudah jalan (dan terbukti)

| Komponen | Status | Bukti |
|---|---|---|
| `tools/xbe_info.py` — parser header XBE | **Jalan, teruji** | `python3 tools/test_xbe_info.py` → semua lulus |
| `xbe_loader.cpp` — port C++ | **Jalan, teruji** | `./build/host_test --selftest build/synthetic.xbe` |
| Kesamaan Python vs C++ | **Identik** | `python3 tools/compare_parsers.py` |
| `kernel_shim.cpp` — shim kernel + guest memory | **Sebagian, teruji** | `./build/host_test --shimtest` |
| Proyek Android (Kotlin + JNI + GLES3) | **Belum pernah di-build** | tidak ada Android SDK di mesin ini |

Yang dibaca parser: magic `XBEH`, base address, entry point, init flags,
TLS, stack/heap, sertifikat (TitleID, TitleName UTF-16, region, versi),
semua section (nama, VA, ukuran, offset), tabel *library version*
(D3D8, DSOUND, XAPILIB, XGRPHC, XNETLIB, …), dan **ordinal impor kernel**
dibaca dari pola thunk `B8 <ordinal>`.

---

## 2. Struktur

```
tools/
  xbe_info.py            parser XBE (Python) + laporan/JSON
  make_test_xbe.py       pembuat XBE sintetis untuk tes
  test_xbe_info.py       tes parser Python
  compare_parsers.py     uji silang Python vs C++
  build_host_test.sh     compile host_test dengan g++/clang++
  run_all_tests.sh       jalankan semua tes sekaligus
android/
  app/src/main/cpp/
    xbe_loader.{h,cpp}          parser XBE (C++17, tanpa dependensi)
    kernel_ordinal_names.inc    tabel nama ordinal (auto-generate dari Python)
    kernel_shim.{h,cpp}         GuestMemory + shim fungsi kernel
    jni_bridge.cpp              jembatan JNI
    host_test.cpp               tes di PC (tanpa Android)
    CMakeLists.txt
  app/src/main/java/com/drxbox/
    MainActivity.kt    pilih default.xbe (SAF), tampilkan hasil
    NativeBridge.kt    deklarasi fungsi native
    GameRenderer.kt    GLES 3.0 kosong (segitiga) — tempat renderer nanti
  app/src/main/res/... layout, tema, ikon adaptive
```

---

## 3. Cara pakai

### 3.1 Tes di komputer (tanpa Android)

```bash
bash tools/run_all_tests.sh
```

Atau manual:

```bash
python3 tools/test_xbe_info.py
bash tools/build_host_test.sh
python3 tools/make_test_xbe.py build/synthetic.xbe
./build/host_test --selftest build/synthetic.xbe
./build/host_test --shimtest
python3 tools/compare_parsers.py
```

### 3.2 Membedah XBE milikmu sendiri

```bash
python3 tools/xbe_info.py default.xbe
python3 tools/xbe_info.py default.xbe --json > xbe.json
```

Hasilnya menentukan besar pekerjaan: library apa yang dipakai, dan berapa
banyak ordinal kernel yang harus di-shim.

### 3.3 Build aplikasi Android

Butuh Android Studio (Ladybug atau lebih baru) atau Android SDK + NDK 26.1.

```bash
cd android
./gradlew assembleDebug     # butuh Android SDK terpasang & ANDROID_HOME diset
```

APK ada di `android/app/build/outputs/apk/debug/`.

Di aplikasi: tekan **Pilih default.xbe** → pilih file dari penyimpanan →
aplikasi menyalin ke cache, membaca header, dan menampilkan ringkasan +
laporan cakupan shim.

---

## 4. Status jujur

**Yang belum ada, dan ini yang paling penting:**

1. **Belum ada emulasi CPU.** Belum ada penerjemah x86 → ARM64. Tanpa ini,
   kode game tidak bisa dijalankan sama sekali. Ini pekerjaan terbesar
   (static recompilation atau interpreter).
2. **Belum ada renderer D3D8.** `GameRenderer` hanya menggambar segitiga.
   Perlu penerjemah D3D8 → GLES3/Vulkan, termasuk *register combiner* NV2A.
3. **Belum ada audio.** DirectSound → AAudio belum disambungkan.
4. **Belum ada I/O file game.** `NtCreateFile`/`NtReadFile` masih stub;
   butuh integrasi SAF + VFS.
5. **Proyek Android belum pernah di-build** di mesin ini (tidak ada SDK/NDK).
   Kemungkinan ada error kecil saat build pertama — kirim log-nya.
6. **Belum diuji dengan XBE asli.** Semua tes memakai XBE sintetis.
7. **Tabel nama ordinal tidak lengkap** (334 entri, ditulis dari referensi
   umum). Verifikasi terhadap daftar resmi sebelum dipercaya; `--names`
   memungkinkan menimpanya.

Perkiraan realistis: mencapai **layar judul** saja masih butuh emulasi CPU
+ D3D8 dasar + VFS. Ini proyek berbulan-bulan, bukan akhir pekan.

---

## 5. Catatan desain penting: alamat guest vs host

Xbox 32-bit; host 64-bit. Alamat seperti `0x0012A000` **bukan** pointer.
Versi pertama shim ini salah (cast langsung) dan langsung segfault saat tes —
sekarang semua akses lewat `GuestMemory::Translate(va, len)` yang memvalidasi
rentang. Ini pola yang harus dipakai konsisten di seluruh runtime.

---

## 6. Urutan kerja berikutnya

1. Jalankan `xbe_info.py` pada XBE asli → kumpulkan daftar library & ordinal.
2. Lengkapi `kernel_shim` untuk ordinal yang benar-benar dipakai (prioritas:
   memori, event, file I/O).
3. Tulis VFS: peta `\Device\Harddisk0\Partition1\...` → aset dari disc dump.
4. Implementasi emulasi CPU (mulai dari interpreter kecil; profil dulu).
5. Sambungkan D3D8 → GLES3 untuk satu draw call pertama.
6. Baru bicara soal "bisa main".

---

## 7. Hukum

Kode dan aset *Digimon Rumble Arena 2* milik Bandai Namco / pemegang haknya.
Repositori ini **hanya berisi runtime/alat**; tidak memuat XBE, ISO, atau aset
game. Gunakan hanya dump dari disc yang kamu miliki sendiri, dan jangan
mendistribusikannya.
