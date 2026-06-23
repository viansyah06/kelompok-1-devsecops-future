# Refleksi Kelompok: Supply Chain Security Final Project

**Mata Kuliah**: Operasional Pengembang (DevOps)
**Judul Proyek**: Supply Chain Security: Artifact Integrity Verification in CI/CD Pipeline
**Kelompok**: kelompok-1

---

## Pembagian Kerja

| Anggota | Kontribusi Utama |
|:--------|:-----------------|
| Muhammad Rifqi Oktaviansyah | Paper reading (SBOM & Vendor Affinity), Gap Analysis, State-of-the-Art |
| Muhammad Arsy Athallah | Design Decisions, Implementasi pipeline (GitHub Actions, Dockerfile) |
| Hafiz Akmaldi Santosa | Evaluasi & Analisis, Refleksi, README |
| Angella Christie | Presentasi slides, koordinasi review |

---

## Pertanyaan Refleksi

### 1. Apa yang paling mengejutkan dari paper yang kamu baca, sesuatu yang berbeda dari asumsimu sebelumnya? Bagaimana temuan itu mengubah keputusan implementasimu?

Sebelum membaca paper O'Donoghue dkk. (2024), asumsi kami  dan kemungkinan besar asumsi mayoritas praktisi DevSecOps  adalah bahwa *vulnerability scanning* adalah proses yang relatif objektif: selama kamu menggunakan tool yang populer dan terpercaya, hasilnya akan konsisten dan dapat diandalkan. Kami berasumsi bahwa perbedaan antara menggunakan Trivy atau Syft hanyalah soal preferensi antarmuka dan kecepatan, bukan soal berapa banyak kerentanan yang benar-benar terdeteksi.

Yang paling mengejutkan dari paper ini adalah skala ketidakkonsistenannya. O'Donoghue dkk. (2024) menemukan bahwa pilihan *generation tool*  alat yang membuat SBOM  memiliki efek yang jauh lebih besar terhadap jumlah CVE yang dilaporkan dibanding pilihan format SBOM (CycloneDX vs SPDX). Ini berlawanan dengan intuisi kami. Kami mengira format data adalah faktor utama interoperabilitas, karena format adalah kontrak bersama antar tool. Faktanya, bahkan dengan format yang sama persis (CycloneDX), kombinasi Syft→Grype (sesama vendor Anchore) menghasilkan deteksi CVE yang secara statistik signifikan lebih tinggi dibanding Syft→Trivy (lintas vendor). Efek ini mereka sebut *Vendor Affinity Effect*, dan magnitude-nya dibuktikan dengan *effect size* Cohen's D, bukan sekadar perbedaan kuantitas yang bisa saja terjadi secara acak.

Temuan ini juga mengungkap sesuatu yang lebih mengganggu: *dropout rate* 43,7%. Artinya hampir setengah dari SBOM yang dibuat gagal diproses oleh scanner downstream secara akurat  terutama ketika SBOM melewati batas vendor. CVE-bin-tool adalah penyumbang terbesar kegagalan ini. Ini berarti pipeline DevSecOps yang terlihat berjalan normal, menghasilkan laporan, dan tidak memberikan alarm  bisa saja melewatkan ratusan kerentanan nyata secara diam-diam. *False sense of security* yang sesungguhnya bukan dari false negative yang terlihat, melainkan dari false negative yang tidak terlihat sama sekali.

Dampak langsung terhadap keputusan implementasi kami sangat konkret. Awalnya, rencana kami adalah menggunakan Trivy untuk segalanya  generate SBOM lalu scan dengan Trivy juga, karena sederhana dan satu tool. Setelah membaca paper ini, kami menyadari bahwa pendekatan single-tool itu secara akademis tidak dapat dipertahankan. Kami mengubah arsitektur pipeline menjadi dual-toolchain paralel: jalur Anchore (Syft→Grype) dan jalur Aqua Security (Trivy→Trivy), dijalankan secara bersamaan, lalu hasilnya digabungkan. Keputusan ini bukan preferensi estetika  ini adalah respons langsung terhadap data empiris dari eksperimen dengan 2.313 container image yang dilakukan O'Donoghue dkk.

Yang juga mengubah perspektif kami adalah ketidakhadiran *ground truth* dalam paper ini. Penulis hanya bisa mengukur variabilitas *antartool*, bukan akurasi absolut masing-masing tool. Ini artinya kami tidak bisa mengklaim bahwa gabungan Syft+Grype dan Trivy+Trivy adalah deteksi yang "paling benar"  kami hanya bisa mengklaim bahwa union keduanya lebih luas cakupannya dan lebih sulit memiliki blind spot dibanding salah satu saja.

---

### 2. Di mana implementasimu berbeda dari yang diusulkan paper, dan mengapa? Apakah karena keterbatasan waktu/resources, atau karena kamu tidak setuju dengan pendekatannya?

Ada beberapa perbedaan signifikan antara apa yang kami implementasikan dengan apa yang sepenuhnya diusulkan dalam kedua paper, dan penting untuk jujur tentang alasan di balik setiap perbedaan itu.

**Perbedaan pertama: Tidak menggunakan CVE-bin-tool sebagai jalur ketiga.** Paper O'Donoghue dkk. (2024) sebenarnya menguji tiga scanner: Grype, Trivy, dan CVE-bin-tool. Kami hanya mengimplementasikan dua jalur pertama dan mengabaikan CVE-bin-tool. Alasannya bukan keterbatasan waktu, melainkan karena kami membaca paper dengan cermat dan menemukan bahwa CVE-bin-tool adalah penyumbang utama *dropout rate* 43,7% tersebut. Tool ini gagal memproses SBOM dalam jumlah besar karena ketidakcocokan format dan nama paket yang dianggap cacat. Menyertakannya dalam pipeline kami justru akan menambah kompleksitas tanpa menambah reliabilitas. Ini adalah keputusan yang kami buat *karena* membaca paper, bukan terlepas dari paper.

**Perbedaan kedua: Tidak mengevaluasi format SPDX.** Paper O'Donoghue dkk. (2024) menguji dua format SBOM (CycloneDX dan SPDX) secara paralel. Kami memilih untuk hanya menggunakan CycloneDX. Ini sebagian adalah keterbatasan waktu  menambah variabel format SBOM akan menggandakan kompleksitas pipeline  namun juga didukung oleh temuan paper itu sendiri yang menunjukkan bahwa *generation tool* memiliki efek variabilitas yang jauh lebih besar dibanding format. Dengan waktu satu minggu, kami memprioritaskan memaksimalkan coverage melalui variasi toolchain daripada variasi format.

**Perbedaan ketiga: Skala eksperimen yang jauh lebih kecil.** Paper O'Donoghue dkk. (2024) menggunakan 2.313 container image dari Docker Hub untuk memastikan temuan mereka signifikan secara statistik. Kami hanya menggunakan satu image: aplikasi Node.js kecil yang kami buat sendiri. Ini adalah keterbatasan resource dan waktu yang nyata. Satu image tidak cukup untuk membuat klaim statistik tentang efektivitas pipeline kami  kami hanya bisa mendemonstrasikan bahwa pipeline berjalan dan menghasilkan output yang berbeda dari kedua jalur. Untuk penelitian yang lebih serius, eksperimen pada puluhan image dengan profil ketergantungan yang beragam akan jauh lebih representatif.

**Perbedaan keempat: Implementasi keyless signing kami tidak mencakup *policy enforcement*.** Newman dkk. (2022) tidak hanya membahas cara menandatangani artefak, tetapi juga cara *menegakkan* verifikasi signature sebagai gate deployment  misalnya melalui admission controller Kubernetes yang menolak image tanpa signature valid. Kami mengimplementasikan bagian signing dan logging ke Rekor, tetapi tidak mengimplementasikan enforcement di sisi deployment. Ini murni keterbatasan waktu dan scope: menyiapkan kluster Kubernetes dengan admission controller di luar jangkauan proyek satu minggu. Kami mendokumentasikan ini sebagai kesenjangan yang disadari dalam `evaluation/analysis.md`.

Secara keseluruhan, perbedaan-perbedaan ini tidak kami sembunyikan  kami anggap sebagai bagian penting dari evaluasi yang jujur. Implementasi yang mengakui batasannya jauh lebih berharga secara ilmiah dibanding implementasi yang mengklaim sempurna tetapi tidak transparan tentang asumsi dan keterbatasannya.

---

### 3. Jika kamu punya waktu satu bulan penuh dan akses ke production cluster nyata, apa yang akan kamu lakukan berbeda atau tambahkan? Gunakan paper sebagai landasan argumenmu.

Dengan satu bulan penuh dan akses ke production cluster nyata, ada empat arah pengembangan yang kami yakini akan secara signifikan meningkatkan dampak dan kekokohan implementasi ini  semuanya berakar pada keterbatasan yang diidentifikasi oleh paper maupun keterbatasan yang kami sadari sendiri selama implementasi.

**Pertama: Memperluas eksperimen ke puluhan container image berbeda.** Ini adalah kelemahan paling fundamental dari implementasi kami saat ini. O'Donoghue dkk. (2024) menggunakan 2.313 image karena mereka perlu membuktikan *Vendor Affinity Effect* secara statistik, bukan hanya anekdotal. Dengan satu image demo Node.js, kami hanya bisa menunjukkan bahwa pipeline berjalan  kami tidak bisa membuat klaim tentang seberapa besar perbedaan deteksi CVE antara kedua jalur secara rata-rata. Dengan satu bulan, kami akan membangun eksperimen yang menscan setidaknya 50 container image dari Docker Hub dengan profil ketergantungan yang beragam (Python, Go, Java, Ruby), mencatat CVE dari setiap kombinasi, dan menganalisis apakah efek Vendor Affinity yang dibuktikan O'Donoghue dkk. (2024) terkonfirmasi pada dataset kami. Ini akan mengubah proyek dari demonstrasi menjadi replikasi ilmiah.

**Kedua: Mengimplementasikan Cosign Admission Controller di Kubernetes.** Saat ini, penandatanganan via Cosign hanya berjalan di sisi CI/CD  tapi tidak ada yang mencegah seseorang men-deploy image yang tidak ditandatangani langsung ke kluster tanpa melewati pipeline. Newman dkk. (2022) secara eksplisit membahas bahwa nilai signing baru terealisasi penuh ketika ada *policy enforcement* di sisi deployment. Dengan akses ke production cluster, kami akan mengintegrasikan Cosign dengan Kubernetes admission controller (melalui `policy-controller` dari Sigstore) yang secara otomatis menolak pod yang menggunakan image tanpa signature valid. Ini menutup gap yang kami sadari ada dalam implementasi saat ini: signing tanpa enforcement adalah jaminan setengah hati.

**Ketiga: Menambahkan SLSA Provenance sebagai lapisan ketiga verifikasi.** Selain SBOM dan signature, framework SLSA (Supply chain Levels for Software Artifacts) yang diusulkan oleh komunitas OpenSSF mendefinisikan level kepercayaan artefak berdasarkan *provenance*  bukti kriptografis tentang *bagaimana* artefak dibuild, bukan hanya *apa isinya*. Cosign mendukung *attestation* SLSA Provenance natively melalui perintah `cosign attest`. Dengan menambahkan attestasi ini, konsumen downstream tidak hanya bisa memverifikasi bahwa image sudah ditandatangani, tetapi juga bahwa image tersebut dibuild melalui pipeline GitHub Actions yang spesifik, dari commit yang spesifik, dan tidak pernah disentuh proses lain. Ini menutup skenario serangan yang masih terbuka dalam implementasi kami saat ini: seseorang dengan akses ke GHCR masih bisa men-push image yang ditandatangani tetapi dibuild dari lingkungan yang tidak terotorisasi.

**Keempat: Membangun dashboard manajemen kerentanan yang terintegrasi.** Saat ini, laporan CVE dari Grype dan Trivy hanya tersimpan sebagai artifact JSON di GitHub Actions yang tidak persisten lebih dari 30 hari. Dengan waktu satu bulan, kami akan mengintegrasikan pipeline dengan platform seperti DefectDojo atau dependency-track  dua platform open source yang dapat menerima laporan dari Grype dan Trivy, menyimpan riwayat historis, mendeteksi kerentanan baru yang muncul antara dua build berturut-turut, dan memberikan notifikasi ketika CVE baru dirilis untuk komponen yang sudah ada dalam SBOM. Ini mengubah pipeline dari *event-driven* (scan hanya ketika ada push) menjadi *continuous* (monitoring berkelanjutan bahkan tanpa perubahan kode)  sebuah kapabilitas yang secara operasional jauh lebih bernilai untuk production environment.
