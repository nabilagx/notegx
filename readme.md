# PostgreSQL Logical Replication dengan Docker

## 📑 Table of Contents
- [Tujuan](#-tujuan)
  - [Keuntungan Logical vs Physical](#-keuntungan-logical-vs-physical)
- [1. Setup Docker Network & Containers](#-1-setup-docker-network--containers)
- [2. Konfigurasi Publisher](#-2-konfigurasi-publisher)
- [3. Buat Tabel & Publication di Publisher](#-3-buat-tabel--publication-di-publisher)
- [4. Buat Tabel di Subscriber](#-4-buat-tabel-di-subscriber)
- [5. Buat Subscription](#-5-buat-subscription)
- [6. Test Sinkronisasi](#-6-test-sinkronisasi)
- [7. Common Errors & Solusi](#-7-common-errors--solusi)
- [8. Tips & Trik](#-8-tips--trik)
- [9. Step-by-Step Ringkas](#-9-step-by-step-ringkas)
- [10. Kesimpulan](#-10-kesimpulan)


## 📌 Tujuan
Membuat sinkronisasi tabel tertentu (`data_penting`) dari **publisher** ke **subscriber** menggunakan **logical replication** PostgreSQL.

### 🔑 Keuntungan Logical vs Physical
- Logical → pilih tabel tertentu, bisa filter data/kolom, lintas versi PostgreSQL
- Physical → clone seluruh DB, cocok untuk HA / backup

---

## 📂 1. Setup Docker Network & Containers

```powershell
# Create network (skip jika sudah ada)
docker network create pgnet

# Publisher
docker run -d --name pg_publisher --network pgnet \
  -e POSTGRES_PASSWORD=pass_publisher \
  -v pgdata_pub:/var/lib/postgresql/data \
  -p 5432:5432 postgres:14

# Subscriber
docker run -d --name pg_subscriber --network pgnet \
  -e POSTGRES_PASSWORD=pass_subscriber \
  -v pgdata_sub:/var/lib/postgresql/data \
  -p 5433:5432 postgres:14
```

> 💡 **Tips:** Gunakan custom network supaya container bisa connect tanpa expose banyak port.  
> Cek container dengan: `docker ps`

---

## ⚙️ 2. Konfigurasi Publisher

```sql
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

Tambahkan di `pg_hba.conf`:

```bash
docker exec -u root pg_publisher bash -c \
  'echo "host replication repl_role 0.0.0.0/0 md5" >> /var/lib/postgresql/data/pg_hba.conf'
docker restart pg_publisher
```

---

## 🗂 3. Buat Tabel & Publication di Publisher

```sql
CREATE TABLE data_penting (
  id SERIAL PRIMARY KEY,
  nama TEXT,
  harga INT
);

CREATE PUBLICATION pub_bisnis FOR TABLE data_penting;

INSERT INTO data_penting (nama,harga) VALUES ('Sepatu A',500000);
```

> **Note:** Publication hanya berlaku untuk tabel yang dipilih.

---

## 🗂 4. Buat Tabel di Subscriber

Jika `copy_data=false`, buat tabel manual:

```sql
CREATE TABLE data_penting (
  id SERIAL PRIMARY KEY,
  nama TEXT,
  harga INT
);
```

---

## 🔗 5. Buat Subscription

```sql
CREATE SUBSCRIPTION sub_bisnis
CONNECTION 'host=pg_publisher port=5432 user=repl_role password=replica_pass dbname=postgres'
PUBLICATION pub_bisnis
WITH (copy_data = true);
```

> 💡 **Tips:**  
> - Jika error `relation "public.data_penting" does not exist` → buat tabel dulu.  
> - Jika subscription sudah ada → hapus dengan `DROP SUBSCRIPTION sub_bisnis;`.

---

## ✅ 6. Test Sinkronisasi

Publisher:

```sql
INSERT INTO data_penting (nama,harga) VALUES ('Tas B',850000);
```

Subscriber:

```sql
SELECT * FROM data_penting;
```

Jika data tidak muncul:

```sql
SELECT * FROM pg_stat_subscription;
ALTER SUBSCRIPTION sub_bisnis ENABLE;
```

Reset ID:

```sql
TRUNCATE TABLE data_penting RESTART IDENTITY;
```

---

## 🛠 7. Common Errors & Solusi

| Error                                           | Solusi                                                      |
| ----------------------------------------------- | ----------------------------------------------------------- |
| `relation "public.data_penting" does not exist` | Buat tabel di subscriber dulu, atau pakai `copy_data=false` |
| `could not drop relation mapping`               | Drop subscription dulu sebelum drop tabel                   |
| Duplicate subscription                          | `DROP SUBSCRIPTION sub_bisnis;` sebelum create lagi         |

---

## 💡 8. Tips & Trik

- `TRUNCATE ... RESTART IDENTITY` → reset SERIAL id  
- `copy_data=true` → copy data awal (sekali saja)  
- Bisa punya >1 subscriber (beda dengan physical replication)  
- PostgreSQL 15+ mendukung filter kolom/row  

---

## 📝 9. Step-by-Step Ringkas

1. Setup Docker network & containers  
2. Konfigurasi publisher (`wal_level`, slots, role, hba)  
3. Buat tabel & publication di publisher  
4. Buat tabel subscriber (jika `copy_data=false`)  
5. Buat subscription  
6. Test insert & sinkronisasi  
7. Reset data & identity jika perlu  
8. Debug dengan `pg_stat_subscription`  

---

## 🎯 10. Kesimpulan

- **Logical replication** = fleksibel untuk partial sync, reporting, integrasi sistem  
- **Physical replication** = cocok untuk HA / backup full DB  
- Selalu cek status subscription, drop sebelum drop tabel, sesuaikan `copy_data` dengan kebutuhan
