## Reflection

### 1. What are the key differences between unary, server streaming, and bi-directional streaming RPC methods, and in what scenarios would each be most suitable?
**Unary RPC** adalah pola paling sederhana di mana client mengirim satu request dan menunggu satu response, seperti pemanggilan fungsi biasa. Pola ini cocok untuk operasi sederhana seperti autentikasi pengguna, pemrosesan pembayaran, atau pengambilan satu data di mana responsnya langsung dan terbatas.
 
**Server Streaming** cocok digunakan ketika client membuat satu request tetapi server perlu mengembalikan data dalam jumlah besar atau berkelanjutan, seperti mengambil riwayat transaksi, mengirimkan pembaruan harga saham, atau mengirim file besar dalam potongan-potongan kecil. Client membaca dari stream hingga data habis.
 
**Bidirectional Streaming** memungkinkan client dan server mengirim dan menerima pesan secara independen melalui koneksi yang persisten. Pola ini ideal untuk skenario interaktif secara real-time seperti aplikasi chat, pengeditan kolaboratif, atau telemetri langsung di mana kedua pihak perlu merespons pesan satu sama lain tanpa menunggu pihak lain selesai.
 
---

### 2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?
Untuk **enkripsi**, gRPC sebaiknya di-deploy menggunakan TLS untuk melindungi data saat transit, yang didukung oleh tonic melalui konfigurasi TLS-nya. Tanpa TLS, semua pesan protocol buffer dikirim dalam format binary tetapi tidak terenkripsi, sehingga rentan terhadap penyadapan.
 
Untuk **autentikasi**, pendekatan umum meliputi penambahan metadata header pada setiap request yang membawa token seperti JWT atau API key, yang divalidasi oleh server interceptor sebelum diteruskan ke service handler.
 
**Otorisasi** harus ditangani di level service, dengan memverifikasi bahwa identitas yang telah diautentikasi memiliki izin untuk melakukan operasi yang diminta sebelum memprosesnya.
 
Selain itu, server harus memvalidasi semua pesan protobuf yang masuk dengan cermat karena pesan yang salah format atau terlalu besar dapat digunakan sebagai vektor serangan denial of service. Dalam produksi, pembatasan laju (rate limiting) dan mutual TLS menambahkan lapisan perlindungan lebih lanjut.
 
---

### 3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications? 
Salah satu tantangan signifikan adalah mengelola **backpressure**. Jika server menghasilkan pesan lebih cepat dari yang dapat dikonsumsi client, atau sebaliknya, buffer channel dapat penuh dan memblokir pengirim. Dalam tutorial ini, channel MPSC memiliki ukuran buffer tetap sebesar 10, yang berarti client yang lambat dapat menghambat pengiriman pesan.
 
Tantangan lainnya adalah **graceful shutdown**: karena kedua pihak mempertahankan stream yang terbuka, mendeteksi ketika pihak lain terputus memerlukan penanganan error yang cermat dari `stream.message().await` dan `tx.send().await`. Jika salah satu pihak panik atau melepaskan ujung channel-nya, pihak lain harus mendeteksi ini dan membersihkan resource daripada menunggu tanpa batas.
 
**Bug konkurensi** juga menjadi perhatian karena beberapa tokio task berjalan bersamaan, dan state yang dibagi di antara mereka harus ditangani dengan hati-hati untuk menghindari race condition.
 
---

### 4. What are the advantages and disadvantages of using the `tokio_stream::wrappers::ReceiverStream` for streaming responses in Rust gRPC services? 
**Kelebihan:**
- Menjembatani kesenjangan antara MPSC channel Tokio dan trait `Stream` yang diharapkan tonic untuk streaming response
- Memudahkan spawning background task yang menghasilkan data dan mengirimkannya melalui channel sementara main async task langsung mengembalikan stream ke tonic
- Terintegrasi secara alami dengan ekosistem async Tokio dan mendukung backpressure melalui bounded channel buffer-nya
**Kekurangan:**
- Menambahkan lapisan indirection ekstra melalui channel, yang memperkenalkan sedikit overhead dibandingkan menghasilkan stream secara langsung
- Memerlukan penentuan ukuran buffer yang cermat karena buffer yang terlalu kecil dapat menyebabkan producer sering terblokir, dan buffer yang terlalu besar dapat mengonsumsi memori berlebihan
- Propagasi error kurang transparan karena error harus dibungkus secara eksplisit dalam pesan channel
---

### 5. In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time? 
Implementasi service dapat dipisah menjadi modul atau file terpisah daripada menempatkan semuanya dalam satu file `grpc_server.rs`, dengan setiap service dalam modulnya sendiri seperti `payment_service.rs`, `transaction_service.rs`, dan `chat_service.rs`. Fungsi main di server kemudian hanya perlu mengimpor dan mendaftarkan setiap service.
 
Logika bisnis harus dipisahkan dari layer handler gRPC, sehingga handler hanya menangani parsing request dan pemformatan response sambil mendelegasikan logika aktual ke layer service.
 
Tipe dan utilitas yang dibagikan seperti helper konversi error atau pembuat response umum dapat ditempatkan dalam modul bersama. Trait dapat didefinisikan untuk layer logika bisnis sehingga implementasi mock dapat diinjeksikan selama pengujian, memisahkan layer gRPC dari logika aktual dan membuat pengujian unit menjadi mudah tanpa memerlukan server yang berjalan.
 
---

### 6. In the `MyPaymentService` implementation, what additional steps might be necessary to handle more complex payment processing logic? 
 
Dalam skenario produksi, implementasi `MyPaymentService` perlu:
 
1. **Memvalidasi** field `PaymentRequest` yang masuk, seperti memeriksa bahwa jumlahnya positif, mata uangnya didukung, dan ID pengguna ada
2. **Memanggil payment gateway eksternal** atau layanan downstream, yang memerlukan penanganan error yang tepat dan logika retry untuk kegagalan sementara
3. **Menyimpan** hasil percobaan pembayaran ke database, memerlukan integrasi dengan database client seperti SQLx atau Diesel
4. **Menangani idempotency** agar request yang diulang dengan ID pembayaran yang sama tidak menghasilkan tagihan duplikat
5. **Mengembalikan response yang lebih kaya** termasuk transaction ID, timestamp, dan kode error yang detail daripada hanya boolean success, sehingga client dapat menangani berbagai skenario kegagalan dengan tepat
---

### 7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms? 
Mengadopsi gRPC mendorong **pendekatan desain contract-first** di mana file `.proto` menjadi sumber kebenaran tunggal untuk kontrak API antar service. Ini meningkatkan konsistensi dan mengurangi bug integrasi karena stub client dan server di-generate dari definisi yang sama.
 
Ini juga mendorong **arsitektur polyglot** karena file `.proto` yang sama dapat menghasilkan client dan server dalam berbagai bahasa, memungkinkan tim memilih bahasa terbaik untuk setiap service tanpa mengorbankan interoperabilitas.
 
Namun, **dukungan browser yang terbatas** untuk gRPC berarti service yang ditujukan untuk dikonsumsi langsung oleh frontend web masih memerlukan layer REST atau GraphQL di depannya, sering diimplementasikan melalui gateway proxy. Integrasi dengan sistem legacy yang hanya berbicara HTTP/1.1 atau SOAP juga memerlukan lapisan terjemahan tambahan yang menambah kompleksitas arsitektur secara keseluruhan.
 
---

### 8. What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs? 
**Kelebihan HTTP/2:**
- Menawarkan multiplexing melalui satu koneksi TCP, artinya beberapa request dan response dapat berjalan secara bersamaan tanpa head-of-line blocking, yang secara signifikan mengurangi latensi dibandingkan HTTP/1.1
- Kompresi header via HPack mengurangi overhead, terutama bermanfaat ketika banyak request berbagi header yang serupa
- Dibandingkan WebSocket, stream HTTP/2 sudah built-in dalam protokol dan tidak memerlukan upgrade handshake, serta terintegrasi secara alami dengan infrastruktur HTTP yang ada seperti load balancer dan proxy
**Kekurangan HTTP/2:**
- Lebih kompleks untuk diimplementasikan dan di-debug daripada HTTP/1.1 karena binary framing menggantikan format teks yang dapat dibaca manusia
- Tidak semua infrastruktur jaringan sepenuhnya mendukung HTTP/2
- Dalam lingkungan dengan packet loss tinggi, ketergantungan HTTP/2 pada satu koneksi TCP sebenarnya dapat berkinerja lebih buruk karena satu paket yang hilang memblokir semua stream yang dimultipleks
---

### 9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness? 
Model request-response REST secara inheren bersifat **pull-based**: client harus memulai setiap interaksi dan menunggu response lengkap sebelum melanjutkan. Mensimulasikan perilaku real-time melalui REST biasanya memerlukan polling, di mana client berulang kali mengirim request pada interval tertentu, atau long-polling di mana server menahan koneksi sampai data tersedia. Keduanya tidak efisien dan memperkenalkan latensi. Server-sent events memberikan solusi parsial dengan memungkinkan server mendorong pembaruan melalui stream satu arah, tetapi client tetap tidak dapat mengirim pesan balik pada koneksi yang sama.
 
Bidirectional streaming gRPC secara fundamental mengubah ini dengan memungkinkan kedua pihak mengirim pesan secara independen kapan saja melalui koneksi persisten yang sama, menjadikannya benar-benar cocok untuk skenario real-time seperti chat, live dashboard, dan alat kolaboratif. Trade-off-nya adalah bidirectional streaming jauh lebih kompleks untuk diimplementasikan, diuji, dan dipahami dibandingkan REST endpoint yang stateless. 
---

### 10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads? 
**Protocol Buffers** menegakkan schema ketat yang didefinisikan dalam file `.proto`, yang berarti client dan server harus menyepakati struktur pesan pada waktu kompilasi. Ini menangkap ketidakcocokan tipe dan field yang hilang lebih awal sebelum kode pernah berjalan, yang meningkatkan keandalan dan mengurangi bug integrasi dalam tim besar atau sistem multi-service. Pengkodean binary juga jauh lebih kompak dan lebih cepat untuk di-parse daripada JSON, yang menguntungkan performa dalam skala besar.
 
Kelemahannya adalah **fleksibilitas yang berkurang**: menambah atau mengubah field memerlukan pembaruan dan recompile file `.proto` serta regenerasi stub di kedua sisi, yang dapat memperlambat iterasi dalam pengembangan tahap awal.
 
**Sifat schema-less JSON** memudahkan penambahan atau penghapusan field tanpa merusak client yang ada jika mereka toleran terhadap field yang tidak dikenal, dan payload JSON dapat diperiksa langsung di browser atau dengan alat dasar seperti `curl` tanpa langkah decoding apapun.
 
Untuk microservice internal di mana performa dan kebenaran penting, penegakan schema Protocol Buffers adalah keunggulan yang jelas, sementara JSON tetap lebih disukai untuk API publik di mana developer experience dan dukungan tooling yang luas menjadi prioritas.
