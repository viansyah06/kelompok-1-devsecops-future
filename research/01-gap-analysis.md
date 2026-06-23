# Gap Analysis: Artifact Integrity Verification in CI/CD Pipeline

Dokumen ini merinci hasil analisis celah keamanan (*gap analysis*) dalam ekosistem DevSecOps modern, khususnya terkait manajemen kerentanan dan penandatanganan artefak pada rantai pasok perangkat lunak (*software supply chain*). Analisis ini didasarkan pada temuan empiris dari literatur ilmiah terkini untuk menjustifikasi urgensi pengembangan proyek **"Supply Chain Security: Artifact Integrity Verification in CI/CD Pipeline"**.

---

## Celah 1: Fenomena *Vendor Affinity* dan Tingginya Variabilitas Deteksi CVE pada Ekosistem SBOM

### 1. Kondisi Saat Ini (State of Practice)
Saat ini, organisasi dan praktisi DevSecOps mulai patuh pada regulasi global (seperti U.S. Executive Order 14028) dengan menghasilkan *Software Bill of Materials* (SBOM) menggunakan alat-alat sumber terbuka populer seperti Syft atau Trivy. Manifes komponen ini kemudian didorong ke dalam alat analisis pihak ketiga untuk mendeteksi kerentanan (*vulnerability scanning*). Banyak tim berasumsi bahwa memindai SBOM yang dihasilkan oleh alat mana pun akan memberikan visualisasi keamanan yang setara dan konsisten.

### 2. Masalah Nyata (*The Gap*)
Eksperimen berskala besar yang dilakukan oleh O'Donoghue dkk. (2024) terhadap 2.313 citra kontainer Docker membuktikan adanya **variabilitas dan ketidakpastian yang sangat tinggi** pada hasil pelaporan kerentanan downstream. Terdapat celah buta (*blind spot*) keamanan yang masif akibat dua faktor utama:

* **Efek Hubungan Vendor (Vendor Affinity Effect)**: Jumlah kerentanan yang terdeteksi melonjak drastis jika alat pembuat SBOM (*generation tool*) dan alat pemindai kerentanan (*analysis tool*) berasal dari vendor pengembang yang sama. Kombinasi *tooling* dari vendor Anchore (Syft + Grype) menghasilkan median laporan kerentanan tertinggi, disusul oleh vendor Aqua Security (Trivy + Trivy). Namun, ketika menggunakan alat dari vendor silang (misalnya SBOM dibuat dengan Syft tetapi dianalisis menggunakan Trivy atau CVE-bin-tool), jumlah kerentanan terdeteksi menurun drastis secara konsisten. Hal ini membuktikan bahwa **banyak kerentanan kritis terlewat hanya karena ketidakcocokan vendor alat di dalam pipeline**.
* **Inkonsistensi dan Ambiguitas Format Data**: Meskipun format standar seperti CycloneDX dan SPDX diadopsi secara luas, terdapat perbedaan implementasi pengisian kolom data (seperti lokasi *supplier information*). Inkonsistensi ini menyebabkan alat pemindai hilir gagal melakukan pencocokan skema *Common Platform Enumeration* (CPE) terhadap basis data kerentanan nasional (NVD), sehingga menurunkan tingkat akurasi pelaporan risiko.

### 3. Dampak Jika Dibiarkan
Tim keamanan akan memiliki rasa aman yang palsu (*false sense of security*). Jika pipeline hanya mengandalkan satu kombinasi alat yang tidak cocok (misalnya menghasilkan SBOM dengan Syft lalu memindainya dengan Trivy), ratusan celah keamanan nyata (*true positives*) berpotensi lolos ke lingkungan produksi cluster.

---

## Celah 2: Hambatan Adopsi Tanda Tangan Artefak Akibat Kompleksitas Manajemen Kunci Kriptografi

### 1. Kondisi Saat Ini (State of Practice)
Penandatanganan digital (*code/artifact signing*) diakui secara luas sebagai satu-satunya cara untuk membuktikan chain-of-custody dan memastikan bahwa kode yang berjalan di server tidak dimodifikasi oleh aktor jahat di tengah jalan. Industri memiliki berbagai perkakas enkripsi tradisional untuk melakukan hal ini.

### 2. Masalah Nyata (*The Gap*)
Berdasarkan investigasi Newman dkk. (2022), tingkat adopsi penandatanganan artefak pada repositori publik sangat rendah (misalnya di bawah 5% pada PyPI dan RubyGems). Rendahnya adopsi ini disebabkan oleh celah friksi operasional dan teknis dari arsitektur konvensional:

* **Beban Manajemen Kunci Kriptografi Jangka Panjang**: Skema tradisional seperti PGP/GPG mewajibkan pengembang untuk membuat, mendistribusikan, mengecek secara manual, serta menjaga keamanan kunci privat mereka secara mandiri dalam jangka panjang. Hal ini menciptakan titik kerentanan baru (*key compromise*). Jika kunci privat yang disimpan di komputer lokal pengembang atau di dalam *secret manager* CI/CD bocor, penyerang dapat memalsukan tanda tangan artefak berbahaya secara sah.
* **Keterikatan Waktu Validitas Sertifikat**: Pendekatan penandatanganan lama mengikat masa berlaku artefak dengan masa berlaku sertifikat. Hal ini menuntut proses penandatanganan ulang secara berkala (*periodic re-signing*). Jika kunci privat harus terus dibawa ke dalam jaringan (*online*) untuk proses rotasi atau tanda tangan ulang, risiko intersepsi oleh penyerang menjadi jauh lebih besar.
* **Ketiadaan Transparansi Publik yang Akuntabel**: Penandatanganan konvensional tidak mencatat riwayat klaim identitas ke dalam log publik yang bersifat *append-only* secara otomatis. Akibatnya, jika terjadi kompromi pada akun *maintainer* atau terjadi penerbitan sertifikat palsu oleh Certificate Authority (CA) yang korup, insiden tersebut sulit dideteksi dan diaudit secara *real-time* oleh pihak ketiga.

### 3. Dampak Jika Dibiarkan
Otomatisasi pipeline CI/CD rentan terhadap serangan manipulasi artefak di "mil terakhir" (*the last mile: software delivery pipeline*). Aktor jahat yang berhasil menyusup ke server repositori kontainer dapat menukar citra aplikasi dengan citra yang telah disisipi <i>backdoor</i> tanpa terdeteksi, karena sistem downstream tidak memiliki mekanisme verifikasi integritas yang mudah dijalankan.

---

## Kesimpulan & Fokus Solusi Projek
Projek **"Supply Chain Security: Artifact Integrity Verification in CI/CD Pipeline"** hadir untuk menjembatani kedua celah di atas secara simultan dengan mengimplementasikan:
1. **Multi-Tool Vendor Affinity Scanning**: Memerangi *blind spot* SBOM dengan menjalankan pemindaian paralel menggunakan ekosistem alat yang selaras (Syft+Grype dan Trivy+Trivy) demi menangkap cakupan kerentanan secara maksimal[.
2. **Keyless Signing & Transparency Logging via Sigstore**: Menghilangkan beban manajemen kunci jangka panjang dengan menerapkan kunci jangka pendek (*ephemeral keys*) berbasis OIDC GitHub Actions, dan mencatat manifes integritas tersebut ke dalam transparansi log publik Rekor.
