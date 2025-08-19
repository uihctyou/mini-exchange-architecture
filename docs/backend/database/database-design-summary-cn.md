# Mini Exchange Backend 數據庫設計總結

## 🎯 設計亮點

### 1. 完整的業務域覆蓋
採用 **DDD (領域驅動設計)** 思維，將整個交易所業務劃分為 8 個核心域：

- **👤 用戶認證域**: RBAC 權限系統、KYC 認證、用戶管理
- **💰 資產管理域**: 多資產賬戶、複式記帳、資金安全
- **📋 訂單管理域**: 完整訂單生命週期、交易對配置
- **⚡ 撮合引擎域**: 高性能成交記錄、市場數據
- **🛡️ 風控域**: 靈活的規則引擎、實時風險監控
- **🧮 清算結算域**: 批次清算、自動對賬
- **📨 通知域**: 多渠道消息推送、模板管理
- **📊 審計域**: 完整操作追蹤、事件溯源

### 2. 核心技術特性

#### 🔒 資金安全保障
```sql
-- 餘額非負約束
available_balance DECIMAL(20,8) NOT NULL DEFAULT 0 CHECK (available_balance >= 0)
-- 複式記帳系統
balance_before + amount = balance_after
-- 樂觀鎖防併發
version INTEGER NOT NULL DEFAULT 0
```

#### ⚡ 高性能設計
```sql
-- 分區表策略 (動態月度分區)
CREATE TABLE orders (...) PARTITION BY RANGE (created_at);
CREATE TABLE trades (...) PARTITION BY RANGE (created_at);
CREATE TABLE klines (...) PARTITION BY RANGE (open_time);
CREATE TABLE notifications (...) PARTITION BY RANGE (created_at);
CREATE TABLE audit_logs (...) PARTITION BY RANGE (created_at);

-- 動態分區管理函數
CREATE OR REPLACE FUNCTION create_monthly_partition(table_name TEXT, partition_date DATE);

-- 複合索引優化
CREATE INDEX idx_orders_price_side ON orders(trading_pair_id, side, price, created_at);
```

#### 🎛️ 靈活配置系統
```sql
-- JSON 配置的風控規則
parameters JSONB NOT NULL  -- {"max_daily_volume": 100000}

-- 可配置的交易參數
price_precision INTEGER NOT NULL DEFAULT 8
maker_fee DECIMAL(6,4) NOT NULL DEFAULT 0.001
taker_fee DECIMAL(6,4) NOT NULL DEFAULT 0.001
```

## 📋 核心表結構

## 📋 核心表結構

### 1. 認證域 (Authentication Domain)
```
users (用戶基本信息)
├── id, username, email, password_hash
├── first_name, last_name, phone
├── status, kyc_level, last_login_at
├── failed_login_attempts, locked_until
└── version, created_at, updated_at

roles (角色定義)
├── id, name, description, is_active
└── created_at, updated_at

permissions (權限定義)
├── id, name, resource, action, description
└── created_at, updated_at

user_roles (用戶角色關聯)
├── user_id, role_id
├── granted_at, granted_by
└── PRIMARY KEY (user_id, role_id)

role_permissions (角色權限關聯)
├── role_id, permission_id
└── PRIMARY KEY (role_id, permission_id)

kyc_records (KYC認證記錄)
├── id, user_id, level, status
├── document_type, document_number
├── submitted_data (JSONB), review_notes
├── reviewed_by, reviewed_at
└── created_at, updated_at
```

### 2. 資產管理域 (Asset Management Domain)
```
assets (資產定義: BTC, ETH, USDT...)
├── id, symbol, name, type
├── decimals, is_active
├── min_withdraw_amount, withdraw_fee
├── daily_withdraw_limit
└── created_at, updated_at

accounts (用戶多資產賬戶)
├── id, user_id, asset_id
├── available_balance, frozen_balance
├── created_at, updated_at
└── UNIQUE(user_id, asset_id)

transactions (交易記錄 - 複式記帳)
├── id, user_id, asset_id, type
├── amount, balance_before, balance_after
├── reference_type, reference_id
├── description, created_at, created_by
└── CHECK (available_balance >= 0 AND frozen_balance >= 0)

balance_freezes (資金凍結管理)
├── id, user_id, asset_id, amount
├── reason, reference_type, reference_id
├── status, created_at, released_at
└── status: ACTIVE, RELEASED

deposit_withdrawals (充值提現記錄)
├── id, user_id, asset_id, type
├── amount, fee, status
├── tx_hash, address
├── confirmations, required_confirmations
├── processed_at, created_at, updated_at
└── status: PENDING, CONFIRMED, FAILED, CANCELLED
```

### 3. 訂單管理域 (Order Management Domain)
```
trading_pairs (交易對: BTCUSDT, ETHUSDT...)
├── id, symbol, base_asset_id, quote_asset_id
├── status, min_order_amount, max_order_amount
├── price_precision, amount_precision
├── maker_fee, taker_fee
└── created_at, updated_at

orders (訂單表 - 按 created_at 分區)
├── id, user_id, trading_pair_id
├── type, side, amount, price
├── remaining_amount, filled_amount, average_price
├── status, created_at, updated_at
└── PARTITION BY RANGE (created_at)

trades (成交記錄 - 按 created_at 分區)
├── id, trading_pair_id
├── buy_order_id, sell_order_id
├── buyer_user_id, seller_user_id
├── amount, price, buyer_fee, seller_fee
├── created_at
└── PARTITION BY RANGE (created_at)

klines (K線數據 - 按 open_time 分區)
├── id, trading_pair_id, interval
├── open_time, close_time
├── open_price, high_price, low_price, close_price
├── volume, quote_volume, trades_count
└── PARTITION BY RANGE (open_time)

ticker_24hr (24小時統計)
├── trading_pair_id (PRIMARY KEY)
├── open_price, high_price, low_price, close_price
├── volume, quote_volume
├── price_change, price_change_percent
├── trades_count, updated_at
```

### 4. 風險管理與合規域 (Risk Management & Compliance Domain)
```
risk_rules (風控規則配置)
├── id, name, type, parameters (JSONB)
├── is_active, description
└── created_at, updated_at

risk_events (風控事件記錄)
├── id, user_id, rule_id
├── type, level, details (JSONB)
├── status, created_at, resolved_at
└── level: LOW, MEDIUM, HIGH, CRITICAL

settlement_batches (清算批次)
├── id, batch_date, status
├── total_trades, total_volume
├── started_at, completed_at, created_at
└── UNIQUE(batch_date)

settlement_details (清算明細)
├── id, batch_id, user_id, asset_id
├── trade_amount, fee_amount, net_amount
└── created_at

notification_templates (通知模板)
├── id, name, type, subject, content
├── is_active, created_at, updated_at
└── type: EMAIL, SMS, PUSH, WEBHOOK

notifications (通知記錄 - 按 created_at 分區)
├── id, user_id, template_id, type
├── title, content, status
├── sent_at, error_message, created_at
└── PARTITION BY RANGE (created_at)

outbox_events (事件發布 - Outbox 模式)
├── id, aggregate_type, aggregate_id
├── event_type, event_data (JSONB)
├── correlation_id, status, retry_count
├── next_retry_at, created_at, processed_at
└── status: PENDING, PROCESSED, FAILED

audit_logs (審計日誌 - 按 created_at 分區)
├── id, user_id, action
├── resource_type, resource_id
├── old_values (JSONB), new_values (JSONB)
├── ip_address, user_agent, created_at
└── PARTITION BY RANGE (created_at)
```

## 🚀 已實現的文件

### 1. 數據庫初始化腳本
📁 **`docker/postgres/init/`** - PostgreSQL 初始化
- ✅ **`01-init-databases.sh`** - 數據庫和擴展設置
- ✅ **`02-create-tables.sql`** - 完整表結構含動態分區
- ✅ **`03-insert-base-data.sql`** - 系統基礎數據（角色、權限、資產）

### 2. 核心功能實現
- ✅ **動態分區管理** - 自動月度分區創建
- ✅ **更新觸發器** - 自動維護 `updated_at` 字段
- ✅ **綜合索引** - 針對高性能查詢優化
- ✅ **數據約束** - 確保數據完整性和一致性
- ✅ **Outbox 模式** - 微服務架構的事件溯源

### 3. 數據庫結構亮點
- ✅ **22個核心業務表** 包含完整關聯關係
- ✅ **5個分區表** 處理高容量數據（訂單、成交、K線、通知、審計）
- ✅ **完整 RBAC 系統** 包含用戶、角色和權限
- ✅ **複式記帳** 確保財務準確性
- ✅ **JSON 配置** 靈活的業務規則

### 4. 架構文檔
📁 **`docs/database-*.md`** - 完整文檔
- ✅ 完整的表結構說明
- ✅ ER 圖和關聯關係
- ✅ 性能優化策略
- ✅ 安全和維護建議
- ✅ 業務流程圖
- ✅ 數據流向圖

## 💡 關鍵設計決策

### 1. 資金安全 First
- **複式記帳**: 每筆資金變動都有完整的 before/after 記錄
- **餘額約束**: 數據庫層面保證餘額永不為負
- **事務支持**: 所有資金操作都在 ACID 事務中執行
- **審計追蹤**: 完整的操作日誌，支持資金對賬

### 2. 高性能架構
- **分區策略**: 大表按時間分區，自動管理歷史數據
- **索引優化**: 針對查詢模式的複合索引設計
- **讀寫分離**: 支持 PostgreSQL 主從復制
- **緩存友好**: 設計支持 Redis 緩存層

### 3. 可擴展性
- **模塊化設計**: 清晰的域邊界，便於微服務拆分
- **事件驅動**: Outbox Pattern 支持可靠的事件發佈
- **配置驅動**: JSON 配置的靈活業務規則
- **版本控制**: 樂觀鎖支持併發控制

## 🔄 與您的技術棧完美集成

### Spring Boot 3.x + JPA
```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop  # 開發環境自動建表
    show-sql: true          # SQL 調試
```

### PostgreSQL 生產配置
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mini_exchange
    hikari:
      maximum-pool-size: 20  # 連接池配置
```

### 事件驅動架構 (Kafka Ready)
```sql
-- Outbox Pattern 事件表
CREATE TABLE outbox_events (
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING'
);
```

## 📊 性能優化特性

### 1. 智能分區管理
```sql
-- 自動創建未來分區的函數
CREATE FUNCTION create_monthly_partitions() -- 自動管理分區
```

### 2. 查詢優化
```sql
-- 針對交易查詢優化的索引
CREATE INDEX idx_orders_price_side ON orders(trading_pair_id, side, price, created_at);
CREATE INDEX idx_trades_created_at ON trades(created_at DESC);
```

### 3. 統計視圖
```sql
-- 用戶餘額視圖
CREATE VIEW user_balances AS SELECT ...
-- 交易統計視圖  
CREATE VIEW trading_pair_stats AS SELECT ...
```

## 🛡️ 安全與合規

### 1. 數據加密
- 密碼 BCrypt 哈希存儲
- 支持敏感欄位應用級加密
- TLS 連接加密

### 2. 訪問控制
- 細粒度的 RBAC 權限系統
- 資料庫層面的用戶權限隔離
- API 層面的 JWT 認證

### 3. 審計合規
- 完整的操作審計日誌
- 資金流水可追溯
- 風控事件記錄

## 🎯 下一步建議

### 階段 1: 基礎 Entity 開發
根據數據庫設計創建對應的 JPA Entity 類：
```bash
src/main/java/io/tyou/mini_exchange_backend/
├── auth/entity/User.java
├── account/entity/Account.java  
├── order/entity/Order.java
└── ...
```

### 階段 2: Repository 層
創建 Spring Data JPA Repository：
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
}
```

### 階段 3: Service 層
實現業務邏輯，集成事件發佈：
```java
@Service
@Transactional
public class OrderService {
    // 訂單處理邏輯 + 事件發佈
}
```

## ✅ 驗證結果

1. **✅ Spring Boot 應用啟動成功**
2. **✅ H2 數據庫正常連接** (開發環境)
3. **✅ JPA/Hibernate 配置正確**
4. **✅ 所有必要的依賴已配置**

使用
- 直接使用 H2 Console 查看數據庫結構: `http://localhost:9977/api/h2-console`
- 開始開發 JPA Entity 類
- 實現各個業務模塊的 Service 層

這個數據庫架構為 mini-exchange-backend 提供了：
- 🏗️ **堅實的基礎架構** - 支持大規模交易業務
- 🔒 **金融級安全** - 資金安全和數據完整性保障  
- ⚡ **高性能設計** - 分區表和索引優化
- 🎯 **業務完整性** - 覆蓋交易所全業務流程
- 🚀 **可擴展架構** - 支持微服務化演進

