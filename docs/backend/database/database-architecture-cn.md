# Mini Exchange Database Architecture

## 概述

本文檔描述了 Mini Exchange Backend 的完整數據庫架構設計，基於 PostgreSQL 15+ 構建，採用微服務 DDD 架構模式。

## 設計原則

### 1. 業務領域劃分 (DDD)
- **用戶認證域**: 用戶管理、角色權限、KYC認證
- **資產管理域**: 資產定義、賬戶餘額、交易記錄、複式記帳
- **訂單管理域**: 訂單生命週期、交易對配置
- **撮合引擎域**: 成交記錄、市場數據
- **風控域**: 風險規則、風險事件
- **清算結算域**: 批次清算、結算明細
- **通知域**: 消息模板、通知記錄
- **審計域**: 操作日誌、事件溯源

### 2. 數據完整性保障
- **ACID 事務**: 所有資金操作支持強一致性
- **外鍵約束**: 確保數據引用完整性
- **檢查約束**: 餘額非負、金額合理性驗證
- **唯一約束**: 防止重複數據

### 3. 性能優化策略
- **動態分區表**: 按月分區的大表 (orders, trades, klines, notifications, audit_logs)
- **分區管理函數**: 自動分區創建和管理
- **索引優化**: 針對查詢模式優化的複合索引
- **更新觸發器**: 自動維護 `updated_at` 字段
- **數據約束**: 全面的數據完整性約束

## 核心表結構

### 用戶認證域 (Auth Domain)

#### users - 用戶基本信息
```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    phone VARCHAR(20),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    kyc_level INTEGER NOT NULL DEFAULT 0,
    last_login_at TIMESTAMP WITH TIME ZONE,
    failed_login_attempts INTEGER NOT NULL DEFAULT 0,
    locked_until TIMESTAMP WITH TIME ZONE,
    version INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

**關鍵特性**:
- UUID 作為外部引用標識
- 密碼哈希存儲 (BCrypt)
- KYC 等級控制 (0-2)
- 樂觀鎖版本控制

#### roles & permissions - RBAC 權限系統
```sql
CREATE TABLE roles (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    description TEXT
);

CREATE TABLE permissions (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    resource VARCHAR(50) NOT NULL,
    action VARCHAR(50) NOT NULL
);
```

**權限模型**: Resource-Action 模式，支援細粒度權限控制

### 資產管理域 (Account & Asset Domain)

#### assets - 資產定義
```sql
CREATE TABLE assets (
    id SERIAL PRIMARY KEY,
    symbol VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(20) NOT NULL, -- CRYPTO, FIAT
    decimals INTEGER NOT NULL DEFAULT 8,
    min_withdraw_amount DECIMAL(36,18) NOT NULL DEFAULT 0,
    withdraw_fee DECIMAL(36,18) NOT NULL DEFAULT 0
);
```

**資產類型**: 支持加密貨幣和法幣，精度可配置

#### accounts - 用戶資產賬戶
```sql
CREATE TABLE accounts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    asset_id INTEGER NOT NULL REFERENCES assets(id),
    available_balance DECIMAL(36,18) NOT NULL DEFAULT 0 CHECK (available_balance >= 0),
    frozen_balance DECIMAL(36,18) NOT NULL DEFAULT 0 CHECK (frozen_balance >= 0),
    total_balance DECIMAL(36,18) GENERATED ALWAYS AS (available_balance + frozen_balance) STORED,
    version INTEGER NOT NULL DEFAULT 0,
    UNIQUE(user_id, asset_id)
);
```

**餘額管理**: 
- 可用餘額 + 凍結餘額 = 總餘額
- 非負約束確保資金安全
- 樂觀鎖防止並發問題

#### transactions - 交易記錄 (複式記帳)
```sql
CREATE TABLE transactions (
    id BIGSERIAL PRIMARY KEY,
    uuid UUID NOT NULL DEFAULT uuid_generate_v4(),
    user_id BIGINT NOT NULL REFERENCES users(id),
    asset_id INTEGER NOT NULL REFERENCES assets(id),
    type VARCHAR(20) NOT NULL,
    amount DECIMAL(36,18) NOT NULL,
    balance_before DECIMAL(36,18) NOT NULL,
    balance_after DECIMAL(36,18) NOT NULL,
    reference_type VARCHAR(20),
    reference_id BIGINT
);
```

**記帳原則**: 每筆資金變動都有完整記錄，支持審計和對賬

### 訂單管理域 (Order Domain)

#### trading_pairs - 交易對配置
```sql
CREATE TABLE trading_pairs (
    id SERIAL PRIMARY KEY,
    symbol VARCHAR(20) NOT NULL UNIQUE,
    base_asset_id INTEGER NOT NULL REFERENCES assets(id),
    quote_asset_id INTEGER NOT NULL REFERENCES assets(id),
    min_order_amount DECIMAL(36,18) NOT NULL DEFAULT 0,
    price_precision INTEGER NOT NULL DEFAULT 8,
    amount_precision INTEGER NOT NULL DEFAULT 8,
    taker_fee_rate DECIMAL(10,6) NOT NULL DEFAULT 0.001,
    maker_fee_rate DECIMAL(10,6) NOT NULL DEFAULT 0.001
);
```

**交易配置**: 精度控制、最小訂單量、手續費率配置

#### orders - 訂單表 (分區表)
```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    uuid UUID NOT NULL DEFAULT uuid_generate_v4(),
    user_id BIGINT NOT NULL REFERENCES users(id),
    trading_pair_id INTEGER NOT NULL REFERENCES trading_pairs(id),
    type VARCHAR(20) NOT NULL, -- MARKET, LIMIT, STOP_LOSS
    side VARCHAR(10) NOT NULL, -- BUY, SELL
    amount DECIMAL(36,18) NOT NULL CHECK (amount > 0),
    price DECIMAL(36,18),
    filled_amount DECIMAL(36,18) NOT NULL DEFAULT 0,
    remaining_amount DECIMAL(36,18) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    version INTEGER NOT NULL DEFAULT 0
) PARTITION BY RANGE (created_at);
```

**分區策略**: 按月分區，提高查詢性能和維護效率

#### trades - 成交記錄 (分區表)
```sql
CREATE TABLE trades (
    id BIGSERIAL PRIMARY KEY,
    uuid UUID NOT NULL DEFAULT uuid_generate_v4(),
    trading_pair_id INTEGER NOT NULL REFERENCES trading_pairs(id),
    buyer_order_id BIGINT NOT NULL,
    seller_order_id BIGINT NOT NULL,
    buyer_user_id BIGINT NOT NULL REFERENCES users(id),
    seller_user_id BIGINT NOT NULL REFERENCES users(id),
    price DECIMAL(36,18) NOT NULL,
    amount DECIMAL(36,18) NOT NULL,
    buyer_fee DECIMAL(36,18) NOT NULL DEFAULT 0,
    seller_fee DECIMAL(36,18) NOT NULL DEFAULT 0,
    is_maker_buyer BOOLEAN NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
) PARTITION BY RANGE (created_at);
```

**成交記錄**: 完整的買賣雙方信息，支持 Maker/Taker 手續費模式

### 市場數據域 (Market Data Domain)

#### klines - K線數據 (分區表)
```sql
CREATE TABLE klines (
    id BIGSERIAL PRIMARY KEY,
    trading_pair_id INTEGER NOT NULL REFERENCES trading_pairs(id),
    interval VARCHAR(10) NOT NULL, -- 1m, 5m, 15m, 1h, 4h, 1d
    open_time TIMESTAMP WITH TIME ZONE NOT NULL,
    open_price DECIMAL(36,18) NOT NULL,
    high_price DECIMAL(36,18) NOT NULL,
    low_price DECIMAL(36,18) NOT NULL,
    close_price DECIMAL(36,18) NOT NULL,
    volume DECIMAL(36,18) NOT NULL DEFAULT 0,
    quote_volume DECIMAL(36,18) NOT NULL DEFAULT 0,
    trades_count INTEGER NOT NULL DEFAULT 0,
    UNIQUE(trading_pair_id, interval, open_time)
) PARTITION BY RANGE (open_time);
```

**K線管理**: 多時間維度、高性能查詢、唯一性約束

#### ticker_24hr - 24小時統計
```sql
CREATE TABLE ticker_24hr (
    trading_pair_id INTEGER PRIMARY KEY REFERENCES trading_pairs(id),
    open_price DECIMAL(36,18) NOT NULL,
    high_price DECIMAL(36,18) NOT NULL,
    low_price DECIMAL(36,18) NOT NULL,
    close_price DECIMAL(36,18) NOT NULL,
    volume DECIMAL(36,18) NOT NULL DEFAULT 0,
    price_change_percent DECIMAL(10,4) NOT NULL DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

**實時統計**: 快速訪問的24小時交易統計

### 風控域 (Risk Management Domain)

#### risk_rules - 風控規則
```sql
CREATE TABLE risk_rules (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50) NOT NULL,
    conditions JSONB NOT NULL,
    actions JSONB NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    priority INTEGER NOT NULL DEFAULT 100
);
```

**規則引擎**: JSON 配置的靈活風控規則系統

#### risk_events - 風控事件
```sql
CREATE TABLE risk_events (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    rule_id INTEGER REFERENCES risk_rules(id),
    type VARCHAR(50) NOT NULL,
    level VARCHAR(20) NOT NULL, -- LOW, MEDIUM, HIGH, CRITICAL
    details JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
);
```

**事件管理**: 結構化的風控事件記錄和處理流程

### Outbox Pattern 事件表

#### outbox_events - 事件發佈
```sql
CREATE TABLE outbox_events (
    id BIGSERIAL PRIMARY KEY,
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(100) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    correlation_id UUID,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING'
);
```

**事件溯源**: 支持可靠的事件發佈和微服務間通信

## 索引策略

### 主要索引設計

```sql
-- 用戶查詢優化
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_status ON users(status);

-- 賬戶查詢優化
CREATE INDEX idx_accounts_user_id ON accounts(user_id);
CREATE INDEX idx_transactions_user_id_created_at ON transactions(user_id, created_at DESC);

-- 訂單查詢優化
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_trading_pair_id ON orders(trading_pair_id);
CREATE INDEX idx_orders_price_side ON orders(trading_pair_id, side, price, created_at);

-- 成交查詢優化
CREATE INDEX idx_trades_trading_pair_id ON trades(trading_pair_id);
CREATE INDEX idx_trades_created_at ON trades(created_at DESC);

-- K線查詢優化
CREATE INDEX idx_klines_trading_pair_interval ON klines(trading_pair_id, interval, open_time);
```

### 分區表索引

每個分區自動繼承主表索引，同時支持分區剪枝優化。

## 視圖定義

### 業務視圖

#### user_balances - 用戶餘額視圖
```sql
CREATE VIEW user_balances AS
SELECT 
    u.id as user_id,
    u.username,
    a.symbol as asset_symbol,
    acc.available_balance,
    acc.frozen_balance,
    acc.total_balance
FROM users u
JOIN accounts acc ON u.id = acc.user_id
JOIN assets a ON acc.asset_id = a.id
WHERE u.status = 'ACTIVE' AND a.is_active = true;
```

#### trading_pair_stats - 交易對統計視圖
```sql
CREATE VIEW trading_pair_stats AS
SELECT 
    tp.symbol,
    base.symbol as base_asset,
    quote.symbol as quote_asset,
    COUNT(o.id) as total_orders,
    COUNT(t.id) as total_trades,
    COALESCE(SUM(t.amount), 0) as total_volume
FROM trading_pairs tp
LEFT JOIN assets base ON tp.base_asset_id = base.id
LEFT JOIN assets quote ON tp.quote_asset_id = quote.id
LEFT JOIN orders o ON tp.id = o.trading_pair_id
LEFT JOIN trades t ON tp.id = t.trading_pair_id
GROUP BY tp.id, tp.symbol, base.symbol, quote.symbol;
```

## 觸發器和函數

### 自動更新觸發器
```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';
```

### 餘額校驗觸發器
```sql
CREATE OR REPLACE FUNCTION check_account_balance()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.available_balance < 0 OR NEW.frozen_balance < 0 THEN
        RAISE EXCEPTION '賬戶餘額不能為負數';
    END IF;
    RETURN NEW;
END;
$$ language 'plpgsql';
```

## 性能優化建議

### 1. 查詢優化
- 使用預編譯語句減少解析開銷
- 合理使用 LIMIT 和 OFFSET 進行分頁
- 避免 SELECT * 查詢
- 使用 EXPLAIN ANALYZE 分析查詢計劃

### 2. 分區管理
```sql
-- 自動創建未來分區的函數
CREATE OR REPLACE FUNCTION create_monthly_partitions()
RETURNS void AS $$
DECLARE
    start_date date;
    end_date date;
    table_name text;
BEGIN
    -- 創建未來3個月的分區
    FOR i IN 0..2 LOOP
        start_date := date_trunc('month', CURRENT_DATE + (i || ' months')::interval);
        end_date := start_date + interval '1 month';
        
        -- 創建 orders 分區
        table_name := 'orders_' || to_char(start_date, 'YYYY_MM');
        EXECUTE format('CREATE TABLE IF NOT EXISTS %I PARTITION OF orders 
                       FOR VALUES FROM (%L) TO (%L)', 
                       table_name, start_date, end_date);
        
        -- 創建 trades 分區
        table_name := 'trades_' || to_char(start_date, 'YYYY_MM');
        EXECUTE format('CREATE TABLE IF NOT EXISTS %I PARTITION OF trades 
                       FOR VALUES FROM (%L) TO (%L)', 
                       table_name, start_date, end_date);
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### 3. 連接池配置
- 設置合適的連接池大小 (CPU 核心數 * 2 + 1)
- 配置連接超時和空閒回收
- 監控連接池使用情況

### 4. 監控指標
- 慢查詢監控 (log_min_duration_statement)
- 連接數監控
- 緩存命中率監控
- 磁盤使用量監控

## 備份和恢復策略

### 1. 物理備份
```bash
# 全量備份
pg_basebackup -D /backup/full_backup -Ft -z -P

# 增量備份 (WAL 歸檔)
archive_command = 'cp %p /backup/wal_archive/%f'
```

### 2. 邏輯備份
```bash
# 完整數據庫備份
pg_dump -h localhost -U postgres -d mini_exchange > backup.sql

# 僅結構備份
pg_dump -h localhost -U postgres -d mini_exchange --schema-only > schema.sql

# 僅數據備份
pg_dump -h localhost -U postgres -d mini_exchange --data-only > data.sql
```

### 3. 災難恢復
- 配置流復制 (Streaming Replication)
- 設置自動故障轉移
- 定期恢復測試

## 安全考慮

### 1. 數據加密
- 啟用 TLS 連接加密
- 敏感欄位應用級加密
- 透明數據加密 (TDE)

### 2. 訪問控制
```sql
-- 創建應用專用用戶
CREATE USER mini_exchange_app WITH PASSWORD 'strong_password';

-- 授予必要權限
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO mini_exchange_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO mini_exchange_app;

-- 撤銷危險權限
REVOKE CREATE ON SCHEMA public FROM mini_exchange_app;
```

### 3. 審計日誌
- 啟用 PostgreSQL 審計擴展
- 記錄所有 DDL 操作
- 監控異常訪問模式

## 維護操作

### 1. 定期維護任務
```sql
-- 更新統計信息
ANALYZE;

-- 重建索引
REINDEX DATABASE mini_exchange;

-- 清理舊分區
DROP TABLE IF EXISTS orders_2023_01;
DROP TABLE IF EXISTS trades_2023_01;
```

### 2. 監控腳本
- 檢查分區創建狀態
- 監控表大小增長
- 檢查索引使用情況
- 監控慢查詢

這個數據庫架構設計為 Mini Exchange Backend 提供了完整的數據存儲解決方案，支持高並發交易、實時風控、可靠的資金管理和全面的審計追蹤。
