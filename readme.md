# 🚀 README DEMO: Load Balancing & Read-Write Splitting dengan Pgpool-II

## 🧩 Pendahuluan

Demo ini bertujuan membuktikan bahwa arsitektur **database terdistribusi** yang terdiri dari:

* **PostgreSQL Master (primary_db)**
* **PostgreSQL Slave (secondary_db)**
* **Pgpool-II Proxy (pgpool_proxy)**

berhasil melakukan **Read-Write Splitting** dan **Load Balancing**.

---

## ⚙️ Arsitektur Server

| Server     | Kontainer      | Port | Fungsi                                   |
| :--------- | :------------- | :--- | :--------------------------------------- |
| **Master** | `primary_db`   | 5432 | Menerima query **WRITE (INSERT/UPDATE)** |
| **Slave**  | `secondary_db` | 5433 | Menerima query **READ (SELECT)**         |
| **Proxy**  | `pgpool_proxy` | 9999 | Load Balancer & Query Router             |

---

## 🧱 1. Persiapan Awal — Verifikasi Replikasi

**Asumsi:**
Docker Compose sudah dijalankan dengan `docker compose up -d`, dan Pgpool sudah aktif.

### A. Verifikasi Status Koneksi (Master)

**Tujuan:** Pastikan Slave (`secondary_db`) tersambung ke Master (`primary_db`) dan replikasi aktif.

```bash
docker exec -it primary_db psql -U admin -d main_db \
-c "SELECT client_addr, state FROM pg_stat_replication;"
```

**Hasil Diharapkan:**
Tampil satu baris dengan `state = streaming`, artinya Slave sedang menyalin data dari Master.

---

### B. Verifikasi Data (Slave)

**Tujuan:** Memastikan Slave memiliki tabel yang sama dengan Master.

```bash
docker exec -it secondary_db psql -U admin -d main_db \
-c "SELECT * FROM products;"
```

**Hasil Diharapkan:**
Tabel `products` muncul (walau kosong).

---

## 💾 2. DEMO 1 — Read-Write Splitting

> 🎓 **Pertanyaan Dosen:** “Silakan didemokan Read-Write Splitting.”

### A. Uji WRITE (INSERT) melalui Proxy

**Tujuan:** Membuktikan query `INSERT` diarahkan ke Master (`primary_db`).

```bash
echo "--- Uji WRITE via Proxy (Port 9999) ---"

docker exec -it pgpool_proxy psql -h localhost -p 9999 -U admin -d main_db \
-c "INSERT INTO products (item_name, price) VALUES ('Produk Uji A', 100000);"
```

🗣️ **Penjelasan:**
Karena ini query **WRITE**, Pgpool-II otomatis merutekannya ke **Master** untuk menjaga konsistensi data.

---

### B. Verifikasi Split (Master)

**Tujuan:** Memastikan data berhasil ditulis di Master.

```bash
docker exec -it primary_db psql -U admin -d main_db \
-c "SELECT * FROM products WHERE item_name = 'Produk Uji A';"
```

**Hasil Diharapkan:**
Data `'Produk Uji A'` muncul di tabel Master.

---

## ⚖️ 3. DEMO 2 — Load Balancing

> 🎓 **Pertanyaan Dosen:** “Mana buktinya Load Balancing?”

### A. Uji READ (SELECT) sebanyak 5 kali via Proxy

**Tujuan:** Menghasilkan traffic SELECT agar Pgpool mencatat distribusi beban.

```bash
echo "--- Uji SELECT 5 Kali untuk Load Balancing ---"

# Jalankan 5x
docker exec -it pgpool_proxy psql -h localhost -p 9999 -U admin -d main_db \
-c "SELECT pg_backend_pid();"
# ... (ulang total 5 kali)
```

---

### B. Bukti Visual Load Balancing (Counter)

**Tujuan:** Menunjukkan hasil perhitungan `select_cnt` pada tiap node.

```bash
docker exec -it pgpool_proxy psql -h localhost -p 9999 -U admin -d main_db \
-c "SHOW pool_nodes;"
```

| Kolom          | Node 0 (Master) | Node 1 (Slave) | Penjelasan                                                     |
| :------------- | :-------------- | :------------- | :------------------------------------------------------------- |
| **lb_weight**  | 0.000000        | 1.000000       | Master tidak menerima SELECT; semua SELECT dialihkan ke Slave. |
| **select_cnt** | 0               | 5              | Terbukti: Slave menerima seluruh beban baca.                   |

💡 **Interpretasi:**
Master bebas dari beban baca (SELECT), sementara Slave menangani semua query baca dengan load balancing ideal **(100% SELECT ke Slave).**

---

## 🏁 4. Kesimpulan

| Aspek                    | Hasil                                                        |
| :----------------------- | :----------------------------------------------------------- |
| **Read-Write Splitting** | Query WRITE → Master, Query READ → Slave ✅                   |
| **Load Balancing**       | 100% query SELECT dialihkan ke Slave ✅                       |
| **Efektivitas**          | Master fokus pada penulisan data, Slave menangani pembacaan. |

🗣️ **Pernyataan Akhir:**

> “Dengan ini kami membuktikan bahwa arsitektur PostgreSQL + Pgpool-II kami berhasil menjalankan *Read-Write Splitting* dan *Load Balancing* sesuai tujuan proyek.”

