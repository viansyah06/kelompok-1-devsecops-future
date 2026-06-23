# State-of-the-Art: Supply Chain Security & Artifact Integrity

Dokumen ini merangkum posisi teknologi mutakhir (*State-of-the-Art*) yang diadopsi industri saat ini dalam menjaga keamanan *software supply chain*, mengidentifikasi *gap* atau kekurangan yang masih ada berdasarkan literatur ilmiah, serta memetakan posisi proyek **"Supply Chain Security: Artifact Integrity Verification in CI/CD Pipeline"** sebagai bentuk *enhancement*.

---

## 1. Manifes Komponen Perangkat Lunak (Software Bill of Materials - SBOM)

### Apa yang Sudah Ada:
* **Automated Open-Source Tools**: Industri saat ini mengandalkan *SBOM generation tools* canggih seperti **Syft** (oleh Anchore) dan **Trivy** (oleh Aqua Security) untuk membedah lapisan *container image* (Docker) dan mengekstrak daftar dependensi, *libraries*, serta lisensi yang digunakan.
* **Global Standard Formats**: Terdapat dua format standar mesin yang diakui secara internasional untuk *data interoperability*, yaitu **CycloneDX** (OWASP) dan **SPDX** (Linux Foundation).
* **Downstream Scanners**: *SBOM analysis tools* seperti **Grype** dan **CVE-bin-tool** digunakan secara luas untuk membaca manifes SBOM tersebut dan mencocokkannya dengan basis data *vulnerability* seperti *National Vulnerability Database* (NVD) melalui skema *CPE matching*.

### Apa yang Masih Kurang (Berdasarkan Paper O'Donoghue dkk., 2024):
* **Ilusi Konsistensi**: Industri berasumsi bahwa hasil *vulnerability scanning* akan sama terlepas dari *tools* apa yang digunakan untuk membuat SBOM. Faktanya, terdapat variabilitas kuantitas laporan CVE yang sangat tinggi akibat perbedaan algoritma pencarian masing-masing vendor.
* **Blind Spot Akibat Vendor Affinity**: *Analysis tools* hilir sering kali gagal membaca entri paket secara akurat jika format SBOM dikonversi atau dibuat oleh vendor yang berbeda (misalnya membuat dengan Syft tetapi dianalisis dengan Trivy). Ambiguitas penempatan kolom informasi (seperti *supplier name*) membuat pencocokan skema CPE ke *vulnerability database* gagal, yang berujung pada tingginya angka kegagalan pemrosesan (*dropout rate* mencapai 43,7%).

---

## 2. Penandatanganan Kriptografi & Validasi Integritas Artefak

### Apa yang Sudah Ada:
* **Traditional Public Key Infrastructure (PKI)**: Penggunaan PGP/GPG atau sertifikat X.509 berbasis korporasi untuk menandatangani *container image* sebelum didorong ke *registry* (seperti Docker Hub atau GitHub Container Registry).
* **Secret Management**: Memanfaatkan fitur GitHub Secrets atau HashiCorp Vault untuk menyimpan *private key* penandatanganan di dalam lingkungan CI/CD pipeline.

### Apa yang Masih Kurang (Berdasarkan Paper Newman dkk., 2022):
* **Beban Manajemen Kunci Jangka Panjang**: Developer dibebani tanggung jawab untuk menjaga, merotasi, dan mengamankan *private key* mereka secara manual. Ini memicu friksi operasional yang membuat angka adopsi penandatanganan kode di repositori publik sangat rendah (di bawah 5%).
* **Risiko Key Compromise**: Jika kunci privat jangka panjang (*long-lived keys*) yang disimpan di lokal komputer developer atau di dalam *secret manager* CI/CD berhasil dicuri, penyerang dapat menandatangani *malware* secara sah tanpa bisa dideteksi secara instan.
* **Ketiadaan Rekam Jejak Publik Terpusat**: Penandatanganan tradisional tidak terikat pada *transparency log* publik yang bersifat *append-only*, sehingga jika ada Certificate Authority (CA) yang korup menerbitkan sertifikat palsu, tidak ada mekanisme audit eksternal yang dapat mendeteksinya secara *real-time*.

---

## 3. Posisi dan Peningkatan Proyek Kami (Our Enhancement)

Proyek **"Supply Chain Security: Artifact Integrity Verification in CI/CD Pipeline"** mengintegrasikan teknologi mutakhir untuk menyelesaikan kekurangan-kekurangan di atas melalui arsitektur pipeline berikut:

### Bagaimana Proyek Kami Mengatasi Kekurangan Tersebut:

| Area Fokus | Kekurangan di Industri Saat Ini | Solusi Implementasi Proyek Kami |
| :--- | :--- | :--- |
| **Akurasi Deteksi CVE** | *Blind spot* deteksi karena ketidakcocokan vendor *tools* SBOM (*Vendor Affinity*). | **Multi-Tool & Vendor-Aligned Scanning**: Menjalankan skrip paralel untuk menghasilkan SBOM dari Syft dan Trivy, lalu memindainya menggunakan pasangannya yang optimal (Syft dipindai Grype; Trivy dipindai Trivy) untuk menangkap spektrum kerentanan secara maksimal. |
| **Integritas Format** | Kegagalan deteksi akibat konversi format (CycloneDX <-> SPDX) di tengah pipeline. | **Native CycloneDX Standard**: Menetapkan format CycloneDX asli secara konsisten di seluruh rantai proses tanpa konversi transasional untuk memitigasi kegagalan pencocokan CPE pada database NVD. |
| **Keamanan Kunci** | Risiko kebocoran kunci privat jangka panjang (*long-lived private keys*). | **Keyless Signing via Cosign/Sigstore**: Menggunakan identitas OIDC berumur pendek (10 menit) dari GitHub Actions runner untuk menandatangani citra. *Private key* langsung dimusnahkan setelah digunakan, menghilangkan risiko pencurian kunci. |
| **Akuntabilitas Audit** | Sulitnya mendeteksi pemalsuan sertifikat atau manipulasi pipeline secara eksternal. | **Public Transparency Logging (Rekor)**: Setiap tanda tangan dan manifes integritas artefak dicatat secara otomatis ke *transparency log* publik Rekor yang bersifat *append-only* untuk menjamin akuntabilitas audit. |
