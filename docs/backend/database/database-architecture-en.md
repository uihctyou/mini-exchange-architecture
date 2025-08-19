# Mini Exchange Database Architecture

## Overview

This document describes the complete database architecture design for Mini Exchange Backend, built on PostgreSQL 15+ with microservices DDD architecture pattern.

## Design Principles

### 1. Business Domain Division (DDD)
- **Authentication Domain**: User management, role permissions, KYC verification
- **Asset Management Domain**: Asset definitions, account balances, transaction records, double-entry bookkeeping
- **Order Management Domain**: Order lifecycle, trading pair configuration
- **Matching Engine Domain**: Trade records, market data
- **Risk Management Domain**: Risk rules, risk events
- **Clearing & Settlement Domain**: Batch clearing, settlement details
- **Notification Domain**: Message templates, notification records
- **Audit Domain**: Operation logs, event sourcing

### 2. Data Integrity Assurance
- **ACID Transactions**: All fund operations support strong consistency
- **Foreign Key Constraints**: Ensure data referential integrity
- **Check Constraints**: Non-negative balance, amount validity verification
- **Unique Constraints**: Prevent duplicate data

### 3. Performance Optimization Strategy
- **Dynamic Partitioned Tables**: Monthly partitioning for large tables (orders, trades, klines, notifications, audit_logs)
- **Partition Management Functions**: Automatic partition creation and management
- **Index Optimization**: Composite indexes optimized for query patterns
- **Updated Triggers**: Automatic `updated_at` field maintenance
- **Data Constraints**: Comprehensive constraints for data integrity

## Core Table Structure

### Authentication Domain (Auth Domain)

#### users - User Basic Information
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

**Key Features**:
- UUID as external reference identifier
- Password hash storage (BCrypt)
- KYC level control (0-2)
- Optimistic locking version control

#### roles & permissions - RBAC Permission System
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

**Permission Model**: Resource-Action pattern, supporting fine-grained permission control

### Asset Management Domain (Account & Asset Domain)

#### assets - Asset Definitions
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

**Asset Types**: Support for cryptocurrencies and fiat currencies with configurable precision

#### accounts - User Asset Accounts
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

**Balance Management**: 
- Available balance + Frozen balance = Total balance
- Non-negative constraints ensure fund safety
- Optimistic locking prevents concurrency issues

#### transactions - Transaction Records (Double-Entry Bookkeeping)
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

**Bookkeeping Principles**: Complete record of every fund movement, supporting audit and reconciliation

### Order Management Domain (Order Domain)

#### trading_pairs - Trading Pair Configuration
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

**Trading Configuration**: Precision control, minimum order amount, fee rate configuration

#### orders - Orders Table (Partitioned Table)
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

**Partitioning Strategy**: Monthly partitioning for improved query performance and maintenance efficiency

#### trades - Trade Records (Partitioned Table)
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

**Trade Records**: Complete buyer and seller information, supporting Maker/Taker fee model

### Market Data Domain (Market Data Domain)

#### klines - K-line Data (Partitioned Table)
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

**K-line Management**: Multi-timeframe, high-performance queries, uniqueness constraints

#### ticker_24hr - 24-hour Statistics
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

**Real-time Statistics**: Fast access to 24-hour trading statistics

### Risk Management Domain (Risk Management Domain)

#### risk_rules - Risk Rules
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

**Rule Engine**: Flexible risk rule system with JSON configuration

#### risk_events - Risk Events
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

**Event Management**: Structured risk event recording and processing workflow

### Outbox Pattern Events Table

#### outbox_events - Event Publishing
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

**Event Sourcing**: Supporting reliable event publishing and microservice communication

## Index Strategy

### Primary Index Design

```sql
-- User query optimization
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_status ON users(status);

-- Account query optimization
CREATE INDEX idx_accounts_user_id ON accounts(user_id);
CREATE INDEX idx_transactions_user_id_created_at ON transactions(user_id, created_at DESC);

-- Order query optimization
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_trading_pair_id ON orders(trading_pair_id);
CREATE INDEX idx_orders_price_side ON orders(trading_pair_id, side, price, created_at);

-- Trade query optimization
CREATE INDEX idx_trades_trading_pair_id ON trades(trading_pair_id);
CREATE INDEX idx_trades_created_at ON trades(created_at DESC);

-- K-line query optimization
CREATE INDEX idx_klines_trading_pair_interval ON klines(trading_pair_id, interval, open_time);
```

### Partitioned Table Indexes

Each partition automatically inherits the main table indexes while supporting partition pruning optimization.

## View Definitions

### Business Views

#### user_balances - User Balance View
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

#### trading_pair_stats - Trading Pair Statistics View
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

## Triggers and Functions

### Auto-update Triggers
```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';
```

### Balance Validation Triggers
```sql
CREATE OR REPLACE FUNCTION check_account_balance()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.available_balance < 0 OR NEW.frozen_balance < 0 THEN
        RAISE EXCEPTION 'Account balance cannot be negative';
    END IF;
    RETURN NEW;
END;
$$ language 'plpgsql';
```

## Performance Optimization Recommendations

### 1. Query Optimization
- Use prepared statements to reduce parsing overhead
- Properly use LIMIT and OFFSET for pagination
- Avoid SELECT * queries
- Use EXPLAIN ANALYZE to analyze query plans

### 2. Partition Management
```sql
-- Function to automatically create future partitions
CREATE OR REPLACE FUNCTION create_monthly_partitions()
RETURNS void AS $$
DECLARE
    start_date date;
    end_date date;
    table_name text;
BEGIN
    -- Create partitions for the next 3 months
    FOR i IN 0..2 LOOP
        start_date := date_trunc('month', CURRENT_DATE + (i || ' months')::interval);
        end_date := start_date + interval '1 month';
        
        -- Create orders partition
        table_name := 'orders_' || to_char(start_date, 'YYYY_MM');
        EXECUTE format('CREATE TABLE IF NOT EXISTS %I PARTITION OF orders 
                       FOR VALUES FROM (%L) TO (%L)', 
                       table_name, start_date, end_date);
        
        -- Create trades partition
        table_name := 'trades_' || to_char(start_date, 'YYYY_MM');
        EXECUTE format('CREATE TABLE IF NOT EXISTS %I PARTITION OF trades 
                       FOR VALUES FROM (%L) TO (%L)', 
                       table_name, start_date, end_date);
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### 3. Connection Pool Configuration
- Set appropriate connection pool size (CPU cores * 2 + 1)
- Configure connection timeout and idle recovery
- Monitor connection pool usage

### 4. Monitoring Metrics
- Slow query monitoring (log_min_duration_statement)
- Connection count monitoring
- Cache hit ratio monitoring
- Disk usage monitoring

## Backup and Recovery Strategy

### 1. Physical Backup
```bash
# Full backup
pg_basebackup -D /backup/full_backup -Ft -z -P

# Incremental backup (WAL archiving)
archive_command = 'cp %p /backup/wal_archive/%f'
```

### 2. Logical Backup
```bash
# Complete database backup
pg_dump -h localhost -U postgres -d mini_exchange > backup.sql

# Schema-only backup
pg_dump -h localhost -U postgres -d mini_exchange --schema-only > schema.sql

# Data-only backup
pg_dump -h localhost -U postgres -d mini_exchange --data-only > data.sql
```

### 3. Disaster Recovery
- Configure streaming replication
- Set up automatic failover
- Regular recovery testing

## Security Considerations

### 1. Data Encryption
- Enable TLS connection encryption
- Application-level encryption for sensitive fields
- Transparent Data Encryption (TDE)

### 2. Access Control
```sql
-- Create application-specific user
CREATE USER mini_exchange_app WITH PASSWORD 'strong_password';

-- Grant necessary permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO mini_exchange_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO mini_exchange_app;

-- Revoke dangerous permissions
REVOKE CREATE ON SCHEMA public FROM mini_exchange_app;
```

### 3. Audit Logs
- Enable PostgreSQL audit extension
- Record all DDL operations
- Monitor abnormal access patterns

## Maintenance Operations

### 1. Regular Maintenance Tasks
```sql
-- Update statistics
ANALYZE;

-- Rebuild indexes
REINDEX DATABASE mini_exchange;

-- Clean old partitions
DROP TABLE IF EXISTS orders_2023_01;
DROP TABLE IF EXISTS trades_2023_01;
```

### 2. Monitoring Scripts
- Check partition creation status
- Monitor table size growth
- Check index usage
- Monitor slow queries

This database architecture design provides a complete data storage solution for Mini Exchange Backend, supporting high-concurrency trading, real-time risk management, reliable fund management, and comprehensive audit tracking.