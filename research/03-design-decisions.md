# Design Decisions: Supply Chain Security Pipeline

Dokumen ini mencatat setiap keputusan desain arsitektur yang diambil dalam proyek **"Supply Chain Security: Artifact Integrity Verification in CI/CD Pipeline"**, disertai justifikasi berbasis temuan ilmiah dari kedua paper referensi.

---

## Keputusan 1: Format SBOM — CycloneDX sebagai Standar Tunggal

### Keputusan
Seluruh pipeline menggunakan format **CycloneDX** secara konsisten dari tahap *generation* hingga *analysis*. Format SPDX tidak digunakan dalam alur utama.

### Justifikasi
Kami memilih CycloneDX sebagai format tunggal karena O'Donoghue dkk. (2024) menunjukkan bahwa ambiguitas penempatan kolom data — seperti `supplier name` — berbeda antara implementasi CycloneDX dan SPDX, menyebabkan kegagalan pencocokan CPE pada database NVD yang menurunkan akurasi deteksi kerentanan secara konsisten. Dalam konteks pipeline kami, ini berarti menggunakan satu format secara konsisten dari *generation* hingga *analysis* mengeliminasi satu sumber kegagalan yang tidak perlu, tanpa mengorbankan cakupan deteksi.

### Alternatif yang Ditolak
- **SPDX**: Ditolak karena dukungan native dari Grype (Anchore) lebih kuat untuk CycloneDX, dan ekosistem toolchain kami lebih stabil pada format ini.
- **Dual format (CycloneDX + SPDX)**: Ditolak karena menambah kompleksitas pipeline tanpa keuntungan deteksi yang signifikan berdasarkan temuan paper.

---

## Keputusan 2: Strategi Scanning — Multi-Tool Vendor-Aligned Paralel

### Keputusan
Pipeline menjalankan **dua pasang toolchain** secara paralel dalam job terpisah:
- **Jalur Anchore**: Syft (generate SBOM) → Grype (scan CVE)
- **Jalur Aqua Security**: Trivy (generate SBOM) → Trivy (scan CVE)

Hasil dari kedua jalur digabungkan (*merged*) sebelum keputusan *gate* diambil.

### Justifikasi
Kami memilih strategi dual-toolchain vendor-aligned karena O'Donoghue dkk. (2024) menunjukkan bahwa kombinasi alat dari vendor yang sama menghasilkan deteksi CVE secara signifikan lebih tinggi dibanding kombinasi lintas vendor — fenomena yang mereka sebut *Vendor Affinity Effect*, dibuktikan melalui analisis statistik *bootstrapping* (20.000 sampel) dan *effect size* Cohen's D pada 2.313 container image Docker Hub. Dalam konteks pipeline kami, ini berarti menjalankan jalur Anchore (Syft→Grype) dan Aqua Security (Trivy→Trivy) secara paralel menghasilkan *union* dari kedua spektrum deteksi, sehingga CVE yang hanya terdeteksi oleh database internal Anchore dan CVE yang hanya terdeteksi oleh database internal Aqua Security keduanya masuk ke dalam laporan akhir.

### Alternatif yang Ditolak
- **Single-tool scanning (hanya Trivy atau hanya Syft+Grype)**: Ditolak karena terbukti meninggalkan blind spot signifikan berdasarkan paper.
- **Cross-vendor scanning (Syft → Trivy atau Trivy → Grype)**: Ditolak secara eksplisit karena paper membuktikan kombinasi ini adalah yang paling buruk dalam mendeteksi kerentanan.

---

## Keputusan 3: Metode Signing — Keyless Signing via Cosign/Sigstore

### Keputusan
Container image ditandatangani menggunakan **Cosign** dengan metode *keyless signing* berbasis OIDC token dari GitHub Actions runner. Tidak ada *private key* jangka panjang yang disimpan di GitHub Secrets.

### Justifikasi
Kami memilih keyless signing via Cosign/Sigstore karena Newman dkk. (2022) menunjukkan bahwa angka adopsi penandatanganan artefak di repositori publik sangat rendah (di bawah 5% pada PyPI dan RubyGems), dan hambatan utamanya adalah beban manajemen kunci privat jangka panjang yang membutuhkan pembuatan, distribusi, penyimpanan, dan rotasi kunci secara manual. Dalam konteks pipeline kami, ini berarti menggunakan *ephemeral key pair* berbasis OIDC token dari GitHub Actions runner — pasangan kunci dibuat otomatis oleh Fulcio CA, digunakan selama ±10 menit, lalu dimusnahkan — sehingga tidak ada kunci privat yang perlu disimpan di GitHub Secrets, mengeliminasi risiko *key compromise* sepenuhnya.

### Alternatif yang Ditolak
- **PGP/GPG dengan long-lived key**: Ditolak karena Newman dkk. (2022) membuktikan pendekatan ini menciptakan risiko key compromise dan friksi operasional yang secara langsung menekan angka adopsi di bawah 5%.
- **Cosign dengan key pair yang disimpan di GitHub Secrets**: Ditolak karena meskipun lebih sederhana dari PGP, tetap mempertahankan risiko *secret leakage* dan memerlukan rotasi manual.

---

## Keputusan 4: Transparency Logging — Rekor (Sigstore)

### Keputusan
Setiap event penandatanganan dicatat secara otomatis ke **Rekor**, *transparency log* publik milik Sigstore yang bersifat *append-only*.

### Justifikasi
Kami memilih Rekor sebagai transparency log karena Newman dkk. (2022) menunjukkan bahwa ketiadaan rekam jejak publik yang bersifat *append-only* membuat insiden seperti penerbitan sertifikat palsu oleh CA yang korup tidak dapat dideteksi atau diaudit secara real-time oleh pihak ketiga mana pun. Dalam konteks pipeline kami, ini berarti setiap event penandatanganan secara otomatis menghasilkan entri di Rekor yang dapat diverifikasi secara kriptografis menggunakan `cosign verify` oleh siapa saja — auditor, operator deployment, atau pengguna downstream — tanpa harus mempercayai operator pipeline secara langsung.

### Alternatif yang Ditolak
- **Private/self-hosted transparency log**: Ditolak karena menghilangkan nilai akuntabilitas publik. Log privat hanya dapat diaudit oleh operator yang sama, menciptakan titik kepercayaan tunggal yang rentan.
- **Tidak menggunakan transparency log**: Ditolak karena melanggar prinsip dasar non-repudiation yang menjadi tujuan proyek ini.

---

## Keputusan 5: Arsitektur Job GitHub Actions

### Keputusan
Pipeline dibagi menjadi lima job berurutan dengan dependensi eksplisit:

```
build → [sbom-anchore, sbom-aqua] (paralel) → sign-and-verify
                                             ↓
                                    [scan-anchore, scan-aqua] (paralel)
                                             ↓
                                       gate (merge results)
```

| Job | Alat | Output |
|:----|:-----|:-------|
| `build` | Docker | Container image + push ke GHCR |
| `sbom-anchore` | Syft | `sbom-syft.cyclonedx.json` |
| `sbom-aqua` | Trivy | `sbom-trivy.cyclonedx.json` |
| `scan-anchore` | Grype | `report-grype.json` |
| `scan-aqua` | Trivy | `report-trivy.json` |
| `sign-and-verify` | Cosign | Signature + Rekor entry |

### Justifikasi
- **Paralelisasi job SBOM dan scan**: Memotong waktu total pipeline karena kedua jalur scanning tidak saling bergantung.
- **Job `gate` terpisah**: Memisahkan logika pengambilan keputusan (apakah ada CVE kritis yang harus memblokir deployment) dari logika scanning, memudahkan kustomisasi threshold.
- **Sign setelah scan**: Penandatanganan hanya dilakukan setelah image melewati gate, memastikan hanya image yang sudah diverifikasi keamanannya yang mendapat signature.

---

## Keputusan 6: Sample Application Target

### Keputusan
Menggunakan aplikasi **Node.js (Express)** sederhana sebagai target scanning, bukan container base image kosong.

### Justifikasi
Container dengan dependensi aplikasi nyata (npm packages) menghasilkan SBOM yang lebih representatif dan demonstrasi deteksi CVE yang lebih informatif. Scanning image `alpine` kosong tidak akan menunjukkan perbedaan antar toolchain secara bermakna.
