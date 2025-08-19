# Mini Exchange Database ER Diagram

## 完整實體關係圖

```mermaid
erDiagram
    %% 用戶認證域 (Auth Domain)
    users {
        bigint id PK
        varchar username UK
        varchar email UK
        varchar password_hash
        varchar first_name
        varchar last_name
        varchar phone
        varchar status
        integer kyc_level
        timestamp last_login_at
        integer failed_login_attempts
        timestamp locked_until
        integer version
        timestamp created_at
        timestamp updated_at
    }
    
    roles {
        serial id PK
        varchar name UK
        text description
        boolean is_active
        timestamp created_at
        timestamp updated_at
    }
    
    permissions {
        serial id PK
        varchar name UK
        varchar resource
        varchar action
        text description
        timestamp created_at
        timestamp updated_at
    }
    
    user_roles {
        bigint user_id PK,FK
        integer role_id PK,FK
        timestamp granted_at
        bigint granted_by FK
    }
    
    role_permissions {
        integer role_id PK,FK
        integer permission_id PK,FK
    }
    
    kyc_records {
        bigserial id PK
        bigint user_id FK
        integer level
        varchar status
        varchar document_type
        varchar document_number
        jsonb submitted_data
        text review_notes
        bigint reviewed_by FK
        timestamp reviewed_at
        timestamp created_at
        timestamp updated_at
    }

    %% 資產管理域
    assets {
        serial id PK
        varchar symbol UK
        varchar name
        varchar type
        integer decimals
        boolean is_active
        decimal min_withdraw_amount
        decimal withdraw_fee
        decimal daily_withdraw_limit
        timestamp created_at
        timestamp updated_at
    }
    
    accounts {
        bigserial id PK
        bigint user_id FK
        integer asset_id FK
        decimal available_balance
        decimal frozen_balance
        timestamp created_at
        timestamp updated_at
    }
    
    transactions {
        bigserial id PK
        bigint user_id FK
        integer asset_id FK
        varchar type
        decimal amount
        decimal balance_before
        decimal balance_after
        varchar reference_type
        varchar reference_id
        text description
        timestamp created_at
        bigint created_by FK
    }
    
    balance_freezes {
        bigserial id PK
        bigint user_id FK
        integer asset_id FK
        decimal amount
        varchar reason
        varchar reference_type
        varchar reference_id
        varchar status
        timestamp created_at
        timestamp released_at
    }
    
    deposit_withdrawals {
        bigserial id PK
        bigint user_id FK
        integer asset_id FK
        varchar type
        decimal amount
        decimal fee
        varchar status
        varchar tx_hash
        varchar address
        integer confirmations
        integer required_confirmations
        timestamp processed_at
        timestamp created_at
        timestamp updated_at
    }

    %% 訂單管理域
    trading_pairs {
        serial id PK
        varchar symbol UK
        integer base_asset_id FK
        integer quote_asset_id FK
        varchar status
        decimal min_order_amount
        decimal max_order_amount
        integer price_precision
        integer amount_precision
        decimal maker_fee
        decimal taker_fee
        timestamp created_at
        timestamp updated_at
    }
    
    orders {
        bigserial id PK
        bigint user_id FK
        integer trading_pair_id FK
        varchar type
        varchar side
        decimal amount
        decimal price
        decimal remaining_amount
        decimal filled_amount
        decimal average_price
        varchar status
        timestamp created_at
        timestamp updated_at
    }
    
    trades {
        bigserial id PK
        integer trading_pair_id FK
        bigint buy_order_id FK
        bigint sell_order_id FK
        bigint buyer_user_id FK
        bigint seller_user_id FK
        decimal amount
        decimal price
        decimal buyer_fee
        decimal seller_fee
        timestamp created_at
    }
    
    klines {
        bigserial id PK
        integer trading_pair_id FK
        varchar interval
        timestamp open_time
        timestamp close_time
        decimal open_price
        decimal high_price
        decimal low_price
        decimal close_price
        decimal volume
        decimal quote_volume
        integer trades_count
    }
    
    ticker_24hr {
        integer trading_pair_id PK,FK
        decimal open_price
        decimal high_price
        decimal low_price
        decimal close_price
        decimal volume
        decimal quote_volume
        decimal price_change
        decimal price_change_percent
        integer trades_count
        timestamp updated_at
    }

    %% 風險管理域
    risk_rules {
        serial id PK
        varchar name UK
        varchar type
        jsonb parameters
        boolean is_active
        text description
        timestamp created_at
        timestamp updated_at
    }
    
    risk_events {
        bigserial id PK
        bigint user_id FK
        integer rule_id FK
        varchar type
        varchar level
        jsonb details
        varchar status
        timestamp created_at
        timestamp resolved_at
    }

    %% 清算結算域
    settlement_batches {
        bigserial id PK
        date batch_date UK
        varchar status
        integer total_trades
        decimal total_volume
        timestamp started_at
        timestamp completed_at
        timestamp created_at
    }
    
    settlement_details {
        bigserial id PK
        bigint batch_id FK
        bigint user_id FK
        integer asset_id FK
        decimal trade_amount
        decimal fee_amount
        decimal net_amount
        timestamp created_at
    }

    %% 通知域
    notification_templates {
        serial id PK
        varchar name UK
        varchar type
        varchar subject
        text content
        boolean is_active
        timestamp created_at
        timestamp updated_at
    }
    
    notifications {
        bigserial id PK
        bigint user_id FK
        integer template_id FK
        varchar type
        varchar title
        text content
        varchar status
        timestamp sent_at
        text error_message
        timestamp created_at
    }

    %% 系統表
    outbox_events {
        bigserial id PK
        varchar aggregate_type
        varchar aggregate_id
        varchar event_type
        jsonb event_data
        uuid correlation_id
        varchar status
        integer retry_count
        timestamp next_retry_at
        timestamp created_at
        timestamp processed_at
    }
    
    audit_logs {
        bigserial id PK
        bigint user_id FK
        varchar action
        varchar resource_type
        varchar resource_id
        jsonb old_values
        jsonb new_values
        inet ip_address
        text user_agent
        timestamp created_at
    }

    %% 關聯關係
    %% 認證域關聯
    users ||--o{ user_roles : "擁有角色"
    roles ||--o{ user_roles : "分配給用戶"
    roles ||--o{ role_permissions : "擁有權限"
    permissions ||--o{ role_permissions : "授予角色"
    users ||--o{ kyc_records : "提交KYC"
    users ||--o{ kyc_records : "審核KYC"
    users ||--o{ user_roles : "授予角色"

    %% 資產管理關聯
    users ||--o{ accounts : "擁有賬戶"
    assets ||--o{ accounts : "資產賬戶"
    users ||--o{ transactions : "創建交易"
    assets ||--o{ transactions : "資產交易"
    users ||--o{ transactions : "創建者"
    users ||--o{ balance_freezes : "凍結資金"
    assets ||--o{ balance_freezes : "凍結資產"
    users ||--o{ deposit_withdrawals : "發起充提"
    assets ||--o{ deposit_withdrawals : "充提資產"

    %% 訂單管理關聯
    assets ||--o{ trading_pairs : "基礎資產"
    assets ||--o{ trading_pairs : "計價資產"
    users ||--o{ orders : "下訂單"
    trading_pairs ||--o{ orders : "交易對訂單"
    trading_pairs ||--o{ trades : "交易對成交"
    orders ||--o{ trades : "買單"
    orders ||--o{ trades : "賣單"
    users ||--o{ trades : "買方"
    users ||--o{ trades : "賣方"
    trading_pairs ||--o{ klines : "價格數據"
    trading_pairs ||--|| ticker_24hr : "24小時統計"

    %% 風險管理關聯
    users ||--o{ risk_events : "觸發風險事件"
    risk_rules ||--o{ risk_events : "定義風險規則"

    %% 結算關聯
    settlement_batches ||--o{ settlement_details : "包含明細"
    users ||--o{ settlement_details : "用戶結算"
    assets ||--o{ settlement_details : "結算資產"

    %% 通知關聯
    users ||--o{ notifications : "接收通知"
    notification_templates ||--o{ notifications : "使用模板"

    %% 審計關聯
    users ||--o{ audit_logs : "用戶操作記錄"
```
    }
    
    kyc_records {
        bigserial id PK
        bigint user_id FK
        integer level
        varchar status
        varchar document_type
        varchar document_number
        timestamp submitted_at
        timestamp reviewed_at
        bigint reviewed_by FK
        text notes
        timestamp created_at
    }

    %% 資產管理域 (Asset Domain)
    assets {
        serial id PK
        varchar symbol UK
        varchar name
        varchar type
        integer decimals
        boolean is_active
        decimal min_withdraw_amount
        decimal max_withdraw_amount
        decimal withdraw_fee
        decimal deposit_fee
        timestamp created_at
        timestamp updated_at
    }
    
    accounts {
        bigserial id PK
        bigint user_id FK
        integer asset_id FK
        decimal available_balance
        decimal frozen_balance
        decimal total_balance
        bigint last_transaction_id
        timestamp created_at
        timestamp updated_at
        integer version
    }
    
    transactions {
        bigserial id PK
        uuid uuid UK
        bigint user_id FK
        integer asset_id FK
        varchar type
        decimal amount
        decimal balance_before
        decimal balance_after
        varchar reference_type
        bigint reference_id
        text description
        timestamp created_at
        bigint created_by FK
    }
    
    balance_freezes {
        bigserial id PK
        bigint account_id FK
        decimal amount
        varchar type
        bigint reference_id
        varchar status
        timestamp expires_at
        timestamp created_at
        timestamp released_at
    }
    
    deposit_withdrawals {
        bigserial id PK
        uuid uuid UK
        bigint user_id FK
        integer asset_id FK
        varchar type
        decimal amount
        decimal fee
        decimal net_amount
        varchar status
        varchar address
        varchar tx_hash
        integer confirmations
        integer required_confirmations
        timestamp processed_at
        timestamp created_at
        timestamp updated_at
    }

    %% 訂單管理域 (Order Domain)
    trading_pairs {
        serial id PK
        varchar symbol UK
        integer base_asset_id FK
        integer quote_asset_id FK
        varchar status
        decimal min_order_amount
        decimal max_order_amount
        decimal min_price
        decimal max_price
        integer price_precision
        integer amount_precision
        decimal taker_fee_rate
        decimal maker_fee_rate
        timestamp created_at
        timestamp updated_at
    }
    
    orders {
        bigserial id PK
        uuid uuid UK
        bigint user_id FK
        integer trading_pair_id FK
        varchar type
        varchar side
        decimal amount
        decimal price
        decimal filled_amount
        decimal remaining_amount
        decimal avg_price
        varchar status
        varchar time_in_force
        decimal stop_price
        decimal fee_paid
        integer fee_asset_id FK
        varchar client_order_id
        timestamp created_at
        timestamp updated_at
        timestamp filled_at
        timestamp cancelled_at
        integer version
    }
    
    trades {
        bigserial id PK
        uuid uuid UK
        integer trading_pair_id FK
        bigint buyer_order_id FK
        bigint seller_order_id FK
        bigint buyer_user_id FK
        bigint seller_user_id FK
        decimal price
        decimal amount
        decimal buyer_fee
        decimal seller_fee
        integer buyer_fee_asset_id FK
        integer seller_fee_asset_id FK
        boolean is_maker_buyer
        timestamp created_at
    }

    %% 市場數據域 (Market Data Domain)
    klines {
        bigserial id PK
        integer trading_pair_id FK
        varchar interval
        timestamp open_time
        timestamp close_time
        decimal open_price
        decimal high_price
        decimal low_price
        decimal close_price
        decimal volume
        decimal quote_volume
        integer trades_count
        timestamp created_at
    }
    
    ticker_24hr {
        integer trading_pair_id PK,FK
        decimal open_price
        decimal high_price
        decimal low_price
        decimal close_price
        decimal volume
        decimal quote_volume
        decimal price_change
        decimal price_change_percent
        integer trades_count
        timestamp updated_at
    }

    %% 風控域 (Risk Domain)
    risk_rules {
        serial id PK
        varchar name
        varchar type
        jsonb conditions
        jsonb actions
        boolean is_active
        integer priority
        timestamp created_at
        timestamp updated_at
    }
    
    risk_events {
        bigserial id PK
        bigint user_id FK
        integer rule_id FK
        varchar type
        varchar level
        jsonb details
        varchar status
        timestamp resolved_at
        bigint resolved_by FK
        timestamp created_at
    }

    %% 清算結算域 (Settlement Domain)
    settlement_batches {
        bigserial id PK
        date batch_date UK
        varchar status
        integer total_trades
        decimal total_volume
        timestamp started_at
        timestamp completed_at
        timestamp created_at
    }
    
    settlement_details {
        bigserial id PK
        bigint batch_id FK
        bigint user_id FK
        integer asset_id FK
        decimal trade_amount
        decimal fee_amount
        decimal net_amount
        timestamp created_at
    }

    %% 通知域 (Notification Domain)
    notification_templates {
        serial id PK
        varchar name UK
        varchar type
        varchar title
        text content
        jsonb variables
        boolean is_active
        timestamp created_at
    }
    
    notifications {
        bigserial id PK
        bigint user_id FK
        integer template_id FK
        varchar type
        varchar title
        text content
        varchar status
        timestamp sent_at
        timestamp read_at
        timestamp created_at
    }

    %% 事件域 (Event Domain)
    outbox_events {
        bigserial id PK
        varchar aggregate_type
        varchar aggregate_id
        varchar event_type
        jsonb event_data
        uuid correlation_id
        uuid causation_id
        timestamp created_at
        timestamp processed_at
        varchar status
    }

    %% 審計域 (Audit Domain)
    audit_logs {
        bigserial id PK
        bigint user_id FK
        varchar action
        varchar resource_type
        varchar resource_id
        jsonb old_values
        jsonb new_values
        inet ip_address
        text user_agent
        timestamp created_at
    }

    %% 關係定義
    users ||--o{ user_roles : "has"
    roles ||--o{ user_roles : "assigned to"
    roles ||--o{ role_permissions : "has"
    permissions ||--o{ role_permissions : "granted to"
    users ||--o{ kyc_records : "submits"
    users ||--o{ kyc_records : "reviews"
    
    users ||--o{ accounts : "owns"
    assets ||--o{ accounts : "denominated in"
    accounts ||--o{ balance_freezes : "has"
    users ||--o{ transactions : "executes"
    assets ||--o{ transactions : "involves"
    users ||--o{ transactions : "created by"
    users ||--o{ deposit_withdrawals : "initiates"
    assets ||--o{ deposit_withdrawals : "involves"
    
    assets ||--o{ trading_pairs : "base asset"
    assets ||--o{ trading_pairs : "quote asset"
    users ||--o{ orders : "places"
    trading_pairs ||--o{ orders : "for"
    assets ||--o{ orders : "fee asset"
    trading_pairs ||--o{ trades : "executed on"
    orders ||--o{ trades : "buyer order"
    orders ||--o{ trades : "seller order"
    users ||--o{ trades : "buyer"
    users ||--o{ trades : "seller"
    assets ||--o{ trades : "buyer fee asset"
    assets ||--o{ trades : "seller fee asset"
    
    trading_pairs ||--o{ klines : "generates"
    trading_pairs ||--|| ticker_24hr : "summarized by"
    
    users ||--o{ risk_events : "triggers"
    risk_rules ||--o{ risk_events : "matches"
    users ||--o{ risk_events : "resolves"
    
    settlement_batches ||--o{ settlement_details : "contains"
    users ||--o{ settlement_details : "involves"
    assets ||--o{ settlement_details : "denominated in"
    
    users ||--o{ notifications : "receives"
    notification_templates ||--o{ notifications : "uses"
    
    users ||--o{ audit_logs : "performs"
```

## 核心業務流程圖

### 1. 訂單處理流程

```mermaid
flowchart TD
    A[用戶下單] --> B[訂單驗證]
    B --> C{驗證通過?}
    C -->|否| D[返回錯誤]
    C -->|是| E[資金檢查]
    E --> F{餘額足夠?}
    F -->|否| G[餘額不足]
    F -->|是| H[凍結資金]
    H --> I[創建訂單記錄]
    I --> J[發送到撮合引擎]
    J --> K[撮合處理]
    K --> L{有匹配訂單?}
    L -->|否| M[進入訂單簿]
    L -->|是| N[執行交易]
    N --> O[更新訂單狀態]
    O --> P[資金結算]
    P --> Q[發送成交事件]
    Q --> R[通知用戶]
    
    style A fill:#e1f5fe
    style K fill:#ffecb3
    style N fill:#c8e6c9
    style Q fill:#f3e5f5
```

### 2. 資金管理流程

```mermaid
flowchart TD
    A[充值請求] --> B[生成充值地址]
    B --> C[監控鏈上交易]
    C --> D[確認到賬]
    D --> E[更新賬戶餘額]
    E --> F[記錄交易日誌]
    F --> G[發送充值事件]
    
    H[提現請求] --> I[資金檢查]
    I --> J{餘額足夠?}
    J -->|否| K[提現失敗]
    J -->|是| L[凍結提現資金]
    L --> M[風控檢查]
    M --> N{風控通過?}
    N -->|否| O[解凍資金]
    N -->|是| P[發起鏈上交易]
    P --> Q[等待確認]
    Q --> R[提現完成]
    R --> S[記錄交易日誌]
    
    style D fill:#c8e6c9
    style M fill:#ffcdd2
    style R fill:#c8e6c9
```

### 3. 風控處理流程

```mermaid
flowchart TD
    A[業務操作] --> B[觸發風控規則]
    B --> C[規則匹配]
    C --> D{匹配到規則?}
    D -->|否| E[正常處理]
    D -->|是| F[評估風險等級]
    F --> G{風險等級}
    G -->|低| H[記錄事件]
    G -->|中| I[需要審核]
    G -->|高| J[阻止操作]
    G -->|嚴重| K[凍結賬戶]
    
    H --> E
    I --> L[人工審核]
    L --> M{審核通過?}
    M -->|是| E
    M -->|否| J
    
    style F fill:#fff3e0
    style J fill:#ffcdd2
    style K fill:#ffebee
```

## 數據流向圖

```mermaid
flowchart LR
    subgraph "Client Layer"
        A[Web App]
        B[Mobile App]
        C[API Client]
    end
    
    subgraph "API Layer"
        D[REST API]
        E[WebSocket API]
    end
    
    subgraph "Service Layer"
        F[Auth Service]
        G[Account Service]
        H[Order Service]
        I[Matching Engine]
        J[Risk Service]
        K[Market Data Service]
    end
    
    subgraph "Data Layer"
        L[(PostgreSQL)]
        M[(Redis Cache)]
        N[Kafka Events]
    end
    
    subgraph "External Systems"
        O[Blockchain Network]
        P[Email Service]
        Q[SMS Service]
    end
    
    A --> D
    B --> D
    C --> D
    A --> E
    B --> E
    
    D --> F
    D --> G
    D --> H
    D --> J
    E --> K
    
    F --> L
    G --> L
    H --> L
    I --> M
    J --> M
    K --> M
    
    H --> N
    I --> N
    G --> N
    
    N --> K
    N --> P
    N --> Q
    
    G --> O
    
    style I fill:#ff6b6b
    style N fill:#4ecdc4
    style L fill:#45b7d1
    style M fill:#ffa726
```

這個完整的數據庫設計為 mini-exchange-backend 提供了：

1. **完整的業務域覆蓋**: 從用戶管理到交易執行的全流程支持
2. **高性能架構**: 分區表、索引優化、緩存策略
3. **數據一致性**: 事務支持、約束檢查、審計追蹤
4. **可擴展性**: 模塊化設計、事件驅動架構
5. **安全性**: 權限控制、風控系統、加密存儲

