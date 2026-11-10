<div align="center">

# ⚡ Panduan Studi: Kompilasi Alat & Flashing Firmware Phison PS2251-07

**Kompilasi Tool • Backup Firmware • Flashing Kustom Firmware (PS2307)**

[![Tools](https://img.shields.io/badge/Tools-Psychson-8250DF?logo=github&logoColor=white)](https://github.com/brandonlw/Psychson)
[![Controller](https://img.shields.io/badge/Controller-Phison%20PS2251--07-EA4335)](https://flashboot.ru/files/file/647/)
[![Patch](https://img.shields.io/badge/Patch-2307-00C853)](https://vivibit.net/psychson2307/)

</div>

> [!WARNING]
> **Catatan Penting (Konteks Akademik):**
>
> Dokumentasi ini disusun murni untuk tujuan **pembelajaran, studi arsitektur hardware, dan eksperimen mandiri**, BUKAN untuk tindakan peretasan (hacking) atau aktivitas ilegal lainnya. Seluruh prosedur wajib dijalankan pada perangkat milik sendiri atau perangkat yang telah mendapatkan izin operasional penuh.

Panduan ini berisi langkah-langkah terstruktur untuk memodifikasi, mengompilasi, melakukan *backup* firmware asli, hingga melakukan *flashing* kustom firmware pada kontroler **Phison PS2251-07 (PS2307)** menggunakan proyek berbasis **Psychson**.

---

## 📚 Sumber Referensi

Dokumen ini disusun dengan merujuk pada tiga sumber utama berikut:

| 🔗 Sumber | 📌 Kegunaan |
| :--- | :--- |
| [**github.com/brandonlw/Psychson**](https://github.com/brandonlw/Psychson) | Gunakan **DriveCom** dan **EmbedPayload** |
| [**flashboot.ru/files/file/647/**](https://flashboot.ru/files/file/647/) | Firmware **Phison** |
| [**vivibit.net/psychson2307/**](https://vivibit.net/psychson2307/) | Patch **Psychson (BadUSB)** untuk **PS2251-07 (2307)** |

---

## 📑 Daftar Isi

| # | Bagian | Ringkasan |
| :--- | :--- | :--- |
| 1 | [🛠️ Prasyarat & Lingkungan Pengembangan](#prasyarat) | Perangkat lunak, compiler, dan struktur folder kerja |
| 2 | [📂 Bagian 1: Persiapan & Kompilasi Alat Kerja](#bagian-1) | Build `DriveCom.exe` dan `EmbedPayload.exe` |
| 3 | [💾 Bagian 2: Backup Firmware Asli (Dumping)](#bagian-2) | Modifikasi burner, Boot Mode & dumping firmware asli |
| 4 | [⚡ Bagian 3: Kompilasi & Flashing Kustom Firmware](#bagian-3) | Patch seri 07, pembuatan & injeksi payload, pembakaran firmware |
| 5 | [🩺 Bagian 4: Pengujian Hasil Perangkat](#bagian-4) | Indikator keberhasilan mutasi perangkat |

---

<a id="prasyarat"></a>

## 🛠️ Prasyarat & Lingkungan Pengembangan

Sebelum memulai, pastikan perangkat lunak berikut telah terinstal dan siap digunakan di sistem Anda:

### 1. Perangkat Lunak & Compiler

| 🧰 Perangkat Lunak | 📋 Fungsi / Persyaratan |
| :--- | :--- |
| **Visual Studio**: | Untuk membuka dan melakukan *build* pada solusi proyek C# (`.sln`). |
| **SDCC (Small Device C Compiler)**: | Wajib terinstal di direktori standar (`C:\Program Files\SDCC`) untuk mengompilasi kode C mikrokontroler 8051. |
| **Git Bash**: | Digunakan untuk mengeksekusi perintah penambalan otomatis berbasis `.diff`. |
| **Java Runtime Environment (JRE)**: | Diperlukan untuk menjalankan utilitas `duckencoder.jar`. |

### 2. Struktur Direktori Kerja

Buatlah folder kerja terpusat di `C:\BadUSB\`. Pastikan file-file awal berikut sudah diletakkan di dalam folder tersebut sebelum eksekusi:

```text
C:\BadUSB\
├── BN07V502TAW.BIN     # (Berkas burner mentah khusus seri 07)
├── payload.txt         # Skrip Duckyscript sumber (dibuat pada Bagian 3)
├── duckencoder.jar     # Konverter Duckyscript -> inject.bin
└── inject.bin          # (Berkas biner payload hasil konversi Duckyscript)
```

### 3. Isi Repository

```text
.
├── README.md
├── tools/                     # Bagian 1 — alat kompilasi
│   ├── DriveCom/              # Source C# DriveCom.exe (komunikasi dengan chip)
│   └── EmbedPayload/          # Source C# EmbedPayload.exe (injeksi payload)
├── firmware-patch-2307/       # Bagian 3 — paket patch untuk PS2251-07 (2307)
│   ├── firmware-firmware2307.diff
│   ├── DriveCom-DriveCom2307.diff
│   ├── firmware/serial.c
│   ├── templates/FWdummy.bin
│   └── README.ja.txt
├── firmware-source/           # Bagian 3 — kode C firmware Psychson
│   ├── build.bat              # Skrip build SDCC
│   └── *.c / *.h              # main.c, usb.c, control.c, scsi.c, timers.c, defs.h, ...
└── firmware-backup/           # Bagian 2 — cadangan firmware asli
    ├── FW07V050150MICRONB95ANDHYNIXTLC.BIN
    └── mpall 53mlc.rar
```

> [!NOTE]
> `build.bat` di `firmware-source/` memakai path relatif ala Psychson (`..\templates\FWdummy.bin` dan `..\tools\sfk`). Bila ingin menjalankannya langsung dari repository ini, siapkan folder `templates/` (berisi `FWdummy.bin` dari `firmware-patch-2307/templates/`) dan `tools/` (berisi `sfk`) **sejajar** dengan `firmware-source/`.

---

<a id="bagian-1"></a>

## 📂 Bagian 1: Persiapan & Kompilasi Alat Kerja

### 1. Modifikasi & Build DriveCom.exe

`DriveCom.exe` digunakan untuk berkomunikasi dengan kontroler Phison. Untuk seri PS2251-07, diperlukan penyesuaian ukuran *payload length*.

1. Buka folder `tools/` pada repository ini, lalu masuk ke direktori `DriveCom/`.

2. Buka file proyek `DriveCom.sln` atau `DriveCom.csproj` menggunakan **Visual Studio**.

3. Buka file `tools/DriveCom/DriveCom/PhisonDevice.cs` dan cari fungsi `TransferFile` atau variabel `data.Length`.

4. Ubah baris kode berikut:

   ```csharp
   // SEBELUM
   var size = data.Length - 1024;

   // SESUDAH (Ubah menjadi 512 untuk seri 07)
   var size = data.Length - 512;
   ```

5. Simpan perubahan (`Ctrl + S`).

6. Ubah mode konfigurasi di bilah menu atas Visual Studio dari **Debug** menjadi **Release**.

7. Jalankan proses *build* dengan menekan **Ctrl + Shift + B** (atau menu *Build > Build Solution*).

8. Jika berhasil, ambil file `DriveCom.exe` di direktori `...\bin\Release\` dan pindahkan ke `C:\BadUSB\`.

> [!TIP]
> *Jika build gagal, klik kanan proyek di Solution Explorer -> Properties -> Target Framework. Pastikan versinya sesuai dengan .NET Framework yang terpasang di komputer Anda (biasanya v4.0 atau v4.5).*

### 2. Build EmbedPayload.exe

`EmbedPayload.exe` berfungsi untuk menyuntikkan file payload biner ke dalam firmware dasar.

1. Buka file proyek `EmbedPayload.sln` atau `EmbedPayload.csproj` di dalam folder `EmbedPayload` menggunakan **Visual Studio**.

2. Ubah konfigurasi ke mode **Release**.

3. Tekan **Ctrl + Shift + B** untuk melakukan kompilasi.

4. Salin file `EmbedPayload.exe` dari folder `bin/Release/` ke folder utama `C:\BadUSB\`.

---

<a id="bagian-2"></a>

## 💾 Bagian 2: Backup Firmware Asli (Dumping)

Langkah ini wajib dilakukan untuk mengamankan firmware asli flashdisk sebelum melakukan penulisan ulang.

### 1. Persiapan Awal

1. Hubungkan Flashdisk Phison PS2251-07 ke port USB.

2. Identifikasi huruf drive perangkat di Windows Explorer (Contoh: `E:\`).

3. Buka **Command Prompt (CMD)** sebagai **Administrator**, lalu arahkan direktori ke folder kerja:

   ```cmd
   cd C:\BadUSB
   ```

### 2. Modifikasi Berkas Burner (Unlocked Burner)

Sebelum melakukan backup, berkas `BN07V502TAW.BIN` harus dimodifikasi terlebih dahulu pada dua offset proteksi agar chip PS2307 dapat dibaca dan ditulis.

1. **Unduh aplikasi Hex Editor.** Anda membutuhkan aplikasi editor biner; yang paling umum dan gratis digunakan di Windows adalah **HxD Hex Editor** (atau **VS Code** dengan ekstensi *Hex Editor*).

2. **Buka berkas di Hex Editor.** Jalankan HxD, klik menu **File > Open**, lalu pilih berkas `BN07V502TAW.BIN` mentah Anda.

3. **Cari lokasi koordinat (offset).**
   * Di dalam aplikasi Hex Editor, Anda akan melihat deretan angka biner berpasangan. Di sisi paling kiri terdapat kolom baris bernama **Offset** (alamat memori).
   * Tekan tombol kombinasi **Ctrl + G** (Go to) pada keyboard Anda.
   * Masukkan atau ketik alamat offset `4D85` pada kolom pencarian alamat, lalu tekan **Enter**.

4. **Ubah nilainya menjadi `00`.**
   * Kursor Anda akan otomatis melompat tepat di koordinat data untuk posisi `0x4D85` dan `0x4D86` (dua kolom hex yang berdampingan).
   * Ketik langsung angka `00` pada baris koordinat `0x4D85` dan ketik `00` lagi pada koordinat `0x4D86`.
   * Teks yang Anda ubah biasanya akan berubah warna (misalnya menjadi merah) sebagai penanda bahwa data biner aslinya telah ditimpa.

5. **Simpan berkas.** Tekan **Ctrl + S** untuk menyimpan hasil perubahan tersebut.

> [!IMPORTANT]
> **Mengapa file hasil modifikasi ini harus sama untuk Bagian 2 dan Bagian 3?**
>
> Setelah disimpan, berkas `BN07V502TAW.BIN` di folder `C:\BadUSB\` berstatus **"Unlocked Burner"**, yaitu pembuka proteksi chip Phison PS2307. Pertahankan nama berkasnya, karena perintah `DumpFirmware` dan `SendFirmware` memanggilnya sebagai `/burner=BN07V502TAW.BIN`.
>
> * **Pada Bagian 2 (Backup):** berkas modifikasi `00` ini bertugas membypass enkripsi pabrikan agar perintah `DumpFirmware` bisa menyedot keluar isi data biner internal chip menjadi file `FirmwareAsliOriginal.bin`.
> * **Pada Bagian 3 (Flashing):** berkas modifikasi `00` yang sama bertugas membypass gerbang keamanan *write-protection* chip agar perintah `SendFirmware` diizinkan menyuntikkan dan menimpa sistem dengan berkas kustom biner `fw.bin` yang baru.

### 3. Eksekusi Perintah Backup

1. **Masuk ke Boot Mode**: Jalankan perintah berikut untuk mengalihkan flashdisk ke mode aman (lampu indikator akan mati/berkedip konstan, dan huruf drive akan menghilang sementara dari Windows Explorer):

   ```cmd
   DriveCom.exe /drive=E /action=SetBootMode
   ```

2. **Dump Firmware**: Lakukan pembacaan memori menggunakan berkas *burner* yang dimodifikasi (koordinat offset `0x4D85` dan `0x4D86` diubah menjadi `00` via Hex Editor untuk membuka proteksi) dengan perintah:

   ```cmd
   DriveCom.exe /drive=E /action=DumpFirmware /burner=BN07V502TAW.BIN /output=FirmwareAsliOriginal.bin
   ```

3. **Verifikasi**: Pastikan file `FirmwareAsliOriginal.bin` telah terbuat di `C:\BadUSB\` dan ukurannya tidak kosong (bukan 0 KB). Simpan file ini di tempat aman sebagai cadangan pemulihan.

> [!NOTE]
> DriveE → Disesuaikan

---

<a id="bagian-3"></a>

## ⚡ Bagian 3: Kompilasi & Flashing Kustom Firmware

### Tahap 1: Pembuatan Berkas Firmware Dasar (Mutasi ke Seri 07)

Proses ini mengubah arsitektur kode dasar Psychson (default seri 03) agar kompatibel dengan chip seri 07.

1. Buka direktori `Psychson/firmware/`.

2. Salin file `serial.c` dan file patch `firmware-firmware2307.diff` dari folder `firmware-patch-2307/` pada repository ini, lalu tempel (*paste*) ke dalam `Psychson/firmware/`.

3. Klik kanan di area kosong folder tersebut, pilih **Git Bash Here**, lalu jalankan perintah penambalan kode berikut:

   ```bash
   patch -p1 < firmware-firmware2307.diff
   ```

4. Di dalam folder yang sama, buat sebuah file baru bernama `build.bat`. Klik kanan file tersebut, pilih **Edit**, lalu rekatkan (*paste*) skrip otomatisasi di bawah ini:

   ```bat
   @echo off
   sdcc main.c usb.c serial.c
   packihx main.ihx > fw.hex
   makebin fw.hex fw.bin
   echo Kompilasi Berhasil Selesai!
   pause
   ```

5. Simpan dan tutup file tersebut, lalu jalankan dengan klik dua kali pada `build.bat`. File biner baru bernama `fw.bin` akan otomatis tercipta di folder tersebut.

> [!TIP]
> Perintah `pause` di baris paling bawah `build.bat` berfungsi agar jendela CMD tidak langsung menutup otomatis setelah proses selesai, sehingga Anda bisa membaca pesan error dari kompiler SDCC bila kompilasi gagal. Tekan sembarang tombol untuk menutup jendela setelah membacanya.

### Tahap 2: Membuat Payload (`inject.bin`) via CMD

`inject.bin` adalah hasil konversi skrip **Duckyscript** (`.txt`) menjadi biner yang nantinya disuntikkan ke firmware pada tahap berikutnya.

1. **Tulis Perintah Script (Duckyscript).** Buka **Notepad**, tulis skrip aksi otomatis yang Anda inginkan (misalnya membuka Notepad otomatis di laptop target), lalu simpan dengan nama `payload.txt` di folder `C:\BadUSB\`.

   Contoh isi skrip dasar:

   ```text
   DELAY 3000
   GUI r
   DELAY 500
   STRING notepad.exe
   ENTER
   DELAY 500
   STRING Halo, ini adalah uji coba Phison BadUSB HID!
   ENTER
   ```

   > [!CAUTION]
   > Gunakan kode dengan hati-hati.

2. **Jalankan Konversi dengan Duckencoder.** Pastikan Anda sudah meletakkan file pengonversi Java bernama `duckencoder.jar` di dalam folder `C:\BadUSB\`. Buka CMD Administrator, masuk ke direktori tersebut (`cd C:\BadUSB`), lalu jalankan perintah:

   ```cmd
   java -jar duckencoder.jar -i payload.txt -o inject.bin
   ```

   Setelah selesai, file `inject.bin` akan terbentuk di folder `C:\BadUSB\` dan siap digunakan pada tahap selanjutnya.

### Tahap 3: Penggabungan Skrip Injeksi (Payload)

1. Pindahkan file `fw.bin` yang baru saja dibuat ke folder utama `C:\BadUSB\`.

2. Buka kembali jendela CMD Administrator (`cd C:\BadUSB`) dan jalankan perintah berikut untuk menyuntikkan payload skrip pengetikan otomatis Anda:

   ```cmd
   EmbedPayload.exe inject.bin fw.bin
   ```

   *Sekarang file `fw.bin` telah siap sepenuhnya sebagai kustom firmware.*

### Tahap 4: Proses Flashing Utama (Pembakaran ke Chip)

Pastikan posisi flashdisk masih berada dalam **Boot Mode** dari langkah pemulihan/cadangan sebelumnya.

> [!IMPORTANT]
> Gunakan berkas burner `BN07V502TAW.BIN` yang **sama** dengan yang sudah dimodifikasi pada [Bagian 2](#bagian-2) (offset `0x4D85` dan `0x4D86` diubah menjadi `00`). Berkas *Unlocked Burner* inilah yang membuka proteksi tulis agar `SendFirmware` diizinkan menimpa chip dengan `fw.bin`.

1. Jalankan perintah pembakaran berikut pada CMD Administrator:

   ```cmd
   DriveCom.exe /drive=E /action=SendFirmware /burner=BN07V502TAW.BIN /firmware=fw.bin
   ```

   *(Sesuaikan huruf `E` dengan identitas drive asal flashdisk Anda sebelum masuk ke Boot Mode).*

2. ⚠️ **PERINGATAN PROSES:** Tunggu hingga indikator CMD menunjukkan status sukses penuh. **Jangan mencabut flashdisk atau mematikan komputer** selama proses penulisan firmware berlangsung untuk menghindari kerusakan permanen (*brick*).

---

<a id="bagian-4"></a>

## 🩺 Bagian 4: Pengujian Hasil Perangkat

1. Setelah proses flashing dinyatakan sukses oleh program, cabut flashdisk dari port USB.

2. Tancapkan kembali flashdisk tersebut untuk menguji mutasi perangkat.

3. **Indikator Keberhasilan Mutasi:**

   * Flashdisk **tidak akan memunculkan drive penyimpanan lagi** di Windows Explorer.
   * Sistem operasi Windows akan mendeteksinya secara otomatis sebagai perangkat masukan **Keyboard Eksternal (HID / Human Interface Device)**.
   * Perangkat akan langsung mengeksekusi otomatis seluruh rangkaian tugas pengetikan (*payload*) yang telah dirancang di dalam file biner sebelumnya.

---

<div align="center">

**✦ Dokumen akademik — untuk pembelajaran, studi arsitektur hardware, dan eksperimen mandiri. ✦**

</div>
