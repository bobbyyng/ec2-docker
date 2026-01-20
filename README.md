# ec2-docker

PostgreSQL 18 Docker 設定，用於在 EC2 上部署。

## 專案結構

```
ec2-docker/
├── docker-compose.yml          # Docker Compose 配置檔案
├── .env.example                # 環境變數範例
├── .dockerignore               # Docker 忽略檔案
├── postgres/                   # PostgreSQL 服務資料夾
│   └── config/                 # PostgreSQL 配置檔案
│       ├── postgresql.conf     # PostgreSQL 主配置
│       └── pg_hba.conf         # 用戶認證配置
└── README.md                   # 本檔案
```

未來可以為其他服務創建類似的資料夾結構，例如：
- `redis/` - Redis 服務
- `nginx/` - Nginx 服務
- `app/` - 應用程式服務

## 快速開始

### 1. 設定環境變數

複製環境變數範例檔案並設定你的值：

```bash
cp .env.example .env
```

編輯 `.env` 檔案，設定你的資料庫認證資訊：

```bash
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_secure_password_here
POSTGRES_DB=postgres
POSTGRES_PORT=5432
TZ=Asia/Hong_Kong  # 時區設定（可選，預設為 UTC）
```

### 2. 啟動 PostgreSQL

```bash
docker-compose up -d
```

### 3. 檢查服務狀態

```bash
docker-compose ps
```

### 4. 查看日誌

```bash
docker-compose logs -f postgres
```

## 常用指令

### 啟動服務
```bash
docker-compose up -d
```

### 停止服務
```bash
docker-compose down
```

### 停止服務並刪除資料（⚠️ 警告：會刪除所有資料）
```bash
docker-compose down -v
```

### 查看服務狀態
```bash
docker-compose ps
```

### 查看日誌
```bash
# 查看所有日誌
docker-compose logs

# 查看並跟隨日誌
docker-compose logs -f postgres

# 查看最近 100 行日誌
docker-compose logs --tail=100 postgres
```

### 進入 PostgreSQL 容器
```bash
docker-compose exec postgres bash
```

### 連接資料庫
```bash
# 使用 psql 連接
docker-compose exec postgres psql -U postgres -d postgres

# 或從外部連接（如果端口已暴露）
psql -h localhost -p 5432 -U postgres -d postgres
```

### 重啟服務
```bash
docker-compose restart postgres
```

## 配置說明

### PostgreSQL 配置 (postgresql.conf)

`postgres/config/postgresql.conf` 檔案包含 PostgreSQL 的主要配置設定，包括：

- **連接池設定**：`max_connections`（預設 100）
- **記憶體設定**：`shared_buffers`、`effective_cache_size`、`work_mem` 等
- **查詢調優**：`random_page_cost`、`effective_io_concurrency` 等
- **WAL 設定**：`wal_buffers`、`min_wal_size`、`max_wal_size` 等
- **日誌設定**：查詢日誌、錯誤日誌等

#### 調整配置以適應 EC2 實例規格

根據你的 EC2 實例記憶體大小，建議調整以下參數：

- **小型實例（1-2GB RAM）**：
  - `shared_buffers = 256MB`
  - `effective_cache_size = 1GB`
  - `work_mem = 4MB`

- **中型實例（4-8GB RAM）**：
  - `shared_buffers = 1GB`
  - `effective_cache_size = 3GB`
  - `work_mem = 16MB`

- **大型實例（16GB+ RAM）**：
  - `shared_buffers = 4GB`
  - `effective_cache_size = 12GB`
  - `work_mem = 32MB`

修改 `postgres/config/postgresql.conf` 後，需要重啟服務：

```bash
docker-compose restart postgres
```

#### 時區設定

有兩種方式可以設定時區：

**方法 1：使用環境變數（推薦）**

在 `.env` 檔案中設定 `TZ` 環境變數：

```bash
TZ=Asia/Hong_Kong
```

常用時區範例：
- `Asia/Hong_Kong` - 香港時區
- `Asia/Taipei` - 台北時區
- `Asia/Shanghai` - 上海時區
- `UTC` - 協調世界時（預設）
- `America/New_York` - 美國東部時區
- `Europe/London` - 倫敦時區

**方法 2：修改 PostgreSQL 配置檔案**

直接修改 `postgres/config/postgresql.conf` 中的 `timezone` 設定：

```conf
timezone = 'Asia/Hong_Kong'
```

**驗證時區設定**

設定完成後，重啟服務並驗證：

```bash
# 重啟服務
docker-compose restart postgres

# 檢查容器系統時區
docker-compose exec postgres date

# 檢查 PostgreSQL 時區
docker-compose exec postgres psql -U postgres -c "SHOW timezone;"
```

### 用戶認證配置 (pg_hba.conf)

`postgres/config/pg_hba.conf` 檔案控制哪些主機可以連接、如何認證客戶端、可以使用哪些 PostgreSQL 用戶名，以及可以訪問哪些資料庫。

預設配置允許：
- 本地連接（Unix socket）
- Docker 網路連接
- 所有 IP 地址連接（使用 `scram-sha-256` 認證）

#### 安全建議

為了提高安全性，建議：

1. **限制 IP 訪問**：在 `postgres/config/pg_hba.conf` 中將 `0.0.0.0/0` 改為特定的 IP 範圍或 IP 地址
2. **使用 SSL 連接**：配置 SSL 並使用 `hostssl` 而非 `host`
3. **使用強密碼**：確保 `POSTGRES_PASSWORD` 使用強密碼

範例：只允許特定 IP 範圍連接

```
# 只允許特定 IP 範圍
host    all    all    192.168.1.0/24    scram-sha-256
host    all    all    10.0.0.0/8        scram-sha-256
```

修改 `postgres/config/pg_hba.conf` 後，需要重載配置：

```bash
docker-compose exec postgres pg_ctl reload
```

## 資料持久化

資料儲存在 Docker volume `postgres_data` 中，即使容器被刪除，資料也會保留。

### 備份資料

```bash
# 建立備份
docker-compose exec postgres pg_dump -U postgres postgres > backup.sql

# 或使用自定義格式
docker-compose exec postgres pg_dump -U postgres -Fc postgres > backup.dump
```

### 還原資料

```bash
# 從 SQL 檔案還原
docker-compose exec -T postgres psql -U postgres postgres < backup.sql

# 從自定義格式還原
docker-compose exec -T postgres pg_restore -U postgres -d postgres < backup.dump
```

## 故障排除

### 檢查容器健康狀態

```bash
docker-compose ps
```

健康檢查狀態應該顯示為 "healthy"。

### 查看詳細日誌

```bash
docker-compose logs postgres
```

### 檢查配置檔案是否正確載入

```bash
docker-compose exec postgres psql -U postgres -c "SHOW config_file;"
docker-compose exec postgres psql -U postgres -c "SHOW hba_file;"
```

### 測試連接

```bash
docker-compose exec postgres pg_isready -U postgres
```

## 注意事項

1. **安全性**：在生產環境中，請確保：
   - 使用強密碼
   - 限制 `pg_hba.conf` 中的 IP 訪問範圍
   - 考慮使用 SSL/TLS 連接
   - 定期備份資料

2. **效能調優**：根據 EC2 實例規格調整 `postgresql.conf` 中的記憶體和連接池設定

3. **防火牆**：確保 EC2 安全群組允許 PostgreSQL 端口（預設 5432）的連接

4. **資源限制**：可以透過 `docker-compose.yml` 添加資源限制（memory、cpus）以防止容器使用過多資源