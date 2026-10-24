# PROJECT SPEC: Simulator dan Pendeteksi Peleburan Warna untuk Buta Warna (Protanopia, Deuteranopia, Tritanopia)

Dokumen ini adalah sumber kebenaran tunggal (single source of truth) proyek. Setiap kali proyek ini dibahas dengan asisten AI mana pun, dokumen ini harus dibaca lebih dulu dan diikuti. Jika permintaan pengguna bertentangan dengan dokumen ini, tanyakan dahulu apakah dokumen perlu diubah, jangan diam-diam menyimpang.

Versi dokumen: 1.1
Bahasa kerja dengan pengguna: Bahasa Indonesia
Aturan format: tanpa emoji.

---

## 1. Konteks Tugas

- Mata kuliah: Teknologi Berkembang, topik Computer Vision (OpenCV).
- Tugas: Week 8, Progress Submission (ETS).
- Luaran yang dinilai:
  1. Progress Report PDF: 9 bagian, ditambah Appendix A (pembagian tugas), B (pernyataan orisinalitas), C (pernyataan GitHub).
  2. Slides PDF.
  3. Video demo MP4, 3 sampai 5 menit, aplikasi benar-benar berjalan.
  4. Source code ZIP.
- Penamaan file: `GroupXX_OpenCV_Progress_Report.pdf`, `GroupXX_OpenCV_Slides.pdf`, `GroupXX_OpenCV_Demo.mp4`, `GroupXX_OpenCV_Source_Code.zip`.
- Bobot penilaian yang ditekankan: Working Prototype 20 persen, Testing and Evaluation 15 persen, Problem Definition 10 persen, CV Method 10 persen, Dataset 10 persen, Originality dan Responsible Data Use 10 persen.
- Lingkungan pengembangan: Google Colab. Kamera ponsel dipakai sebagai webcam melalui Iriun Webcam.
- Pengguna adalah mahasiswa yang belum terlalu mendalami tugas ini, jadi penjelasan harus bertahap dan jujur tentang batasan.

## 2. Tujuan Prototype

Kamera ponsel (melalui Iriun di Colab) menampilkan gambar seolah-olah dilihat oleh penderita buta warna dikromat, dengan tiga mode yang dapat dipilih. Sebuah toggle "Solusi" menampilkan area yang warnanya melebur (sulit dibedakan oleh penderita) beserta bantuan untuk membedakannya.

Penting: pada dikromat, warna tidak "menghilang". Yang terjadi adalah dua warna berbeda tampak sama (confusion atau peleburan). Istilah "warna yang tak tampak" dalam permintaan awal pengguna diartikan sebagai warna yang melebur ini.

## 3. Keputusan yang Sudah Final

| No | Keputusan | Nilai |
|----|-----------|-------|
| 1 | Platform | Google Colab, kamera lewat browser (getUserMedia via JavaScript) dengan Iriun Webcam |
| 2 | Tampilan utama | Layar penuh selalu tampil sebagai simulasi penderita (100 persen), terlepas dari toggle |
| 3 | Toggle Solusi mati | Simulasi murni, tanpa kotak, tanpa koreksi |
| 4 | Toggle Solusi nyala | Kotak berlabel pada area yang melebur, ditambah daltonization lokal hanya di dalam area tersebut |
| 5 | Daltonization | Tidak pernah diterapkan pada seluruh gambar, hanya pada area yang melebur |
| 6 | Metode deteksi | Gabungan Metode A dan Metode B (lihat bagian 5) |
| 7 | Simulasi protanopia dan deuteranopia | Machado, Oliveira dan Fernandes (2009), severity 1.0, ruang linear RGB |
| 8 | Simulasi tritanopia | Brettel, Viénot dan Mollon (1997), karena Machado kurang andal untuk tritan |
| 9 | Jarak warna | CIEDE2000 (implementasi sendiri, sudah diverifikasi terhadap data Sharma et al. 2005) |
| 10 | Pengaman koreksi | Jika ΔE sesudah koreksi tidak lebih besar dari sebelum, koreksi dibatalkan dan kotak diberi tanda "tidak membaik" |
| 11 | Tombol tambahan | "Bandingkan normal" untuk melihat gambar asli sejenak, karena pengguna perlu memeriksa hasil |
| 12 | Validasi | Hanya validasi komputasional berbasis simulasi, bukan uji ke penderita sungguhan. Wajib ditulis demikian di laporan |

## 4. Ambang Batas ΔE (Direkomendasikan dan Ditetapkan)

Dua ambang dipakai dan harus ditaruh sebagai konstanta yang mudah diubah di bagian atas kode.

```
DE_NORMAL_MIN = 25    # dua warna dianggap jelas berbeda pada penglihatan normal
DE_SIM_MAX    = 10    # dua warna dianggap melebur setelah disimulasikan
```

Alasan pemilihan:
- 25 sebagai batas "jelas berbeda" cukup tinggi agar pasangan yang ditandai benar-benar relevan, tetapi tidak terlalu tinggi sehingga pasangan yang masih bermasalah bagi penderita terlewat.
- 10 sebagai batas "melebur" mencakup pasangan yang secara praktis sulit dibedakan. Nilai yang lebih ketat (misalnya 8) mengurangi jumlah kotak tetapi menambah pasangan yang terlewat, dan itu merugikan pengguna buta warna.
- Dua angka bulat ini mudah dijelaskan saat presentasi dan sudah dipakai pada analisis peta konfusi (hasil protanopia 8 pasangan, deuteranopia 8 pasangan, tritanopia 1 pasangan pada simulasi Machado).
- Ambang tidak boleh dikatakan sebagai standar baku. Ini pilihan rancangan yang dapat disetel. Pada pengujian, tampilkan sensitivitas hasil terhadap beberapa nilai ambang.

Parameter pendukung (nilai awal, dapat disetel):
- Jarak tetangga untuk perbandingan piksel: sekitar 6 piksel pada resolusi proses 320 px.
- Luas wilayah minimum: sekitar 0,4 persen dari frame.
- Resolusi pemrosesan: sisi terpanjang 320 px.

## 5. Arsitektur dan Alur per Frame

```
Frame kamera
   |
   +--> [1] Simulasi penderita (selalu tampil)
   |
   +--> [2] Metode A: cari batas yang melebur  -> kandidat wilayah dan kotak
   |
   +--> [3] Metode B: beri nama warna kedua sisi -> label "Warna1 | Warna2"
   |
   +--> [4] Pelacak (IoU) menstabilkan kotak dan label antar-frame
   |
   +--> [5] Daltonization lokal (hanya jika toggle Solusi nyala)
```

### 5.1 Tahap 1: Simulasi
- Protanopia dan deuteranopia: matriks Machado 2009 pada linear RGB.
- Tritanopia: Brettel 1997 (pustaka DaltonLens atau implementasi sendiri).
- Keluaran tahap ini adalah gambar yang ditampilkan di layar.

### 5.2 Tahap 2: Metode A (menentukan di mana)
- Konversi gambar asli dan gambar simulasi ke CIELAB.
- Untuk tiap piksel, bandingkan dengan tetangga pada beberapa arah. Tandai piksel bila ΔE2000 asli >= DE_NORMAL_MIN dan ΔE2000 simulasi <= DE_SIM_MAX.
- Bersihkan dengan morfologi (dilate atau close), kelompokkan dengan `connectedComponentsWithStats`, buang wilayah di bawah luas minimum, hasilkan kotak.
- Wajib memakai CIEDE2000, bukan jarak Euclid Lab. Pada percobaan awal, jarak sederhana membuat satu pasangan yang jelas melebur (merah dan hijau) terlewat.

### 5.3 Tahap 3: Metode B (menentukan apa)
- Untuk tiap kotak, ambil warna median di dalam wilayah dan warna median di cincin sekitarnya (dua sisi yang melebur).
- Cocokkan masing-masing ke palet 34 warna dengan CIEDE2000 untuk mendapatkan nama, misalnya "Merah | Hijau".
- ΔE sebelum dan sesudah dihitung dari warna nyata di kamera, bukan dari titik tengah palet. Dengan begitu warna di luar palet tetap terdeteksi, dan labelnya memakai nama palet terdekat.
- Label menampilkan angka, misalnya `dE 70 -> 6`, agar dapat diperiksa.

### 5.4 Tahap 4: Pelacak
- Pakai ulang `PelacakWarna` (IoU) dan `PenghalusWarna` dari kode webcam sebelumnya.
- Tahan nama label beberapa frame agar tidak berganti-ganti.

### 5.5 Tahap 5: Daltonization lokal
- Hanya di dalam wilayah yang melebur. Rumus geser-error per tipe:
  - Protan dan deutan: `[[0,0,0],[0.7,1,0],[0.7,0,1]]`
  - Tritan: `[[1,0,0.7],[0,1,0.7],[0,0,0]]`
- Setelah koreksi, simulasikan ulang dan hitung ΔE sesudah. Terapkan pengaman (keputusan 10).
- Catatan jujur: daltonization tidak memperbaiki semua pasangan. Pada evaluasi sebelumnya pasangan Ungu dan Biru justru memburuk. Itu sebabnya pengaman wajib ada.

## 6. Antarmuka Colab

- Tombol mode: Protanopia, Deuteranopia, Tritanopia.
- Toggle Solusi: mati atau nyala.
- Tombol "Bandingkan normal" (tahan atau klik untuk melihat gambar asli).
- Teks status kecil: FPS, jumlah wilayah yang melebur, mode aktif.
- Kamera: pilih perangkat Iriun lewat `pilihKamera` (pencocokan label /iriun/i). Bagian ini belum diuji di browser sungguhan.
- Gambar bergerak lewat browser dan Python, FPS yang diharapkan sekitar 5 sampai 10 (perkiraan, belum diukur dengan kamera sungguhan).

## 7. Aset yang Sudah Ada (Basis Kode)

Lokasi di lingkungan kerja asisten (dapat berbeda jika sesi baru):
- `/home/claude/color_cvd_colab.py`: skrip utama dengan penanda sel `# %%`.
- `/home/claude/Deteksi_Warna_CVD_Colab.ipynb`: notebook yang dihasilkan dari skrip di atas (27 sel).
- `/home/claude/sel22_v3.py`: perbaikan eksposur untuk sel foto (belum digabung ke notebook).

Isi pokok yang dapat dipakai ulang:
- Sel 1 sampai 4: impor, utilitas warna (`rgb_to_lab`, `lab_to_rgb`, `ciede2000`), palet 34 warna (`PALETTE`), penamaan warna.
- Sel 5 sampai 6: `detect_colors`, `load_image`, `show`, `dominant_colors`.
- Sel 7: `CVD_MATRICES`, `ERR_SHIFT`, `simulate_rgb01`, `daltonize_rgb01`, `simulate_cvd`, `daltonize`, `compare_cvd`.
- Sel 8: evaluasi pasangan dan benchmark.
- Sel 20 sampai 24: webcam (JS stream, kalibrasi putih, pelacak IoU, penghalus warna, overlay JS di atas video).

Yang belum digabung: perbaikan eksposur (`sel22_v3.py`) dan patch pemilihan kamera Iriun (Sel 21b).

## 8. Rencana Pengujian dan Evaluasi (untuk Bab Testing and Evaluation)

- Uji sintetis: pasangan warna yang diketahui melebur (misalnya merah dan hijau untuk deutan) dan pasangan kontrol yang tidak melebur (misalnya biru dan kuning). Ukur proporsi terdeteksi dan false positive.
- Uji foto nyata: buah, lampu lalu lintas, kabel warna, grafik berwarna. Catat jumlah wilayah yang ditandai dan apakah masuk akal.
- Evaluasi koreksi: persentase pasangan yang ΔE-nya naik setelah daltonization lokal, rata-rata kenaikan, dan jumlah yang dibatalkan oleh pengaman.
- Sensitivitas ambang: ulangi uji dengan beberapa nilai DE_NORMAL_MIN dan DE_SIM_MAX.
- Kinerja: FPS dan latensi per frame di Colab.
- Perbandingan simulasi: konsistensi implementasi dengan DaltonLens. Hasil sebelumnya (ΔE2000 rata-rata, maksimum): protanopia Machado 0,3 dan 0,8; deuteranopia Machado 0,4 dan 1,1; tritanopia Machado 0,3 dan 0,8 terhadap implementasi Machado di DaltonLens. Terhadap Brettel, tritanopia selisihnya rata-rata 6,5 dan maksimum 18,2, ini alasan memakai Brettel untuk tritan.

## 9. Batasan yang Wajib Dicantumkan Secara Jujur

1. Hanya simulasi komputasional, bukan uji klinis atau uji ke penderita.
2. Simulasi tritanopia paling tidak andal. Hasil kotak tritanopia dapat berbeda dari kenyataan klinis.
3. Daltonization tidak selalu memperbaiki dan dapat memburukkan pasangan tertentu.
4. Warna yang tertangkap kamera bergantung pada cahaya dan eksposur. Kalibrasi putih dan koreksi eksposur hanya mengurangi masalah, tidak menghilangkannya. Area yang sudah terbakar (putih) tidak dapat dipulihkan.
5. Akurasi 99,2 persen pada uji sintetis penamaan warna bukan akurasi dunia nyata.
6. Pendekatan dikromat penuh (severity 1.0). Penderita anomali (misalnya protanomali, deuteranomali) tidak dicakup pada versi ini.
7. Kecepatan Colab dan perilaku kamera Iriun di browser belum diuji pada perangkat sungguhan.
8. Prevalensi per tipe buta warna belum diverifikasi. Hanya klaim NEI "sekitar 1 dari 12 pria" yang sudah diverifikasi. Jangan mengutip angka per tipe tanpa membuka sumbernya.

## 10. Hasil Analisis Peta Konfusi (Ringkas)

Berdasarkan simulasi Machado pada palet 34 warna (ambang peleburan ΔE2000 di bawah 10):
- Protanopia: 24 dari 34 warna ikut menyatu. Contoh kelompok: Merah, Coklat, Hijau Tua; Oranye, Kuning Tua, Hijau; Biru Navy, Ungu Tua. Zona buta hue kira-kira 57 sampai 135 derajat dan sekitar 240 derajat.
- Deuteranopia: 27 dari 34 warna menyatu. Contoh: Merah, Oranye Tua, Hijau; Cyan, Biru Muda, Ungu Muda; Pink Pastel dan Putih. Zona buta hue kira-kira 57 sampai 69 derajat dan 351 sampai 366 derajat.
- Tritanopia: 15 dari 34 warna menyatu. Contoh: Hijau dengan Tosca; Cyan dengan Biru Muda; Pink dengan Oranye. Zona buta hue kira-kira 102 sampai 186 derajat dan sekitar 351 sampai 363 derajat. Status kurang andal.

## 11. Sumber Rujukan (Open Source dan Penelitian)

- Machado, Oliveira, Fernandes (2009): https://www.inf.ufrgs.br/~oliveira/pubs_files/CVD_Simulation/CVD_Simulation.html
- DaltonLens-Python (MIT): https://github.com/DaltonLens/DaltonLens-Python dan https://pypi.org/project/daltonlens
- Paket R dichromat: https://cran.r-project.org/web/packages/dichromat/refman/dichromat.html
- colour-science (machado2009): https://colour.readthedocs.io/en/v0.3.15/_modules/colour/blindness/machado2009.html
- colorspacious: https://colorspacious.readthedocs.io/en/latest/reference.html
- NEI: https://www.nei.nih.gov/eye-health-information/eye-conditions-and-diseases/color-blindness
- Belum terverifikasi isinya (pengguna harus membuka sendiri): https://www.ncbi.nlm.nih.gov/books/NBK11538/, https://pubmed.ncbi.nlm.nih.gov/22472762/, https://pubmed.ncbi.nlm.nih.gov/40769301/
- Sumber pembanding implementasi: repositori referensi GitHub "Color-Detection-using-OpenCV-Python" (hanya dibaca, tidak dijalankan). Perbedaannya dengan notebook ini: ia memakai jarak Manhattan RGB, notebook memakai CIEDE2000 (pada 3000 warna acak, 55 persen nama berbeda, ΔE rata-rata 4,9 berbanding 3,6).

## 12. Orisinalitas dan Penggunaan Data yang Bertanggung Jawab

- Metode (Machado, Brettel, daltonization geser-error) berasal dari literatur dan harus dikutip. Yang orisinal adalah penggabungan deteksi peleburan per piksel dengan penamaan palet, pengaman koreksi, serta antarmuka.
- Pustaka pihak ketiga dicantumkan beserta lisensinya (DaltonLens MIT, dichromat GPL-2, dan seterusnya).
- Dataset foto dibuat atau dikumpulkan sendiri oleh kelompok. Jangan memakai foto orang tanpa izin. Cantumkan sumber dan lisensi bila memakai data publik.
- Isi Appendix B (orisinalitas) dan C (GitHub) dengan jujur sesuai kontribusi nyata kelompok, termasuk bagian yang dibantu asisten AI.

## 13. Petunjuk untuk Asisten AI yang Membaca Dokumen Ini

1. Baca seluruh dokumen sebelum menjawab. Ikuti keputusan pada bagian 3 dan ambang pada bagian 4.
2. Jawab dalam Bahasa Indonesia. Jangan memakai emoji.
3. Kirim kode Colab per sel bila diminta, dengan penanda jelas urutan sel.
4. Jangan menyatakan sesuatu "sudah berjalan" bila belum diuji. Bedakan secara tegas antara yang diuji dengan kamera tiruan, yang diuji secara sintetis, dan yang belum diuji.
5. Jangan menyatakan klaim medis atau angka prevalensi tanpa sumber yang sudah dibuka.
6. Bila ada hal yang belum jelas, validasi dengan pengguna sebelum mengeksekusi (permintaan eksplisit pengguna).
7. Jangan mengubah ambang, arsitektur, atau pilihan metode tanpa persetujuan pengguna. Bila ada alasan kuat, ajukan sebagai usulan beserta alasannya.
8. Hasil yang dikirim ke pengguna harus dapat dipakai langsung untuk laporan, slide, video demo, dan source code, serta mengikuti kriteria penilaian pada bagian 1.
9. Saat mengusulkan perubahan pada dokumen ini, tampilkan perubahan sebagai daftar yang jelas dan naikkan nomor versi.

## 14. Status dan Langkah Berikutnya

Selesai:
- Peta konfusi tiga tipe pada palet 34 warna (gambar peta hue).
- Demo visual perbandingan Metode A dan Metode B.
- Notebook dasar (deteksi nama warna, simulasi, daltonization, evaluasi, webcam multi-warna).
- Sel Kamera Buta Warna (Sel 25 sampai 29 di notebook): simulasi 3 tipe, deteksi peleburan Metode A + B, pelacak, daltonization lokal dengan pengaman, antarmuka JS (tombol mode, toggle Solusi, tahan untuk melihat normal), tes tanpa kamera pada gambar.
- Perbaikan eksposur (Sel 22) sudah digabung ke notebook.

Rincian implementasi Sel 25 sampai 29 (catatan versi 1.1):
- Tritanopia memakai Brettel 1997 dengan konstanta (dua matriks proyeksi + normal bidang pemisah, ruang linear RGB) yang diturunkan dari DaltonLens dan ditanam di kode, tanpa dependensi. Cocok dengan DaltonLens dengan selisih maksimum 1 tingkat (uint8).
- Metode A bekerja pada resolusi proses 240 px sisi terpanjang, jarak tetangga 5 piksel, blur Gauss 5. Saringan cepat dE76 >= 15 hanya mempercepat, bukan ambang keputusan; keputusan memakai CIEDE2000 dengan DE_NORMAL_MIN dan DE_SIM_MAX.
- Metode B: tiap wilayah memilih warna median sisi terang dan sisi gelap, lalu diberi nama palet terdekat. Wilayah dengan "kemurnian" rendah (kurang dari 0,20) dibuang untuk menyingkirkan tepi antialiasing dan tekstur.
- Daltonization lokal: hanya piksel yang dekat dengan dua warna melebur (jarak Lab kurang dari 20) dan terhubung dengan kotak. Parameter `KEKUATAN_COBA = (1.0,)` (kekuatan 1,0 sesuai spesifikasi). Opsi `(1.0, 1.5, 2.0)` mencoba kekuatan lebih besar dan memilih yang terbaik; belum diaktifkan, menunggu keputusan pengguna.
- Hasil uji pada pasangan palet yang melebur (dE normal >= 25, dE simulasi <= 10), kekuatan 1,0: protanopia 12 dari 12 pasangan membaik, deuteranopia 10 dari 10, tritanopia 12 dari 13. Ini diukur pada warna palet murni, bukan pada kamera sungguhan.

Berikutnya (menunggu persetujuan pengguna):
1. Uji di Colab dengan kamera Iriun sungguhan: FPS, pemilihan kamera, hasil pada benda nyata. Belum pernah dijalankan di browser sungguhan.
2. Putuskan apakah `KEKUATAN_COBA` diperluas.
3. Sel evaluasi dan ekspor hasil untuk laporan (tabel, gambar sebelum dan sesudah) pada foto nyata.
4. Susun kerangka laporan 9 bagian, slide, dan skrip video demo berdasarkan dokumen ini.
