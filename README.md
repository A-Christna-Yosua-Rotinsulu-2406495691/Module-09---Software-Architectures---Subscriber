# Penjelasan AMQP dan Konfigurasi Koneksi

## A. Apa itu AMQP?

**AMQP (Advanced Message Queuing Protocol)** adalah sebuah standar protokol terbuka di lapisan aplikasi (application layer) yang digunakan untuk *message-oriented middleware* (middleware yang berorientasi pada pesan). Secara sederhana, AMQP adalah aturan komunikasi yang memungkinkan berbagai aplikasi, sistem, atau layanan (meskipun ditulis dengan bahasa pemrograman yang berbeda) untuk saling mengirim dan menerima pesan secara asinkron, aman, dan andal.

Di dalam *repository* ini, saya menggunakan AMQP sebagai cara bagi aplikasi (subscriber) untuk berkomunikasi dengan *message broker* (yaitu RabbitMQ). Aplikasi saya memanfaatkan *library* `crosstown_bus` untuk mendengarkan *event* spesifik dari antrean (queue) melalui jaringan dengan protokol ini.

Berikut adalah cuplikan kode dari `src/main.rs` di mana saya menggunakan AMQP untuk mendengarkan pesan dari *queue*:
```rust
fn main() {
    let listener = CrosstownBus::new_queue_listener(
        "amqp://guest:guest@localhost:5672".to_owned()
    ).unwrap();

    _ = listener.listen(
        "user_created".to_owned(), 
        UserCreatedHandler{},
        crosstown_bus::QueueProperties { 
            auto_delete: false, 
            durable: false, 
            use_dead_letter: true 
        }
    );

    loop {}
}
```

## B. Apa arti dari `guest:guest@localhost:5672`?

Dalam fungsi `main()`, terdapat string URL `"amqp://guest:guest@localhost:5672"`. String ini adalah format URL otentikasi dan koneksi standar yang saya berikan kepada `crosstown_bus` untuk memberitahukan ke mana ia harus terhubung. Berikut adalah rincian dari setiap komponen pada URL tersebut:

1. **`guest` (pertama)**: 
   Ini adalah **username** (nama pengguna) yang saya gunakan untuk otentikasi dengan broker (RabbitMQ). `guest` merupakan username *default* bawaan yang disediakan oleh RabbitMQ.
   
2. **`guest` (kedua - setelah tanda `:` )**: 
   Ini adalah **password** (kata sandi) dari pengguna tersebut. Sama seperti username, `guest` adalah password *default* yang ditetapkan oleh RabbitMQ.

3. **`localhost` (setelah tanda `@` )**: 
   Ini menunjukkan **hostname** atau alamat server di mana *message broker* (RabbitMQ) saya sedang berjalan. Karena RabbitMQ berjalan di komputer lokal (komputer saya sendiri), maka saya mengarahkannya ke `localhost` (atau `127.0.0.1`).

4. **`5672` (setelah tanda `:` terakhir )**: 
   Ini adalah nomor **port** di mana RabbitMQ menunggu (listen) koneksi masuk untuk jalur komunikasi AMQP. Secara *default*, RabbitMQ mendengarkan port TCP 5672.

Jadi, jika diartikan dalam bahasa manusia, baris tersebut memberi instruksi: *"Tolong hubungkan aplikasi saya melalui protokol **AMQP** ke server yang berjalan di **localhost** pada port **5672**, lalu *login* dengan username **guest** dan password **guest**."*

## C. Simulation Slow Subscriber

![Console Connection Sleep](assets/images/ConsoleConnectionSleep.png)
![RabbitMQ Connection Sleep](assets/images/RabbitMQ-ConnectionSleep.png)

You will see something like this. It means the producer can just keep sending requests, and those requests (as an event) are put on a queue message. Slowly the consumer will process it one by one.

### Why is the total number of queue messages 20?
Berdasarkan observasi saya, jumlah pesan di antrean mencapai 20 karena program **publisher dijalankan sebanyak 4 kali**. Setiap satu kali pemanggilan program publisher, ia akan mengirimkan 5 pesan (event) ke RabbitMQ. Dengan menjalankan publisher 4 kali secara berurutan, maka total ada $5 \times 4 = 20$ pesan yang masuk ke antrean.

Karena program **subscriber sengaja dibuat lambat** dengan adanya perintah `thread::sleep(ten_millis)` (durasi 1 detik) di dalam *handler*-nya, subscriber tidak dapat mengonsumsi pesan secepat publisher mengirimkannya. Akibatnya, pesan-pesan tersebut menumpuk di dalam antrean (queue) RabbitMQ hingga mencapai angka 20 (atau lebih, tergantung berapa kali kita menjalankan publisher sebelum subscriber menghabiskannya).
