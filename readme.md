# Demo Implementasi Citus: 1 Primary + 2 Shard

File ini berisi panduan *step-by-step* untuk demo live implementasi Citus (PostgreSQL Terdistribusi) menggunakan Docker Compose sesuai dengan Tugas Pertemuan 09.

## Prasyarat

  * Docker sudah ter-install.
  * Docker Compose sudah ter-install.

-----

## Langkah 1: Buat file `docker-compose.yml`

Buat file dengan nama `docker-compose.yml` dan isi dengan kode berikut:

```yaml
version: '3.8'

services:
  # Ini adalah Primary / Coordinator Node
  citus_coordinator:
    image: citusdata/citus:12.1
    ports:
      - "5432:5432" # Ini port utama
    environment:
      - POSTGRES_PASSWORD=password123
    volumes:
      - coordinator_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Ini Shard 1 / Worker Node 1
  citus_worker_1:
    image: citusdata/citus:12.1
    environment:
      - POSTGRES_PASSWORD=password123
    volumes:
      - worker1_data:/var/lib/postgresql/data
    depends_on:
      citus_coordinator:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Ini Shard 2 / Worker Node 2
  citus_worker_2:
    image: citusdata/citus:12.1
    environment:
      - POSTGRES_PASSWORD=password123
    volumes:
      - worker2_data:/var/lib/postgresql/data
    depends_on:
      citus_coordinator:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  coordinator_data:
  worker1_data:
  worker2_data:
```

-----

## Langkah 2: Jalankan Semua Kontainer

Buka terminal di folder yang sama dengan file `docker-compose.yml` dan jalankan:


```bash
docker-compose up -d
```

*(Tunggu sampai proses download image dan statusnya `healthy` atau `done`)*

-----

## Langkah 3: Masuk ke Coordinator dan Daftarkan Worker

1.  Masuk ke `psql` di dalam kontainer `citus_coordinator` (Primary Node):

    ```bash
    docker-compose exec citus_coordinator psql -U postgres
    ```

2.  Setelah masuk ke `postgres=#`, daftarkan **Worker 1**:

    ```sql
    SELECT * from citus_add_node('citus_worker_1', 5432);
    ```

3.  Daftarkan **Worker 2**:

    ```sql
    SELECT * from citus_add_node('citus_worker_2', 5432);
    ```

-----

## Langkah 4: Verifikasi (BUKTI TUGAS)

Masih di dalam `psql`, jalankan perintah ini untuk membuktikan 1 Primary (Coordinator) sudah mengenali 2 Shard (Worker).

```sql
SELECT * FROM citus_get_active_worker_nodes();
```

*(Hasilnya akan menampilkan `citus_worker_1` dan `citus_worker_2`)*

-----

## Langkah 5: (Demo Implementasi) Uji Coba Sharding

1.  Buat tabel `users`:

    ```sql
    CREATE TABLE users (
        id BIGSERIAL PRIMARY KEY,
        name TEXT,
        email TEXT
    );
    ```

2.  Ubah tabel `users` menjadi tabel terdistribusi (sharding) berdasarkan kolom `id`:

    ```sql
    SELECT create_distributed_table('users', 'id');
    ```

3.  Masukkan data (data ini akan disebar ke worker):

    ```sql
    INSERT INTO users (name, email) VALUES
    ('Nabila', 'nabila@example.com'),
    ('Dosen Keren', 'dosen@example.com'),
    ('Citus', 'citus@example.com');
    ```

4.  Ambil datanya kembali (Citus akan otomatis mengambil dari semua worker):

    ```sql
    SELECT * FROM users;
    ```

5.  Keluar dari `psql`:

    ```sql
    \q
    ```

-----

## Langkah 6: Hentikan dan Bersihkan Demo

Setelah demo selesai, jalankan perintah ini di terminal awal (bukan `psql`) untuk mematikan dan menghapus semua kontainer & volume.

```bash
docker-compose down -v
```

