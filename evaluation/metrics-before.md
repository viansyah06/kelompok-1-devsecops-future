# Baseline Metrics: Kondisi Sebelum Implementasi Pipeline

Dokumen ini mencatat kondisi keamanan *software supply chain* **sebelum** pipeline DevSecOps diimplementasikan. Baseline ini digunakan sebagai titik pembanding untuk mengukur efektivitas solusi yang dibangun.

---

## Kondisi Awal (Pre-Implementation)

### Proses Deployment yang Ada
Sebelum proyek ini, alur deployment container ke lingkungan target dilakukan secara manual dengan langkah-langkah berikut:

| Langkah | Alat | Kelemahan |
|:--------|:-----|:----------|
| Build image | `docker build` secara lokal | Tidak ada audit trail build environment |
| Push ke registry | `docker push` ke Docker Hub | Tidak ada otentikasi integritas image |
| Deploy | `kubectl apply` atau `docker run` | Image yang di-deploy tidak diverifikasi keamanannya |

### Tidak Ada SBOM Generation
- Tidak ada proses inventarisasi komponen perangkat lunak (SBOM) yang dijalankan secara otomatis.
- Tim tidak mengetahui dependensi transitif yang terdapat di dalam container image.
- Kepatuhan terhadap regulasi seperti U.S. Executive Order 14028 tidak terpenuhi.

### Tidak Ada Vulnerability Scanning Otomatis
- Pemindaian kerentanan hanya dilakukan ad-hoc (sesekali, manual) jika diingat oleh developer.
- Tidak ada *gate* otomatis yang memblokir deployment image dengan kerentanan kritis.
- Tidak ada laporan historis yang dapat diaudit.

### Tidak Ada Artifact Signing
- Container image didorong ke registry tanpa tanda tangan kriptografis.
- Tidak ada mekanisme bagi konsumen downstream (deployment environment) untuk memverifikasi bahwa image yang di-deploy identik dengan yang di-build.
- Serangan *image substitution* pada registry tidak dapat terdeteksi.

---

## Simulasi Baseline: Scanning Image tanpa Pipeline

Untuk memberikan data kuantitatif yang dapat dibandingkan, dilakukan scanning manual satu kali pada image `node:18-alpine` sebagai representasi base image sebelum optimasi.

### Metodologi Baseline
- **Target**: `node:18-alpine` (base image tanpa aplikasi)
- **Alat (satu kali, satu tool)**: Trivy image scan langsung (bukan via SBOM)
- **Pendekatan**: Single-tool, tanpa vendor alignment, tanpa CycloneDX normalization

### Hasil Baseline (Estimasi dari Pendekatan Single-Tool)

Berdasarkan karakteristik yang didokumentasikan oleh O'Donoghue dkk. (2024):

| Metrik | Nilai Baseline |
|:-------|:--------------|
| Alat yang digunakan | 1 (Trivy image scan langsung) |
| Format SBOM | Tidak ada (scan langsung) |
| Jumlah CVE terdeteksi | Bergantung pada satu database vendor |
| Potensi blind spot | Tinggi — CVE yang hanya ada di database Anchore/Grype tidak terdeteksi |
| Artifact signing | ❌ Tidak ada |
| Transparency logging | ❌ Tidak ada |
| Waktu eksekusi | Manual, tidak terukur |
| Audit trail | ❌ Tidak ada |

### Implikasi Blind Spot (dari Paper O'Donoghue dkk., 2024)
Paper membuktikan bahwa ketika hanya menggunakan **satu** toolchain (misalnya hanya Trivy scan langsung), deteksi kerentanan dapat berkurang drastis dibanding pendekatan vendor-aligned multi-tool:

- Cross-vendor combination (mis. Syft SBOM → Trivy scan) menghasilkan deteksi CVE **jauh lebih rendah** dibanding same-vendor combination.
- Dropout rate dalam pemrosesan SBOM lintas vendor mencapai **43,7%**, artinya hampir setengah dari CVE yang seharusnya terdeteksi hilang karena kegagalan pencocokan CPE.

---

## Kesimpulan Baseline

Kondisi sebelum implementasi memiliki tiga kelemahan fundamental:

1. **Blind spot kerentanan**: Tidak ada systematic scanning, sehingga CVE bisa lolos ke produksi tanpa terdeteksi sama sekali.
2. **Tidak ada integritas artefak**: Image yang di-deploy tidak dapat diverifikasi keasliannya, membuka vektor serangan substitusi artefak.
3. **Tidak ada audit trail**: Tidak ada catatan publik yang dapat diaudit jika terjadi insiden keamanan di supply chain.
