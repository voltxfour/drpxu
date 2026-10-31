# PROMPT HANDOFF — drxbox (port native Digimon Rumble Arena 2 Xbox → Android)

> **Cara pakai:** tempel seluruh isi file ini sebagai pesan pertama ke AI berikutnya.
> Dokumen ini menggantikan "memori" percakapan sebelumnya.

---

## 0. Ringkasan satu paragraf

Kita membangun **drxbox**: runtime Android yang menjalankan `default.xbe`
(Xbox, x86 32-bit) milik pengguna sendiri, khusus untuk *Digimon Rumble Arena 2*.
Pendekatannya **static recompilation + lapisan kompatibilitas**, bukan emulator
hardware umum. Saat ini yang ada: parser XBE (Python + C++) yang **teruji**,
`GuestMemory` untuk translasi alamat 32-bit→host, shim kernel awal, dan
kerangka aplikasi Android (Kotlin + JNI + GLES3). **Game belum bisa dimainkan** —
belum ada emulasi CPU, renderer D3D8, audio, atau VFS. Itu pekerjaan berikutnya.

---

## 1. Aturan main (WAJIB dipatuhi AI berikutnya)

1. **Jangan pernah bilang "bisa" untuk sesuatu yang belum diverifikasi.**
   Setiap klaim harus punya bukti berupa output tes yang benar-benar dijalankan.
2. **Jangan pernah menyerahkan APK/artefak palsu.** Kalau tidak bisa build,
   katakan tidak bisa dan jelaskan alasannya.
3. **Jangan distribusikan XBE/ISO/aset game.** Repo hanya berisi runtime + alat.
   Pengguna memakai dump dari disc miliknya sendiri.
4. **Jangan menulis ulang kode yang sudah teruji tanpa alasan.** Jalankan
   `bash tools/run_all_tests.sh` dulu; kalau lulus, jangan diutak-atik.
5. **Jangan menambah fitur baru sebelum milestone saat ini selesai.**
6. Kalau ragu soal struktur XBE atau ordinal kernel, **verifikasi ke
   xboxdevwiki**, jangan mengarang dari ingatan.
7. Komunikasi ke pengguna: Bahasa Indonesia, jujur, tanpa janji berlebihan.

---

## 2. Kondisi terkini — apa yang SUDAH ada dan TERBUKTI

### Struktur file

```
README.md                      dokumentasi + status jujur + roadmap
todo.md                        checklist kerja (semua [x] untuk tahap 1-3)
.gitignore                     abaikan build/, *.xbe, *.iso, .gradle/

tools/
  xbe_info.py                  parser XBE (Python) + laporan teks + --json + --names
  make_test_xbe.py             generator XBE sintetis (struktur valid, nilai diketahui)
  test_xbe_info.py             tes parser Python (24 assertion)
  compare_parsers.py           uji silang Python vs C++ (harus IDENTIK)
  build_host_test.sh           compile host_test dengan g++/clang++
  run_all_tests.sh             jalankan 6 tahap verifikasi sekaligus

android/
  settings.gradle.kts          rootProject "drxbox", include(":app")
  build.gradle.kts             AGP 8.5.2, Kotlin 1.9.24
  gradle.properties            AndroidX, jvmargs 2g
  gradlew / gradlew.bat        Gradle wrapper 8.7 (jar asli sudah ada)
  gradle/wrapper/              gradle-wrapper.jar + .properties (8.7-bin)

  app/build.gradle.kts         namespace com.drxbox, compileSdk 34, minSdk 26,
                               ndkVersion 26.1.10909125, abiFilters arm64-v8a/
                               armeabi-v7a/x86_64, CMake 3.22.1
  app/proguard-rules.pro       keep com.drxbox.NativeBridge

  app/src/main/AndroidManifest.xml       GLES 3.0 required, MainActivity portrait

  app/src/main/cpp/
    CMakeLists.txt             lib "drxbox" SHARED, C++17, -fno-exceptions -fno-rtti
    xbe_loader.h / .cpp        parser XBE C++17, tanpa dependensi eksternal
    kernel_ordinal_names.inc   tabel 334 nama ordinal (AUTO-GENERATE dari Python)
    kernel_shim.h / .cpp       GuestMemory + KernelShim (Dispatch, CoverageReport)
    jni_bridge.cpp             5 fungsi JNI
    host_test.cpp              tes di PC (--selftest, --shimtest)

  app/src/main/java/com/drxbox/
    MainActivity.kt            SAF picker → salin ke cache → tampilkan ringkasan
    NativeBridge.kt            object, 5 external fun
    GameRenderer.kt            GLES 3.0, gambar 1 segitiga (bukti pipeline hidup)

  app/src/main/res/
    layout/activity_main.xml   GLSurfaceView + 2 tombol + TextView monospace
    values/{strings,colors,themes}.xml
    drawable/ic_launcher_foreground.xml
    mipmap-anydpi-v26/ic_launcher{,_round}.xml
```

### Bukti verifikasi (output nyata, bukan klaim)

```
=== 1. Tes parser Python ===        SEMUA TES LULUS (24 assertion)
=== 2. Compile host_test (C++) ===  OK (bersih, -Wall -Wextra -Wpedantic -Werror)
=== 3. Buat XBE sintetis ===        build/synthetic.xbe: 12288 byte
=== 4. Self-test parser C++ ===     SEMUA ASSERTION LULUS (22 assertion)
=== 5. Tes shim kernel ===          SEMUA ASSERTION LULUS (14 assertion)
=== 6. Uji silang Python vs C++ === PARSER PYTHON DAN C++ IDENTIK
```

Tambahan yang sudah dicek:
- 8/8 file XML resource valid (`xml.etree`).
- `jni_bridge.cpp` lolos `-fsyntax-only` dengan `jni.h` asli (JDK 17) + stub
  `android/log.h`.
- `nm -D` menunjukkan 5 simbol `Java_com_drxbox_NativeBridge_*` yang **cocok
  persis** dengan 5 `external fun` di `NativeBridge.kt`.

### Yang BELUM ada (jangan diklaim ada)

| Ketiadaan | Dampak |
|---|---|
| Emulasi CPU x86→ARM64 | **Tidak ada satu instruksi game pun yang bisa dijalankan** |
| Renderer D3D8 / NV2A | Tidak ada grafis; `GameRenderer` cuma segitiga |
| Audio DirectSound → AAudio | Tidak ada suara |
| VFS (NtCreateFile/NtReadFile) | Game tidak bisa membaca aset dari disc |
| Build APK | **Belum pernah di-build.** Sandbox hanya punya 2.2 GB bebas; NDK 26 butuh ~3.5 GB |
| Uji dengan XBE asli | Semua tes memakai XBE sintetis |
| Verifikasi tabel ordinal | 334 entri ditulis dari referensi umum, belum dicocokkan ke daftar resmi |

---

## 3. Keputusan desain penting (jangan diubah tanpa alasan kuat)

### 3.1 Alamat guest 32-bit ≠ pointer host 64-bit

Ini jebakan yang **sudah kena dan sudah diperbaiki**. Versi pertama shim
melakukan `reinterpret_cast<void*>(alamat_guest)` → **segfault**, dibuktikan
lewat gdb (`SIGSEGV` di `memset` dalam `KernelShim::Dispatch`).

Aturan sekarang: **semua** akses memori game lewat
`GuestMemory::Instance().Translate(va, len)`, yang mengembalikan `nullptr`
kalau `[va, va+len)` keluar dari region. Kalau `nullptr`, naikkan
`invalid_pointer_count_` dan jangan dereference.

`GuestMemory` saat ini: `std::vector<Region>` + mutex, `next_guest_` mulai
`0x00100000`, alokasi dibulatkan ke 4 KB. Ini **linear scan** — nanti harus
diganti struktur yang lebih cepat (interval tree / page table) saat region
sudah banyak.

### 3.2 Parser XBE harus dua tahap

Jebakan kedua yang sudah kena: nama section berada di `.rdata`, tapi rentang
VA `.rdata` belum terdaftar saat nama section dibaca → nama kosong.
Perbaikannya: **tahap 1** baca semua section header + daftarkan rentang VA,
**tahap 2** baru resolve `SectionNameAddress`. Jangan digabung lagi.

### 3.3 Offset header XBE yang dipakai (xboxdevwiki)

```
0x104 base_address        0x118 certificate_addr     0x158 kernel_thunk_addr
0x108 size_of_headers     0x11C num_sections         0x15C nonkernel_import
0x10C size_of_image       0x120 section_headers      0x160 num_library_versions
0x110 size_of_image_hdr   0x124 init_flags           0x164 library_versions
0x114 time_date           0x128 entry_point          0x150 debug_filename
0x12C tls_addr  0x130 stack_commit  0x134 heap_reserve  0x138 heap_commit
0x13C pe_base   0x140 pe_image_size

section header 0x38 byte: flags 0x00, vaddr 0x04, vsize 0x08,
                          faddr 0x0C, fsize 0x10, name_addr 0x14, refcount 0x18
library version 0x0C byte: name_addr 0x00, major 0x04, minor 0x06,
                           build 0x08, flags 0x0A
certificate: title_id 0x08, title_name 0x0C (40 × UTF-16LE),
             allowed_media 0x9C, region 0xA0, disc_number 0xB0, version 0xB4
```

### 3.4 Pola thunk kernel

```
B8 <ordinal:u32>   mov eax, ordinal
BA <addr:u32>      mov edx, <kernel table entry>
FF D2              call edx
C3                 ret
```
Parser mengambil `u32` di offset +1. Ada fallback: cari byte `B8` di 16 byte
pertama dengan nilai `0 < v < 0x10000`. Tabel thunk = array `u32` (VA) yang
diakhiri `0`.

### 3.5 Konvensi panggilan di shim

`KernelCall { ordinal, args[16], argc, result, handled }` — sengaja array,
bukan fungsi bertanda tangan asli, supaya portabel dan mudah di-log.
Konvensi Xbox = stdcall (argumen di stack, dibersihkan callee).

### 3.6 Tabel ordinal auto-generate

`kernel_ordinal_names.inc` **dihasilkan** dari `KERNEL_ORDINAL_NAMES` di
`tools/xbe_info.py`. Kalau menambah nama, edit Python lalu regenerate:

```bash
python3 - <<'EOF'
import sys; sys.path.insert(0,'tools'); import xbe_info
items = sorted(xbe_info.KERNEL_ORDINAL_NAMES.items())
lines = ["// AUTO-GENERATED dari tools/xbe_info.py — jangan diedit tangan.",
         "static const struct { uint32_t ordinal; const char* name; } kKernelNames[] = {"]
lines += ['    {%d, "%s"},' % (o,n) for o,n in items]
lines.append("};")
open('android/app/src/main/cpp/kernel_ordinal_names.inc','w').write("\n".join(lines)+"\n")
EOF
```

---

## 4. Perintah yang sering dipakai

```bash
# verifikasi penuh (jalankan ini dulu sebelum mengubah apa pun)
bash tools/run_all_tests.sh

# bedah XBE milik pengguna
python3 tools/xbe_info.py default.xbe
python3 tools/xbe_info.py default.xbe --json > xbe.json
python3 tools/xbe_info.py default.xbe --names names.txt   # timpa nama ordinal

# tes manual
python3 tools/test_xbe_info.py
bash tools/build_host_test.sh
python3 tools/make_test_xbe.py build/synthetic.xbe
./build/host_test --selftest build/synthetic.xbe
./build/host_test --shimtest
python3 tools/compare_parsers.py

# build Android (BUTUH Android SDK + NDK, tidak ada di sandbox)
cd android && ./gradlew assembleDebug
```

Dependensi sandbox yang sudah dipasang: `g++`, `cmake`, `gdb`,
`openjdk-17-jdk-headless`. **Tidak ada** Android SDK/NDK.

---

## 5. LANGKAH SELURUHNYA — urutan kerja berikutnya

### Milestone 0 — data nyata (PRASYARAT, minta ke pengguna)

Minta pengguna menjalankan `python3 tools/xbe_info.py default.xbe` dan
mengirim output-nya. Tanpa ini kita bekerja buta.

Yang harus dicatat dari output:
- daftar **library** (apakah ada D3D8? versi build berapa? ada XGRPHC?)
- daftar **ordinal kernel** yang dipakai (ini menentukan urutan kerja shim)
- jumlah & ukuran **section** (perkiraan besar kode yang harus di-recompile)
- **entry point** dan `init_flags`

### Milestone 1 — VFS: game bisa membaca asetnya

Target: `NtCreateFile` / `NtOpenFile` / `NtReadFile` / `NtClose` / `NtQueryInformationFile`
bekerja untuk path Xbox (`\Device\Harddisk0\Partition1\...`, `D:\...`).

Rencana:
1. Buat `Vfs` di C++: map path Xbox → entri di direktori aset.
2. Di Android, aset dibuka lewat **SAF** (`ParcelFileDescriptor`), bukan path
   biasa. Teruskan `fd` ke native, lalu `pread` di native.
3. Handle file = indeks ke tabel internal (bukan pointer host mentah).
4. Uji di PC dulu dengan direktori biasa (`host_test --vfstest`), baru Android.

### Milestone 2 — Emulasi CPU (BAGIAN TERBESAR)

Target: menjalankan instruksi x86 dari `.text` sampai `ret` pertama.

Rencana bertahap:
1. **Interpreter kecil** dulu (bukan JIT): register 32-bit, flag, stack,
   instruksi umum (`mov`, `push`, `pop`, `call`, `ret`, `add`, `sub`, `cmp`,
   `jcc`, `lea`, `test`, `and`, `or`, `xor`, `shl`, `shr`).
2. Muat section ke `GuestMemory` sesuai VA aslinya, set `eip = entry_point`.
3. Log setiap instruksi yang belum dikenal → itu daftar kerja berikutnya.
4. Setelah interpreter stabil, **baru** pertimbangkan static recompilation
   (lift x86 → C++) untuk kecepatan. Jangan mulai dari sini.

Catatan penting: target ARM64 Android, tapi **kembangkan & uji di x86_64 PC
dulu** — jauh lebih cepat iterasi.

### Milestone 3 — Renderer D3D8 minimum

Target: satu frame pertama muncul di layar.

1. Tangkap semua panggilan `IDirect3DDevice8` (via shim import non-kernel).
2. Implementasi minimum: `Clear`, `BeginScene`/`EndScene`, `SetRenderState`
   (subset), `SetStreamSource`, `SetVertexShader` (FVF), `DrawPrimitive`.
3. Terjemahkan FVF → vertex layout GLES3.
4. **Register combiner NV2A** adalah bagian tersulit — mulai dari kasus
   sederhana (modulate, select arg), jangan coba lengkap sekaligus.
5. Uji dengan draw call tunggal (segitiga berwarna) sebelum tekstur.

### Milestone 4 — Audio

`DirectSoundCreate` / `IDirectSoundBuffer::Lock/Unlock/Play` → AAudio.
Mulai dari buffer PCM 16-bit stereo 48 kHz.

### Milestone 5 — Input

Gamepad Bluetooth (AXIS_X/Y, tombol A/B/X/Y, trigger) → struktur input Xbox.
Fallback: on-screen controller overlay.

### Milestone 6 — Integrasi & performa

Ganti `GuestMemory` linear scan → page table. Profil dengan `simpleperf`.
Target awal: menu utama 30 fps di perangkat mid-range.

---

## 6. Jebakan yang sudah terbukti (jangan diulang)

1. **Cast alamat guest ke pointer** → segfault. Selalu `Translate()`.
2. **Resolve nama section sebelum rentang VA terdaftar** → nama kosong.
   Pakai dua tahap.
3. **Menghitung ekspektasi tes dengan tangan** → assertion gagal padahal kode
   benar (kasus `Missing()` dihitung 5 padahal 4). Jalankan dulu, baru tulis
   angka yang diharapkan — atau lebih baik, hitung dari sumber yang sama.
4. **`reinterpret_cast` pointer host dari `jlong`/`jint`** di JNI: alamat guest
   itu 32-bit, jangan pernah diperlakukan sebagai `jlong` pointer.
5. **Mengandalkan nama ordinal dari ingatan** → bisa salah. Verifikasi ke
   xboxdevwiki; sediakan `--names` sebagai jalan keluar.

---

## 7. Yang harus ditanyakan ke pengguna sebelum mulai

1. Sudah punya dump `default.xbe` sendiri? Bisa jalankan `xbe_info.py` dan
   kirim output-nya?
2. Tipe HP & chipset (mis. Snapdragon 8 Gen 1 / Dimensity 1200 / Mali-G85)?
   Ini menentukan target performa dan apakah Vulkan realistis.
3. Apakah ada PC untuk pengembangan (x86_64, Windows/Linux)? Sangat membantu
   karena iterasi di PC jauh lebih cepat daripada di HP.
4. Nyaman dengan C/C++ dan Android NDK, atau perlu penjelasan langkah demi
   langkah?

---

## 8. Kalimat jujur yang harus dipertahankan

> Ini proyek berbulan-bulan, bukan akhir pekan. Sampai layar judul saja masih
> butuh emulasi CPU + D3D8 dasar + VFS. Yang sudah ada sekarang adalah
> fondasi yang terverifikasi, bukan game yang bisa dimainkan.

Jangan pernah menjanjikan tanggal selesai. Jangan pernah menyerahkan APK yang
tidak benar-benar di-build dan diuji.
