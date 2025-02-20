<img src="https://media.geeksforgeeks.org/wp-content/uploads/20240224111842/Master-Slave-Replication.webp" width="100%" height="auto">

# Thiết Lập Primary - Replica (Master/Slave) PostgreSQL Replication Sử Dụng Docker Compose

<!-- TOC -->

# Overview
- [Chuẩn bị môi trường](#chuẩn-bị-môi-trường)
- [Tạo File `docker-compose.yml`](#tạo-file-docker-composeyml)
- [Thiết lập Primary (Master)](#thiết-lập-master)
- [Thiết lập Replica (Slave)](#thiết-lập-slave)
- [Kiểm tra hoạt động của Replication](#kiểm-tra-hoạt-động-của-replication)
- [Giám sát Replication](#giám-sát-replication)
- [Quản lý hệ thống](#quản-lý-hệ-thống)
- [Tối ưu hóa hệ thống](#tối-ưu-hóa-hệ-thống)
- [Tổng kết](#tổng-kết)

<!-- TOC -->

## 1. Chuẩn Bị Môi Trường

### 1.1 Cài đặt Docker và Docker Compose
Trước tiên, bạn cần đảm bảo Docker và Docker Compose đã được cài đặt trên máy.

### 1.2 Tạo thư mục làm việc
```sh
mkdir postgres_replication && cd postgres_replication
mkdir master slave  # Tạo thư mục để lưu dữ liệu PostgreSQL
```

## 2. Tạo File `docker-compose.yml`
Tạo file `docker-compose.yml` với nội dung sau:
```yaml
version: '3.9'
services:
  master:
    image: postgres:15
    container_name: postgres_master
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: 1234
    ports:
      - "5432:5432"
    volumes:
      - ./master:/var/lib/postgresql/data

  slave:
    image: postgres:15
    container_name: postgres_slave
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: 1234
    depends_on:
      - master
    ports:
      - "5433:5432"
    volumes:
      - ./slave:/var/lib/postgresql/data
```

## 3. Thiết Lập Master

### 3.1 Khởi động container
```sh
docker-compose up -d
```

### 3.2 Kết nối vào container Master
```sh
docker exec -it postgres_master bash
```

### 3.3 Cấu hình PostgreSQL
Sửa file `postgresql.conf`:
```sh
nano /var/lib/postgresql/data/postgresql.conf
```
Thêm các dòng sau:
```
wal_level = replica
max_wal_senders = 10
wal_keep_size = 64MB
```

Sửa file `pg_hba.conf`:
```sh
nano /var/lib/postgresql/data/pg_hba.conf
```
Thêm dòng sau:
```
host    replication    replicator2    0.0.0.0/0    md5
```

### 3.4 Khởi động lại PostgreSQL trên Master
```sh
pg_ctl -D /var/lib/postgresql/data restart
```

### 3.5 Tạo tài khoản replication
```sql
CREATE ROLE replicator2 WITH REPLICATION PASSWORD '1234' LOGIN;
```

## 4. Thiết Lập Slave

### 4.1 Kết nối vào container Slave
```sh
docker exec -it postgres_slave bash
```

### 4.2 Xóa dữ liệu cũ và sao chép dữ liệu từ Master
```sh
rm -rf /var/lib/postgresql/data/*
pg_basebackup -h postgres_master -D /var/lib/postgresql/data -U replicator2 -Fp -Xs -P -R
```

## 5. Kiểm Tra Hoạt Động Của Replication

### 5.1 Thêm dữ liệu vào Master
```sh
docker exec -it postgres_master psql -U admin -c "CREATE TABLE test_table (id SERIAL PRIMARY KEY, message TEXT);"
docker exec -it postgres_master psql -U admin -c "INSERT INTO test_table (message) VALUES ('Hello from Master');"
```

### 5.2 Kiểm tra dữ liệu trên Slave
```sh
docker exec -it postgres_slave psql -U admin -c "SELECT * FROM test_table;"
```
Nếu dữ liệu xuất hiện, replication đã hoạt động.

## 6. Giám Sát Replication

### 6.1 Kiểm tra trạng thái replication trên Master
```sh
docker exec -it postgres_master psql -U admin -c "SELECT * FROM pg_stat_replication;"
```

### 6.2 Theo dõi log
```sh
docker logs -f postgres_master
docker logs -f postgres_slave
```

## 7. Quản Lý Hệ Thống

### 7.1 Tắt hệ thống
```sh
docker stop postgres_slave
docker stop postgres_master
```

### 7.2 Khởi động lại hệ thống
```sh
docker start postgres_master
docker start postgres_slave
```

## 8. Tối Ưu Hóa Hệ Thống
- **Tăng kích thước WAL**: Điều chỉnh `wal_keep_size` trong `postgresql.conf` nếu cần lưu trữ nhiều WAL hơn.
- **Giảm checkpoint quá thường xuyên**: Điều chỉnh `checkpoint_timeout` và `max_wal_size`.
- **Giám sát lâu dài**: Dùng `pgAdmin`, `Prometheus` để theo dõi.

Ví dụ chỉnh sửa file cấu hình:
```sh
echo "checkpoint_timeout = 10min" >> /var/lib/postgresql/data/postgresql.conf
echo "max_wal_size = 2GB" >> /var/lib/postgresql/data/postgresql.conf
```

## 9. Tổng Kết
- **Master**:
  - Cấu hình replication trong `postgresql.conf` và `pg_hba.conf`.
  - Tạo user replication.
- **Slave**:
  - Xóa dữ liệu cũ.
  - Sao chép dữ liệu từ Master bằng `pg_basebackup`.
- **Kiểm tra và giám sát** để đảm bảo replication hoạt động ổn định.

