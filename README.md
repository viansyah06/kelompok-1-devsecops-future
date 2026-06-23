# Supply Chain Security: Artifact Integrity Verification in CI/CD Pipeline

**Kelompok 1 | Final Project DevOps**

| Nama | NRP |
|:-----|:----|
| Hafiz Akmaldi Santosa | 5027221061 |
| Muhammad Arsy Athallah | 5027221048 |
| Muhammad Rifqi Oktaviansyah | 5027221067 |
| Angella Christie | 5027221047 |

Proyek ini mengimplementasikan pipeline DevSecOps yang menyelesaikan dua celah kritis dalam keamanan *software supply chain*:
1. **Blind spot CVE** akibat *vendor affinity* pada toolchain SBOM → diselesaikan dengan multi-tool vendor-aligned scanning paralel.
2. **Hambatan adopsi artifact signing** akibat kompleksitas manajemen kunci → diselesaikan dengan keyless signing via Sigstore/Cosign.

---

## Struktur Repository

```
├── README.md                        ← Dokumen ini
├── papers/
│   ├── paper-1-sbom-vulnerability.md   ← Reading notes: O'Donoghue et al. (2024)
│   └── paper-2-sigstore.md             ← Reading notes: Newman et al. (2022)
├── research/
│   ├── 01-gap-analysis.md              ← Analisis celah keamanan
│   ├── 02-state-of-the-art.md          ← Pemetaan teknologi saat ini
│   └── 03-design-decisions.md          ← Keputusan desain + justifikasi
├── implementation/
│   ├── Dockerfile                      ← Container image target
│   ├── app/
│   │   ├── index.js                    ← Node.js Express demo app
│   │   └── package.json
│   └── .github/workflows/
│       └── devsecops-pipeline.yml      ← Pipeline utama GitHub Actions
├── evaluation/
│   ├── metrics-before.md               ← Baseline sebelum implementasi
│   ├── metrics-after.md                ← Hasil eksekusi pipeline
│   └── analysis.md                     ← Analisis efektivitas
├── presentation/
│   └── slides.pdf                      ← Slide presentasi
└── docs/
    └── refleksi-kelompok.md            ← Refleksi tim
```

---

## Arsitektur Pipeline

```
┌─────────┐
│  build  │  → Build & push image ke GHCR
└────┬────┘
     │
  ┌──┴──────────────┐
  ▼                 ▼
┌──────────────┐  ┌─────────────┐
│ sbom-anchore │  │  sbom-aqua  │  → Generate SBOM (Syft & Trivy paralel)
│  (Syft)      │  │  (Trivy)    │
└──────┬───────┘  └──────┬──────┘
       │                 │
  ┌────┴────┐      ┌─────┴────┐
  ▼         │      │          ▼
┌──────────┐│      │┌──────────────┐
│scan-      ││      ││ scan-aqua    │  → Scan CVE (Grype & Trivy paralel)
│anchore    ││      ││ (Trivy)      │
│(Grype)   ││      │└──────┬───────┘
└─────┬────┘│      │       │
      └──┬──┘      └──┬────┘
         └──────┬──────┘
                ▼
     ┌──────────────────┐
     │  sign-and-verify │  → Cosign keyless sign + Rekor logging
     └──────────────────┘
```

---

## Setup & Cara Menjalankan

### Prasyarat

- Repository di GitHub (public atau private dengan GitHub Actions enabled)
- GitHub Container Registry (GHCR) aktif — otomatis tersedia di semua GitHub account
- Tidak diperlukan secret tambahan — pipeline menggunakan `GITHUB_TOKEN` bawaan dan OIDC keyless signing

### Langkah Setup

**1. Fork atau clone repository ini ke akun GitHub kamu**

```bash
git clone https://github.com/<org>/kelompok-1-devsecops-future.git
cd kelompok-1-devsecops-future
```

**2. Aktifkan GitHub Actions**

Masuk ke Settings → Actions → General → pilih "Allow all actions and reusable workflows".

**3. Aktifkan izin write untuk GITHUB_TOKEN**

Settings → Actions → General → Workflow permissions → pilih "Read and write permissions".

**4. Trigger pipeline**

Push ke branch `main`:

```bash
git add .
git commit -m "trigger: jalankan pipeline DevSecOps"
git push origin main
```

**5. Pantau eksekusi**

Buka tab **Actions** di GitHub repository → pilih run terbaru → pantau setiap job.

---

## Output Pipeline

Setelah pipeline selesai, artifact berikut tersedia di tab Actions → run terbaru → bagian Artifacts:

| Artifact | Isi |
|:---------|:----|
| `sbom-syft` | SBOM dalam CycloneDX JSON (dibuat Syft) |
| `sbom-trivy` | SBOM dalam CycloneDX JSON (dibuat Trivy) |
| `report-grype` | Laporan CVE dari Grype (JSON) |
| `report-trivy` | Laporan CVE dari Trivy scan (JSON) |

Image yang telah ditandatangani tersedia di:
```
ghcr.io/<org>/supply-chain-demo:<tag>
```

### Verifikasi Signature (siapa pun bisa melakukan ini)

```bash
# Install cosign
brew install cosign  # atau: go install github.com/sigstore/cosign/v2/cmd/cosign@latest

# Verifikasi
cosign verify \
  --certificate-identity-regexp "https://github.com/<org>/kelompok-1-devsecops-future/*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/<org>/supply-chain-demo@sha256:<digest>
```

---

## Referensi Paper

1. O'Donoghue, E., Boles, B., Izurieta, C., & Reinhold, A. M. (2024). *Impacts of Software Bill of Materials (SBOM) Generation on Vulnerability Detection*. SCORED '24, ACM Digital Library.

2. Newman, T., et al. (2022). *Sigstore: Software Signing for Everybody*. CCS '22, ACM Digital Library.
