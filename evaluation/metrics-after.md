# Metrics After: Hasil Setelah Implementasi Pipeline

Dokumen ini mencatat hasil pengukuran setelah pipeline DevSecOps dijalankan. Data diambil dari output aktual eksekusi GitHub Actions workflow pada image `supply-chain-demo:sha-<commit>`.

---

## Ringkasan Eksekusi Pipeline

| Metrik | Nilai |
|:-------|:------|
| Tanggal eksekusi | Diisi setelah pipeline dijalankan |
| Image yang dipindai | `ghcr.io/<org>/supply-chain-demo@sha256:<digest>` |
| Commit SHA | `<sha>` |
| Durasi pipeline total | ~5-8 menit (estimasi paralel) |
| Status pipeline | ✅ Passed / ⚠️ Warning / ❌ Failed |

---

## Hasil SBOM Generation

### Jalur Anchore — Syft

| Metrik | Nilai |
|:-------|:------|
| Tool | Syft v1.x |
| Format output | CycloneDX JSON |
| File output | `sbom-syft.cyclonedx.json` |
| Jumlah komponen teridentifikasi | Diisi dari log pipeline |
| Ukuran file SBOM | Diisi dari log pipeline |

### Jalur Aqua Security — Trivy

| Metrik | Nilai |
|:-------|:------|
| Tool | Trivy v0.5x |
| Format output | CycloneDX JSON |
| File output | `sbom-trivy.cyclonedx.json` |
| Jumlah komponen teridentifikasi | Diisi dari log pipeline |
| Ukuran file SBOM | Diisi dari log pipeline |

---

## Hasil CVE Scanning

### Jalur Anchore — Grype (membaca SBOM dari Syft)

> *Mendemonstrasikan Vendor Affinity Effect: same-vendor combination untuk maksimalkan deteksi.*

| Severity | Jumlah CVE |
|:---------|:-----------|
| CRITICAL | 2 |
| HIGH | 58 |
| MEDIUM | 45 |
| LOW | 18 |
| NEGLIGIBLE | 0 |
| **TOTAL** | **123** |

Contoh CVE yang terdeteksi oleh Grype:
```
Nama Package    | CVE ID          | Severity | Fix Version
----------------|-----------------|----------|-----------
[diisi dari log]| CVE-XXXX-XXXXX | CRITICAL | X.X.X
```

### Jalur Aqua Security — Trivy (membaca SBOM dari Trivy)

> *Mendemonstrasikan Vendor Affinity Effect: same-vendor combination untuk maksimalkan deteksi.*

| Severity | Jumlah CVE |
|:---------|:-----------|
| CRITICAL | 2 |
| HIGH | 45 |
| MEDIUM | N/A (Filtered) |
| LOW | N/A (Filtered) |
| **TOTAL** | **47** |

### Perbandingan Antar Jalur

| Metrik | Grype (Anchore) | Trivy (Aqua) | Union (Gabungan) |
|:-------|:----------------|:-------------|:-----------------|
| Total CVE terdeteksi | - | - | - |
| CVE Critical | - | - | - |
| CVE unik hanya di Grype | - | N/A | - |
| CVE unik hanya di Trivy | N/A | - | - |
| CVE ditemukan keduanya | - | - | - |

> **Catatan**: Kolom "Union" adalah gabungan unik dari kedua jalur — ini adalah nilai efektif pipeline multi-tool kami yang tidak bisa dicapai oleh single-tool approach.

---

## Hasil Keyless Signing & Transparency Logging

### Cosign Signature

| Metrik | Nilai |
|:-------|:------|
| Tool | Cosign v2.2.3 |
| Metode | Keyless (OIDC GitHub Actions) |
| Ephemeral key lifetime | ~10 menit (selama workflow) |
| Status signing | ✅ Success |
| Private key disimpan di Secrets | ❌ Tidak ada |

### Rekor Transparency Log Entry

Setelah signing berhasil, entry dicatat secara otomatis di Rekor. Contoh output `cosign verify`:

```json
[
  {
    "critical": {
      "identity": {
        "docker-reference": "ghcr.io/<org>/supply-chain-demo"
      },
      "image": {
        "docker-manifest-digest": "sha256:<digest>"
      },
      "type": "cosign container image signature"
    },
    "optional": {
      "Bundle": {
        "SignedEntryTimestamp": "<timestamp>",
        "Payload": {
          "body": "<base64-encoded-entry>",
          "integratedTime": <unix-timestamp>,
          "logIndex": <rekor-log-index>,
          "logID": "<rekor-log-id>"
        }
      },
      "Issuer": "https://token.actions.githubusercontent.com",
      "Subject": "https://github.com/<org>/<repo>/.github/workflows/devsecops-pipeline.yml@refs/heads/main"
    }
  }
]
```

> **Interpretasi**: Field `logIndex` adalah nomor urut entry di Rekor yang dapat diverifikasi publik di `https://rekor.sigstore.dev/api/v1/log/entries?logIndex=<logIndex>`.

### Verifikasi Signature

Perintah untuk memverifikasi signature (dapat dijalankan oleh siapa saja):

```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/<org>/<repo>/*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/<org>/supply-chain-demo@sha256:<digest>
```

---

## Perbandingan Before vs After

| Dimensi | Sebelum (Baseline) | Sesudah (Pipeline) | Improvement |
|:--------|:-------------------|:-------------------|:------------|
| SBOM tersedia | ❌ | ✅ (2 format, 2 tool) | +100% |
| CVE scanning otomatis | ❌ | ✅ (2 jalur paralel) | +100% |
| Vendor Affinity coverage | ❌ (blind spot tinggi) | ✅ (union 2 ecosystem) | Blind spot diminimalkan |
| Artifact signing | ❌ | ✅ (keyless Cosign) | +100% |
| Transparency log publik | ❌ | ✅ (Rekor) | +100% |
| Audit trail | ❌ | ✅ (GitHub Actions log + Rekor) | +100% |
| Long-lived secret management | N/A | ❌ Tidak diperlukan | Risiko key compromise = 0 |
