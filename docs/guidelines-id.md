# 📘 Panduan Single Owner Anti-Pattern PHP 8.4^

## 🎯 Tujuan
Dokumen ini adalah panduan membangun sistem pipeline di PHP 8.4^ dengan prinsip **single owner**. Prinsip ini memastikan bahwa semua data hanya memiliki **satu pemilik** (refcount ~ 1 → 0), setiap variabel wajib **dipindahkan (move)** setelah digunakan, dan tidak ada data yang dibiarkan hidup tanpa pemilik. Semua output harus melewati mekanisme **frame terverifikasi** dengan pola `START–END + token + length (+ checksum)`, sehingga **echo atau debug liar tidak pernah bocor ke STDOUT**.

---

## 🧩 Ide Dasar

1. **Linear Flow**
    - **Goal:** Memastikan semua data hanya melewati satu jalur eksekusi yang linear, sehingga setiap tahap mengonsumsi data dari tahap sebelumnya dan menghasilkan data baru.
    - **Setup:** Setiap variabel yang selesai digunakan langsung diganti dengan hasil baru. Tidak ada data yang diakses ulang atau dipakai bersamaan di beberapa tempat.

2. **Ephemeral Data**
    - **Goal:** Menganggap semua variabel bersifat sementara (ephemeral), sehingga tidak ada cache jangka panjang dalam proses PHP.
    - **Setup:** Setiap data yang butuh persistensi harus disimpan di luar proses (misalnya Redis atau file sementara).

3. **Move Semantics**
    - **Goal:** Menetapkan bahwa setiap assignment dianggap sebagai pemindahan (move) kepemilikan, bukan duplikasi.
    - **Setup:** Setiap variabel yang dipindahkan tidak boleh digunakan kembali. Refcount selalu turun menjadi 0 setelah move.

4. **Framed Contract**
    - **Goal:** Menjamin semua data keluar dengan format frame yang jelas, agar integritas dapat diperiksa.
    - **Setup:** Setiap payload dibungkus dengan header dan footer `START–END`, metadata `token`, panjang data (length), serta checksum/HMAC.

5. **Anti-Caching / Anti-Sharing**
    - **Goal:** Mencegah adanya cache atau sharing variabel dalam proses PHP.
    - **Setup:** Semua data harus bersifat single-use. Jika cache dibutuhkan, letakkan di luar proses dan sifatnya immutable.

6. **Memory = Transit Area**
    - **Goal:** Menjadikan memori hanya sebagai area transit, bukan storage.
    - **Setup:** Setiap data diproses lalu langsung dibuang. Tidak ada data idle di dalam memori.

7. **Eksekusi Deterministik**
    - **Goal:** Menjamin bahwa setiap input selalu menghasilkan output yang sama.
    - **Setup:** Terapkan normalisasi data (misalnya sort key array, format float konsisten) agar hasil deterministik.

8. **Trade-off**
    - **Goal:** Memahami konsekuensi bahwa pola single owner akan menambah beban CPU tetapi sangat mengurangi beban memori.
    - **Setup:** Uji coba dengan payload besar untuk mengukur keseimbangan antara tambahan CPU dan efisiensi memori.

---

## ⚠️ Celah dan Perbaikan

- **Library pihak ketiga yang melakukan echo atau menyimpan state global.**
    - *Celah:* Echo liar atau global state bisa menyalahi prinsip single owner.
    - *Perbaikan:* Drop OB level asing, audit superglobals, dan paksa semua output lewat frame.

- **Recompute data yang mahal.**
    - *Celah:* Tanpa cache internal, recompute bisa memberatkan CPU.
    - *Perbaikan:* Izinkan cache eksternal (Redis, temp file) dengan TTL pendek, dianggap sebagai sumber baru.

- **Copy-on-Write di PHP.**
    - *Celah:* Mutasi kecil pada array besar bisa memicu copy besar.
    - *Perbaikan:* Gunakan pendekatan streaming, minimalkan mutasi, hindari concat string di loop.

- **Length data ambigu pada UTF-8.**
    - *Celah:* Panjang karakter bisa berbeda dengan panjang byte.
    - *Perbaikan:* Gunakan byte length dan catat encoding di header frame.

- **Checksum CRC32 lemah.**
    - *Celah:* CRC32 rawan collision.
    - *Perbaikan:* Gunakan HMAC-SHA256 untuk jalur kritikal.

- **Parsing dengan regex pada buffer besar.**
    - *Celah:* Regex pada data besar memakan CPU tinggi.
    - *Perbaikan:* Gunakan parser state-machine linear.

- **Tanpa cache bisa memukul latency.**
    - *Celah:* Semua recompute bisa menambah waktu eksekusi.
    - *Perbaikan:* Gunakan cache eksternal immutable sebagai sumber.

- **Fragmentasi memori.**
    - *Celah:* Proses panjang bisa menyebabkan fragmentasi.
    - *Perbaikan:* Recycle proses setelah sejumlah frame tertentu.

- **Determinisme rusak karena locale atau urutan key.**
    - *Celah:* Serializer bisa menghasilkan hasil berbeda antar run.
    - *Perbaikan:* Normalisasi array dengan sort key, fix format float, dan set locale.

- **Time-To-First-Byte (TTFB) meningkat.**
    - *Celah:* Flush di akhir membuat respons lambat.
    - *Perbaikan:* Gunakan chunked flush (64–256KB) dengan validasi per chunk.

- **STDOUT blocking karena consumer lambat.**
    - *Celah:* Bisa membuat proses macet.
    - *Perbaikan:* Terapkan backpressure policy dan timeout.

- **Token lemah atau dapat ditebak.**
    - *Celah:* Token mudah diprediksi berisiko disalahgunakan.
    - *Perbaikan:* Gunakan token acak dari CSPRNG dengan masa hidup pendek.

---

## 🧪 Daftar Eksperimen

### A. Validasi dan Keutuhan Frame
1. **Smoke test antara payload valid dan echo liar.**
    - *Goal:* Memastikan hanya payload berframe valid yang keluar ke STDOUT.
    - *Setup:* Jalankan proses dengan satu payload valid dan tiga echo liar (sebelum, di dalam, dan sesudah frame).

2. **Frame dengan panjang data salah atau checksum salah.**
    - *Goal:* Memastikan frame rusak ditolak.
    - *Setup:* Kirim tiga frame (dua valid, satu dengan panjang/CRC salah).

3. **Frame bersarang dan token ganda.**
    - *Goal:* Memastikan parser hanya memproses token resmi.
    - *Setup:* Sisipkan START/END token lain di dalam payload.

4. **Data UTF-8 dengan normalisasi berbeda.**
    - *Goal:* Memastikan frame dihitung berdasarkan byte length, bukan jumlah karakter.
    - *Setup:* Kirim payload berisi emoji atau karakter multi-byte dengan NFC dan NFD.

---

### B. Performa: Throughput dan Memori
5. **Payload dari 1 KB hingga 100 MB.**
    - *Goal:* Melihat skala kinerja sistem.
    - *Setup:* Uji payload dengan ukuran bertingkat (1KB, 1MB, 10MB, 100MB).

6. **Flush tunggal vs flush chunked.**
    - *Goal:* Mengukur TTFB, total waktu, dan peak memori.
    - *Setup:* Bandingkan payload besar yang diflush sekali dengan payload yang diflush per 64KB/128KB.

7. **Sink string concatenation vs php://temp.**
    - *Goal:* Membuktikan efisiensi stream dibanding string concat.
    - *Setup:* Proses payload 10–50MB dengan dua mode sink.

8. **Parser regex vs parser state machine.**
    - *Goal:* Membandingkan konsumsi CPU.
    - *Setup:* Jalankan parsing pada payload besar dengan regex dan parser linear.

9. **Serializer JSON vs igbinary/msgpack.**
    - *Goal:* Mengukur kecepatan serialisasi dan ukuran output.
    - *Setup:* Serialize array besar dengan JSON dan serializer alternatif.

---

### C. Latency, Backpressure, dan Robustness
10. **Backpressure STDOUT dengan consumer lambat.**
    - *Goal:* Memastikan pipeline tidak macet.
    - *Setup:* Pipe output ke proses yang membaca lambat (usleep).

11. **Timeout per frame.**
    - *Goal:* Memastikan frame macet tidak membuat proses hang.
    - *Setup:* Injeksi delay antar chunk lebih lama dari SLA.

12. **Quota dan recycling proses.**
    - *Goal:* Mengendalikan fragmentasi memori.
    - *Setup:* Proses 10.000 frame kecil berturut-turut, ukur RSS dan peak memori.

---

### D. Determinisme dan Normalisasi
13. **Array associative dengan kunci acak.**
    - *Goal:* Memastikan output konsisten.
    - *Setup:* Serialize array dengan urutan key acak.

14. **Stabilitas float dan locale.**
    - *Goal:* Memastikan angka selalu diformat sama.
    - *Setup:* Serialize float dengan berbagai locale.

---

### E. Noise dan Fault Injection
15. **Noise storm dengan echo spam.**
    - *Goal:* Memastikan hanya frame valid yang lolos.
    - *Setup:* Kirim echo spam 10MB ditambah satu frame valid.

16. **Frame terpotong.**
    - *Goal:* Memastikan frame tidak lengkap ditolak.
    - *Setup:* Kirim frame dengan LEN lebih besar dari body aktual.

17. **Header frame dengan fuzzing.**
    - *Goal:* Memastikan parser tahan terhadap input aneh.
    - *Setup:* Gunakan header dengan token panjang atau versi tidak dikenal.

---

### F. Keamanan dan Integritas
18. **Keunikan token pada jumlah besar.**
    - *Goal:* Memastikan token acak tidak kolisi.
    - *Setup:* Hasilkan 1 juta token, cek entropi dan duplikasi.

19. **Perbandingan CRC32 vs HMAC-SHA256.**
    - *Goal:* Mengukur biaya tambahan CPU.
    - *Setup:* Serialize payload besar dengan dua metode checksum.

---

### G. Concurrency dan Proses
20. **Keamanan fork dengan pcntl_fork.**
    - *Goal:* Memastikan output tidak tumpang tindih.
    - *Setup:* Fork beberapa child, masing-masing emit frame dengan token berbeda.

21. **Multi-producer ke single-consumer pipeline.**
    - *Goal:* Memastikan urutan frame tetap benar.
    - *Setup:* Jalankan beberapa producer → pipe ke satu consumer.

---

### H. Observability dan Logging
22. **Disiplin logging.**
    - *Goal:* Memastikan STDOUT bersih, log hanya ke STDERR.
    - *Setup:* Kirim banyak warning/error selama proses.

23. **Metrics sampling.**
    - *Goal:* Memastikan overhead rendah.
    - *Setup:* Catat durasi encode/flush tiap N frame, hitung overhead.

---

### I. Praktikal dan Edge Cases
24. **Batas ukuran frame maksimum.**
    - *Goal:* Memastikan frame terlalu besar ditolak.
    - *Setup:* Kirim frame >128MB.

25. **Perbedaan versi schema.**
    - *Goal:* Memastikan consumer menolak atau fallback elegan.
    - *Setup:* Producer mengirim versi baru, consumer masih versi lama.

26. **Perilaku GC dan COW.**
    - *Goal:* Memastikan move semantics membebaskan memori.
    - *Setup:* Uji array besar dipindahkan antar fungsi, ukur peak memory.

---

## 📊 Template Laporan Eksperimen

| ID | Nama Uji | Setup | Goal | Metrik | Hasil | Status |
|----|----------|-------|------|--------|-------|--------|
| 1  | Smoke test | 1 payload + 3 echo liar | Hanya payload valid keluar | STDOUT bytes | ✅ payload bersih | Pass |
| 5  | Payload besar | 50MB JSON | Mengukur throughput & peak mem | MB/s, memory_get_peak_usage | … | … |
| 6  | Chunked flush | 50MB @64KB | Mengukur TTFB & memory | waktu, peak mem | … | … |
| …  | … | … | … | … | … | … |

---

## ✅ Ringkasan
- **Pola inti:** Semua data wajib dipindahkan (single owner), tidak ada variabel idle.
- **Guard:** Output wajib melewati frame terverifikasi dengan metadata (token, length, checksum).
- **Celah:** Biaya CPU, echo liar, recompute, parsing regex.
- **Perbaikan:** Gunakan stream, chunking, parser linear, HMAC opsional.
- **Eksperimen:** Jalankan checklist A–I (1–26) untuk memastikan validasi, performa, robustness, security, dan determinisme.
