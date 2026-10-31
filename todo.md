# drxbox — lanjutan fondasi port native Digimon Rumble Arena 2 (Xbox) → Android

## 1. Parser XBE (Python)
- [x] `tools/xbe_info.py`: header XBEH, base addr, entry point, sections, certificate (TitleID/TitleName), library versions, kernel thunk ordinals, non-kernel imports
- [x] `tools/make_test_xbe.py` (XBE sintetis untuk tes)
- [x] `tools/test_xbe_info.py` + jalankan → SEMUA TES LULUS

## 2. Parser XBE (C++ / NDK)
- [x] `xbe_loader.h/.cpp` port dari Python
- [x] `host_test.cpp` + compile g++ (-Wall -Wextra -Wpedantic -Werror) → bersih
- [x] `tools/compare_parsers.py` → Python & C++ IDENTIK

## 3. Kerangka aplikasi Android
- [x] Gradle (settings/build/app), gradle.properties, wrapper 8.7, AndroidManifest
- [x] MainActivity.kt (pemilih file SAF + tampilan status)
- [x] NativeBridge.kt + JNI (5 simbol cocok dengan 5 fungsi Kotlin)
- [x] GameRenderer.kt (GLES3, segitiga uji)
- [x] CMakeLists.txt
- [x] kernel_shim.h/.cpp + GuestMemory (translasi alamat guest 32-bit)
- [x] Resource: layout, strings, colors, themes, adaptive icon

## 4. Verifikasi & paket
- [x] Compile host test C++ (g++, -Werror bersih)
- [x] Jalankan tes Python + shim test
- [x] Validasi semua XML resource (8/8 OK)
- [x] Compile-check jni_bridge.cpp dengan jni.h asli + stub android/log.h
- [x] README.md lengkap + status jujur
- [x] Zip proyek

## Catatan
- Build APK tidak bisa dilakukan di sandbox: disk 2.2 GB, NDK 26 butuh ~3.5 GB.
- Perbaikan besar dari versi sebelumnya: shim dulu me-reinterpret_cast alamat
  guest 32-bit jadi pointer host → segfault. Sekarang pakai GuestMemory.

## 5. Koreksi format XBE (terverifikasi ke pyxbe + XBE asli pengguna)
- [x] Ordinal kernel diganti tabel resmi (366+5 entri), shim dipetakan ulang by-name
- [x] XOR entry/thunk (debug+retail), thunk 0x80000000|ordinal, library 0x10 byte, offset cert
- [x] Generator sintetis: ctypes dari pyxbe, varian debug & retail
- [x] Tes: nama shim == tabel resmi; cakupan 158 ordinal DRA2 asli

## 6. Berikutnya
- [x] Milestone 1a — VFS baca-saja di PC: vfs.h/.cpp + NtCreateFile/NtOpenFile/NtReadFile/
      NtClose/NtQueryInformationFile(Standard,Position)/NtSetInformationFile(Position);
      tes `--vfstest` (50 assertion), ASan+UBSan bersih, mutasi uji ditangkap tes.
- [ ] Milestone 1b — sisa VFS: NtQueryVolumeInformationFile=218, NtQueryDirectoryFile=207,
      NtQueryFullAttributesFile=210, NtFlushBuffersFile=198 (semua dipakai DRA2);
      penulisan save (T:/U:) + NtWriteFile; mount drive lain.
- [ ] Milestone 1c — Android: aset lewat SAF (path host tidak bisa dipakai langsung);
      butuh backend berbasis fd dari Kotlin via JNI. Belum dikerjakan.
- [ ] BELUM TERVERIFIKASI ke sumber utama Xbox asli: layout OBJECT_ATTRIBUTES (12 byte),
      OBJECT_STRING ANSI, ukuran FILE_STANDARD_INFORMATION (22), nomor kelas informasi.
      Isolasi di KernelShim::NtOpenCommon/NtQueryInfoImpl; gagal keras bila salah.
- [ ] Penyedia variabel data kernel (KeTickCount, XboxKrnlVersion, LaunchDataPage, ...)
