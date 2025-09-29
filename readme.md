````markdown
# PostgreSQL Logical Replication dengan Docker

## Tujuan
Membuat sinkronisasi tabel tertentu (`data_penting`) dari publisher ke subscriber menggunakan **logical replication** PostgreSQL.

**Keuntungan Logical vs Physical:**
- Pilih tabel tertentu → tidak semua database tersalin
- Bisa filter data / kolom tertentu
- Dapat digunakan lintas versi PostgreSQL
- Physical replication → clone seluruh DB, lebih cocok untuk HA / backup

---

## 1. Setup Docker Network & Containers

```powershell
# Create network (skip jika sudah ada)
docker network create pgnet

# Publisher
docker run -d --name pg_publisher --network pgnet -e POSTGRES_PASSWORD=pass_publisher -v pgdata_pub:/var/lib/postgresql/data -p 5432:5432 postgres:14

# Subscriber
docker run -d --name pg_subscriber --network pgnet -e POSTGRES_PASSWORD=pass_subscriber -v pgdata_sub:/var/lib/postgresql/data -p 5433:5432 postgres:14
````

**Tips:**

* Gunakan network custom supaya container saling connect tanpa expose port banyak
* Cek container running: `docker ps`

---

## 2. Konfigurasi Publisher

```sql
-- Wajib di set untuk logical replication
ALTER SYSTEM SET wal_level='logical';
ALTER SYSTEM SET max_replication_slots=10;
ALTER SYSTEM SET max_wal_senders=10;
ALTER SYSTEM SET max_worker_processes=10;
```

```powershell
docker restart pg_publisher
```

```sql
-- Buat role replication
CREATE ROLE repl_role WITH REPLICATION LOGIN PASSWORD 'replica_pass';
```

```bash
# Tambah di pg_hba.conf agar subscriber bisa login
docker exec -u root pg_publisher bash -c 'echo "host replication repl_role 0.0.0.0/0 md5" >> /var/lib/postgresql/data/pg_hba.conf'
docker restart pg_publisher
```

---

## 3. Buat Tabel & Publication di Publisher

```sql
CREATE TABLE data_penting (
  id SERIAL PRIMARY KEY,
  nama TEXT,
  harga INT
);

CREATE PUBLICATION pub_bisnis FOR TABLE data_penting;

INSERT INTO data_penting (nama,harga) VALUES ('Sepatu A',500000);
```

**Tips:** Publication hanya untuk tabel yang ingin disinkronkan.

---

## 4. Buat Tabel di Subscriber

Jika `copy_data=false`, tabel harus ada dulu:

```sql
CREATE TABLE data_penting (
  id SERIAL PRIMARY KEY,
  nama TEXT,
  harga INT
);
```

---

## 5. Buat Subscription

```sql
CREATE SUBSCRIPTION sub_bisnis
CONNECTION 'host=pg_publisher port=5432 user=repl_role password=replica_pass dbname=postgres'
PUBLICATION pub_bisnis
WITH (copy_data = true);
```

**Tips:**

* Jika error `relation "public.data_penting" does not exist` → buat tabel dulu atau pastikan schema sama
* Jika subscription sudah ada → `DROP SUBSCRIPTION sub_bisnis;` dulu

---

## 6. Test Sinkronisasi

```sql
-- Publisher
INSERT INTO data_penting (nama,harga) VALUES ('Tas B',850000);

-- Subscriber
SELECT * FROM data_penting;
```

Jika data belum muncul:

```sql
SELECT * FROM pg_stat_subscription;
ALTER SUBSCRIPTION sub_bisnis ENABLE;
```

Reset ID supaya kembali 1:

```sql
TRUNCATE TABLE data_penting RESTART IDENTITY;
```

---

## 7. Common Errors & Solusi

| Error                                           | Solusi                                                      |
| ----------------------------------------------- | ----------------------------------------------------------- |
| `relation "public.data_penting" does not exist` | Buat tabel di subscriber dulu, atau pakai `copy_data=false` |
| `could not drop relation mapping`               | Drop subscription dulu sebelum drop tabel                   |
| Duplicate subscription                          | `DROP SUBSCRIPTION sub_bisnis;` sebelum create lagi         |

---

## 8. Tips & Trik Lain

* `TRUNCATE ... RESTART IDENTITY` → reset SERIAL id
* `copy_data=true` → copy data awal, cuma sekali
* Bisa lebih dari 1 subscriber, physical replication hanya 1 standby
* Filter kolom/row di versi Postgres terbaru (15+)

---

## 9. Step-by-Step Ringkas

1. Setup Docker network & containers
2. Konfigurasi publisher (`wal_level`, slots, role, hba)
3. Buat tabel & publication di publisher
4. Buat tabel subscriber (jika copy_data=false)
5. Buat subscription
6. Test insert & sinkronisasi
7. Reset data & identity jika perlu
8. Debug pakai `pg_stat_subscription` / `ALTER SUBSCRIPTION ... ENABLE`

---

## 10. Kesimpulan

* Logical replication = fleksibel untuk partial sync, reporting, integrasi sistem
* Physical replication = cocok untuk HA / backup full DB
* Tips: selalu cek status subscription, drop sebelum drop tabel, sesuaikan `copy_data` dengan kebutuhan

