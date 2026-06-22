# Reading Notes: Sigstore: Software Signing for Everybody

## 1. Info Paper
* **Judul**: Sigstore: Software Signing for Everybody
* **Penulis**: Zachary Newman, John Speed Meyers, Santiago Torres-Arias
* **Tahun Publikasi**: 2022
* **Venue Publikasi**: Proceedings of the 2022 ACM SIGSAC Conference on Computer and Communications Security (CCS '22) - ACM Digital Library

---

## 2. Klaim Utama Paper & Metodologi Pembuktian
* **Klaim Utama**: 
  Metode penandatanganan kode tradisional (seperti PGP/GPG) memiliki tingkat adopsi yang sangat rendah karena kompleksitas manajemen kunci kriptografi jangka panjang oleh pengembang. Sigstore hadir sebagai infrastruktur terbuka yang membuat penandatanganan artefak perangkat lunak menjadi ubiquitously mudah dan "invisible" melalui konsep **keyless signing**. Konsep ini memisahkan masa berlaku sertifikat dengan masa berlaku artefak tanpa membebani pengembang dengan pengelolaan kunci privat.
* **Metodologi Pembuktian**:
  1. **Desain Arsitektur Sistem**: Membangun ekosistem tiga komponen utama: **Fulcio** (CA yang menerbitkan sertifikat jangka pendek 10 menit berdasarkan identitas OIDC), **Rekor** (transparansi log untuk tanda tangan artefak), dan **Cosign** (alat klien untuk menandatangani citra kontainer).
  2. **Studi Kasus & Wawancara**: Melakukan wawancara semi-terstruktur (30–60 menit) terhadap 7 praktisi/pengembang senior yang mengintegrasikan Sigstore ke proyek skala besar (Kubernetes, Shopify, RubyGems, Java client) untuk memvalidasi aspek usabilitas dan efektivitas keamanan.
  3. **Pengujian Kinerja & Skalabilitas (Micro-benchmarking)**: Mengukur latensi waktu proses penandatanganan/verifikasi lokal dan jarak jauh, serta melakukan pengujian beban (*load test*) masif pada Agustus 2021 dengan menyuntikkan >347.000 entri dalam satu hari ke log Rekor untuk membuktikan kesiapan production.

---

## 3. Temuan Kunci 
* **Kunci Ephemeral & OIDC**: Pengembang tidak perlu lagi menyimpan kunci privat jangka panjang. Alur kerja memanfaatkan token OIDC dari penyedia identitas (seperti GitHub Actions) untuk membuktikan kepemilikan akun. Kunci privat dibuat secara instan, digunakan sekali untuk menandatangani, lalu langsung dihancurkan.
* **Timestamping via Rekor**: Karena sertifikat Fulcio hanya berlaku 10 menit, validitas tanda tangan di masa depan dijamin oleh log transparansi Rekor. Rekor memberikan *Signed Certificate Timestamp* (SCT) yang membuktikan secara kriptografis bahwa tanda tangan dibuat saat sertifikat masih aktif.
* **Otomatisasi Pipeline (CI/CD)**: Integrasi terbaik dicapai melalui GitHub Actions Runner. Token OIDC bawaan runner mengaitkan kode biner hasil build langsung dengan komit hash (*source code provenance*), menutup celah serangan manipulasi artefak di luar repositori publik.
* **Pergeseran Paradigma Keamanan**: Hasil wawancara menunjukkan pengembang lebih memilih Sigstore karena berhasil "mengubah masalah manajemen kunci menjadi masalah manajemen identitas", serta memaksa penyerang untuk "bergerak di ruang terbuka" karena semua jejak tercatat di log publik.

---

## 4. Asumsi dan Keterbatasan Paper
* **Ketergantungan pada Penyedia OIDC**: Sigstore mengasumsikan bahwa penyedia identitas OIDC (Google, GitHub, Microsoft) sepenuhnya aman. Jika akun pengembang atau penyedia OIDC itu sendiri diambil alih oleh penyerang, Sigstore tidak dapat mencegah penerbitan sertifikat palsu pada turn tersebut.
* **Trade-off Transparansi vs Privasi**: Karena semua sertifikat dipublikasikan ke log yang tidak bisa diubah (*append-only*), data pribadi seperti alamat email pengembang akan terekspos secara publik selamanya. Hal ini dapat memicu kekhawatiran privasi bagi pengembang yang ingin tetap anonim.
* **Masalah Skalabilitas pada Paket Kompleks**: Pada ekosistem tertentu (seperti Java/Maven), satu rilis package dapat menghasilkan ribuan artefak biner sekaligus. Melakukan alur login OIDC individual untuk setiap file akan menyebabkan *overhead* performa yang luar biasa besar dan membebani server log.

---

## 5. Pemikiran Kritis dan Poin yang Diragukan
* **Kelemahan Deteksi Manusia terhadap Typosquatting**: Paper menyebutkan bahwa identitas email/repositori lebih mudah dibaca manusia daripada string kunci publik. Namun, saya melihat ini sebagai celah baru: manusia sangat rentan terhadap serangan *typosquatting* (misal: verifikator tidak menyadari perbedaan antara `user123@github.com` dan `userl23@github.com` saat memvalidasi tanda tangan). Otomatisasi kebijakan (*policy automation*) mutlak diperlukan dan tidak bisa sekadar mengandalkan inspeksi manual.
* **Risiko Denial of Service (DoS) Akibat Kegagalan Log Jarak Jauh**: Implementasi penandatanganan dan verifikasi online sangat bergantung pada ketersediaan koneksi ke Fulcio dan Rekor. Berdasarkan micro-benchmarking di paper, panggilan remote memakan waktu dominan (~300-500ms). Jika infrastruktur pusat Sigstore mengalami kelangkaan resource atau tumbang, seluruh pipeline rilis global dapat terhenti (DoS).
* **Efektivitas Validasi Tanpa Framework Tambahan (TUF)**: Paper mengakui bahwa Sigstore hanya membuktikan *siapa* yang menandatangani, bukan *apakah* orang itu berhak merilis package tersebut. Tanpa integrasi dengan framework pemetaan seperti TUF (*The Update Framework*), tanda tangan Sigstore menjadi kurang bermakna karena penyerang dengan akun valid dari repositori lain tetap bisa menandatangani paket berbahaya secara legal di sistem Rekor.
