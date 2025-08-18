# Mini Exchange Database ER Diagram

## Complete Entity Relationship Diagram

```mermaid
erDiagram
    %% Authentication Domain (Auth Domain)
    users {
        bigint id PK
        uuid uuid UK
        varchar username UK
        varchar email UK
        varchar password_hash
        varchar phone
        varchar first_name
        varchar last_name
        varchar status
        integer kyc_level
        timestamp last_login_at
        timestamp created_at
        timestamp updated_at
        integer version
    }
    
    roles {
        serial id PK
        varchar name UK
        text description
        timestamp created_at
    }
    
    permissions {
        serial id PK
        varchar name UK
        varchar resource
        varchar action
        text description
        timestamp created_at
    }
    
    user_roles {
        bigint user_id PK,FK
        integer role_id PK,FK
        timestamp created_at
    }
    
    role_permissions {
        integer role_id PK,FK
        integer permission_id PK,FK
        timestamp created_at
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

    %% Asset Management Domain (Asset Domain)
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

    %% Order Management Domain (Order Domain)
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

    %% Market Data Domain (Market Data Domain)
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

    %% Risk Management Domain (Risk Domain)
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

    %% Clearing & Settlement Domain (Settlement Domain)
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

    %% Notification Domain (Notification Domain)
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

    %% Event Domain (Event Domain)
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

    %% Audit Domain (Audit Domain)
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

    %% Relationship Definitions
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

## Core Business Process Diagrams

### 1. Order Processing Flow

```mermaid
flowchart TD
    A[User Places Order] --> B[Order Validation]
    B --> C{Validation Passed?}
    C -->|No| D[Return Error]
    C -->|Yes| E[Fund Check]
    E --> F{Sufficient Balance?}
    F -->|No| G[Insufficient Balance]
    F -->|Yes| H[Freeze Funds]
    H --> I[Create Order Record]
    I --> J[Send to Matching Engine]
    J --> K[Matching Process]
    K --> L{Matching Order Found?}
    L -->|No| M[Enter Order Book]
    L -->|Yes| N[Execute Trade]
    N --> O[Update Order Status]
    O --> P[Fund Settlement]
    P --> Q[Send Trade Event]
    Q --> R[Notify User]
    
    style A fill:#e1f5fe
    style K fill:#ffecb3
    style N fill:#c8e6c9
    style Q fill:#f3e5f5
```

### 2. Fund Management Flow

```mermaid
flowchart TD
    A[Deposit Request] --> B[Generate Deposit Address]
    B --> C[Monitor Blockchain Transaction]
    C --> D[Confirm Receipt]
    D --> E[Update Account Balance]
    E --> F[Record Transaction Log]
    F --> G[Send Deposit Event]
    
    H[Withdrawal Request] --> I[Fund Check]
    I --> J{Sufficient Balance?}
    J -->|No| K[Withdrawal Failed]
    J -->|Yes| L[Freeze Withdrawal Funds]
    L --> M[Risk Control Check]
    M --> N{Risk Control Passed?}
    N -->|No| O[Unfreeze Funds]
    N -->|Yes| P[Initiate Blockchain Transaction]
    P --> Q[Wait for Confirmation]
    Q --> R[Withdrawal Complete]
    R --> S[Record Transaction Log]
    
    style D fill:#c8e6c9
    style M fill:#ffcdd2
    style R fill:#c8e6c9
```

### 3. Risk Control Processing Flow

```mermaid
flowchart TD
    A[Business Operation] --> B[Trigger Risk Rules]
    B --> C[Rule Matching]
    C --> D{Rule Matched?}
    D -->|No| E[Normal Processing]
    D -->|Yes| F[Assess Risk Level]
    F --> G{Risk Level}
    G -->|Low| H[Record Event]
    G -->|Medium| I[Requires Review]
    G -->|High| J[Block Operation]
    G -->|Critical| K[Freeze Account]
    
    H --> E
    I --> L[Manual Review]
    L --> M{Review Approved?}
    M -->|Yes| E
    M -->|No| J
    
    style F fill:#fff3e0
    style J fill:#ffcdd2
    style K fill:#ffebee
```

## Data Flow Diagram

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

This complete database design provides your mini-exchange-backend with:

1. **Complete Business Domain Coverage**: Full process support from user management to trade execution
2. **High-Performance Architecture**: Partitioned tables, index optimization, caching strategies
3. **Data Consistency**: Transaction support, constraint checks, audit trails
4. **Scalability**: Modular design, event-driven architecture
5. **Security**: Permission control, risk management system, encrypted storage

You can adjust this architecture according to actual business requirements and gradually implement the functionality of each module.
