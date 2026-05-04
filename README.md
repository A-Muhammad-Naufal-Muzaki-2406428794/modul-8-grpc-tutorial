## Reflection

**1. Perbedaan utama antara RPC *unary*, *server streaming*, dan *bi-directional streaming*, serta skenario penggunaannya:**
* [cite_start]**Unary:** Klien mengirimkan satu permintaan ke server dan menunggu satu balasan [cite: 50-51]. Sangat cocok untuk operasi standar seperti mengambil satu data dari database, autentikasi pengguna, atau melakukan perhitungan sederhana.
* [cite_start]**Server Streaming:** Klien mengirim satu permintaan, dan server merespons dengan aliran data (*stream*) yang kontinu [cite: 58-59]. Skenario yang tepat adalah ketika server perlu mendorong (*push*) pembaruan *real-time* seperti harga saham, cuaca, *news feed*, atau mengirim file berukuran besar secara bertahap (*chunks*).
* [cite_start]**Bi-directional Streaming:** Klien dan server dapat saling mengirimkan aliran pesan secara bersamaan tanpa saling menunggu [cite: 76-77]. [cite_start]Skenario terbaiknya adalah aplikasi komunikasi interaktif *real-time* seperti aplikasi *chat* atau pertukaran data analitik yang konstan antara kedua pihak [cite: 79-80].

**2. Pertimbangan keamanan dalam mengimplementasikan layanan gRPC di Rust:**
* **Enkripsi (Data in Transit):** Wajib menggunakan TLS/SSL untuk mengenkripsi jalur komunikasi HTTP/2 agar data tidak dapat di-*intercept* oleh pihak ketiga.
* **Autentikasi:** Menggunakan mekanisme *token-based* seperti JWT (JSON Web Tokens) yang disisipkan melalui metadata (mirip dengan HTTP Headers) pada setiap *request* gRPC.
* **Otorisasi:** Mengimplementasikan *interceptor* atau *middleware* di level server Rust untuk memvalidasi peran (Role-Based Access Control) pengguna sebelum fungsi inti gRPC dieksekusi.

**3. Tantangan yang muncul saat menangani *bidirectional streaming* di Rust gRPC (misal: aplikasi chat):**
* **Manajemen *State* (Koneksi):** Server harus melacak daftar klien yang terhubung secara *concurrent*. Di ekosistem Tokio Rust, ini biasanya memerlukan struktur data yang aman untuk multi-utas seperti `Arc<Mutex<HashMap>>` atau `RwLock`.
* **Penanganan *Disconnect*:** Mendeteksi ketika klien tiba-tiba terputus (karena masalah jaringan) dan memastikan *resource* atau *channel* memori dibersihkan agar tidak terjadi *memory leak*.
* **Blocking Operations:** Memastikan operasi pemrosesan pesan tidak memblokir *async runtime* Tokio, yang dapat membuat server gagal merespons pesan dari klien lain.

**4. Kelebihan dan kekurangan menggunakan `tokio_stream::wrappers::ReceiverStream`:**
* **Kelebihan:** Sangat mudah digunakan untuk menjembatani asinkronus Rust dengan gRPC. *Wrapper* ini secara otomatis mengonversi `mpsc::Receiver` (penerima saluran dari Tokio) menjadi tipe data *Stream* yang valid dan dapat langsung dibaca oleh *framework* Tonic untuk respons gRPC [cite: 354-355, 466-467].
* **Kekurangan:** Menambahkan sedikit lapisan abstraksi tambahan yang memakan memori (terutama jika ukuran *buffer channel* tidak dikonfigurasi dengan optimal). Jika saluran penuh dan konsumen (*client*) lambat membaca, ini berpotensi memblokir produsen pesan di sisi server.

**5. Restrukturisasi kode Rust gRPC untuk modularitas dan pemeliharaan:**
* Memisahkan *logic* bisnis murni dari lapisan transport (gRPC). *Handler* gRPC hanya boleh bertugas menerima *request* dan memanggil fungsi *service* yang ada di modul lain.
* Membagi kode ke dalam direktori terpisah, misalnya `src/api` untuk gRPC, `src/service` untuk logika bisnis, dan `src/repository` untuk interaksi database.
* Definisi Protobuf (`.proto`) dan kode yang di-*generate* oleh `tonic-build` sebaiknya dipisahkan ke dalam modul atau *crate* tersendiri agar dapat dibagikan (*shared*) dengan mudah ke layanan lain jika menggunakan arsitektur *microservices*.

**6. Langkah tambahan untuk `MyPaymentService` dalam menangani logika yang kompleks:**
* **Idempotensi:** Menyimpan *idempotency key* di database untuk mencegah terjadinya tagihan ganda (*double charge*) jika klien mengirim ulang permintaan yang sama akibat *timeout*.
* **Integrasi Pihak Ketiga:** Memanggil API eksternal (*Payment Gateway* seperti Midtrans atau Stripe) menggunakan klien HTTP asinkron (misal: `reqwest`).
* **Transaksional Database:** Memastikan pembaruan saldo dan pencatatan riwayat transaksi dilakukan dalam satu sesi transaksi database (ACID) agar data tetap konsisten jika terjadi kegagalan sistem.

**7. Dampak adopsi gRPC terhadap arsitektur sistem terdistribusi (interoperabilitas):**
* Mendorong adopsi arsitektur *Microservices* yang kuat karena komunikasi antar-servis (*backend-to-backend*) menjadi sangat cepat dan efisien.
* Sangat mendukung *Polyglot Architecture*, di mana tim berbeda dapat menggunakan bahasa pemrograman yang berbeda (Rust, Go, Python, Java). Protobuf menjamin interoperabilitas karena *source code* klien dan server dapat di-*generate* secara otomatis untuk semua bahasa tersebut.
* Memerlukan konfigurasi infrastruktur tambahan seperti *gRPC-Web* atau *API Gateway* (seperti Envoy) jika layanan gRPC ingin diakses secara langsung oleh aplikasi *frontend* berbasis *browser*, karena *browser* modern belum mendukung HTTP/2 raw gRPC secara penuh.

**8. Kelebihan dan kekurangan HTTP/2 (gRPC) dibandingkan HTTP/1.1 (REST API):**
* **Kelebihan:** HTTP/2 mendukung *multiplexing* (banyak *request* dalam satu koneksi TCP), kompresi *header* (HPACK), dan transmisi biner yang jauh lebih cepat serta ringan dibandingkan format teks biasa (*plaintext*) di HTTP/1.1.
* **Kekurangan:** Sulit di-*debug* secara manual oleh manusia. Jika HTTP/1.1 bisa dibaca langsung menggunakan cURL atau inspektor jaringan browser (karena berupa teks), HTTP/2 memerlukan alat khusus seperti `grpcurl` atau *Wireshark* untuk melakukan *decoding* paket biner.

**9. Perbandingan model *request-response* REST dengan *bidirectional streaming* gRPC untuk komunikasi *real-time*:**
* REST API tradisional sangat tidak efisien untuk *real-time* karena mengharuskan teknik *Long-Polling* atau *Short-Polling*, di mana klien harus terus-menerus membuka dan menutup koneksi HTTP baru hanya untuk mengecek apakah ada data baru.
* gRPC *bidirectional streaming* mengatasi hal ini dengan mempertahankan satu koneksi HTTP/2 yang tetap terbuka (*persistent*). Ini menghilangkan *overhead* dari *TCP handshake* berulang dan memungkinkan latensi yang sangat rendah saat data mengalir di antara klien dan server secara interaktif.

**10. Implikasi pendekatan berbasis skema (Protobuf) dibandingkan format *schema-less* (JSON):**
* **Keamanan Kontrak (*Type Safety*):** Protobuf mewajibkan kontrak data yang ketat. Jika klien mengirim tipe data yang salah, sistem akan menolaknya pada tahap deserialisasi, menghindari *error* tak terduga di dalam aplikasi. JSON sangat fleksibel, namun sering menyebabkan *runtime error* jika ada bidang (*field*) yang hilang atau tipe datanya berubah tanpa pemberitahuan.
* **Ukuran dan Kecepatan:** Payload Protobuf dikompresi menjadi biner sehingga ukurannya jauh lebih kecil dari JSON. Ini mempercepat waktu transmisi dan mengurangi beban pemrosesan (*parsing*) CPU.
* **Kompatibilitas Mundur (*Backward Compatibility*):** Protobuf menggunakan penomoran tag (misal `string user_id = 1;`), sehingga sangat mudah untuk menambah atau menghapus bidang baru di masa depan tanpa merusak (*breaking change*) klien versi lama yang masih berjalan.