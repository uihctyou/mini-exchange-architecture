# Schema Translation Reference

## Overview

This document provides a comprehensive translation reference for all Chinese comments and content in the Mini Exchange Backend database schema files. This ensures consistency and internationalization standards across the codebase.

## Schema Comments Translation

### 1. Main Section Headers

| Chinese | English |
|---------|---------|
| 設計原則: 微服務架構 + DDD + 事件驅動 | Design Principles: Microservices Architecture + DDD + Event-Driven |
| 支持: PostgreSQL 15+, 分區表, 索引優化 | Support: PostgreSQL 15+, Partitioned Tables, Index Optimization |
| 創建序列和函數 | Create sequences and functions |

### 2. Domain Names

| Chinese | English |
|---------|---------|
| 用戶認證域 (Auth Domain) | Authentication Domain (Auth Domain) |
| 資產管理域 (Account & Asset Domain) | Asset Management Domain (Account & Asset Domain) |
| 訂單管理域 (Order Domain) | Order Management Domain (Order Domain) |
| 市場數據域 (Market Data Domain) | Market Data Domain (Market Data Domain) |
| 風控域 (Risk Management Domain) | Risk Management Domain (Risk Management Domain) |
| 清算結算域 (Clearing & Settlement Domain) | Clearing & Settlement Domain (Clearing & Settlement Domain) |
| 通知域 (Notification Domain) | Notification Domain (Notification Domain) |
| Outbox Pattern 事件表 | Outbox Pattern Events Table |
| 審計日誌表 | Audit Logs Table |

### 3. Table Names

| Chinese | English |
|---------|---------|
| 用戶表 | Users table |
| 角色表 | Roles table |
| 權限表 | Permissions table |
| 用戶角色關聯表 | User roles association table |
| 角色權限關聯表 | Role permissions association table |
| KYC 認證記錄表 | KYC verification records table |
| 資產類型表 | Asset types table |
| 用戶賬戶表 | User accounts table |
| 交易記錄表 (複式記帳) | Transaction records table (Double-entry bookkeeping) |
| 資金凍結記錄表 | Balance freeze records table |
| 充值提現記錄表 | Deposit and withdrawal records table |
| 交易對表 | Trading pairs table |
| 訂單表 | Orders table |
| 成交記錄表 | Trades table |
| 行情數據表 (K線) | Market data table (K-line) |
| 24小時統計表 | 24-hour statistics table |
| 風控規則表 | Risk rules table |
| 風控事件記錄表 | Risk events table |
| 清算批次表 | Settlement batches table |
| 清算明細表 | Settlement details table |
| 通知模板表 | Notification templates table |
| 通知記錄表 | Notifications table |
| 事件發佈表 (Outbox Pattern) | Event publishing table (Outbox Pattern) |
| 操作審計表 | Operation audit table |

### 4. Index and Partition Comments

| Chinese | English |
|---------|---------|
| 訂單表分區 (按月分區) | Orders table partitions (partition by month) |
| 成交記錄表分區 (按月分區) | Trades table partitions (partition by month) |
| K線表分區 (按月分區) | K-line table partitions (partition by month) |
| 通知表分區 | Notifications table partitions |
| 審計日誌分區 | Audit logs partitions |
| 用戶相關索引 | User related indexes |
| 賬戶相關索引 | Account related indexes |
| 訂單相關索引 | Order related indexes |
| 成交相關索引 | Trade related indexes |
| K線相關索引 | K-line related indexes |
| 事件相關索引 | Event related indexes |

### 5. Data and Function Comments

| Chinese | English |
|---------|---------|
| 初始數據 | Initial Data |
| 插入基礎角色 | Insert basic roles |
| 插入基礎權限 | Insert basic permissions |
| 分配角色權限 | Assign role permissions |
| 插入基礎資產 | Insert basic assets |
| 插入基礎交易對 | Insert basic trading pairs |
| 插入基礎風控規則 | Insert basic risk rules |
| 插入通知模板 | Insert notification templates |
| 函數和觸發器 | Functions and Triggers |
| 更新 updated_at 欄位的函數 | Function to update updated_at column |
| 為相關表創建更新觸發器 | Create update triggers for related tables |
| 賬戶餘額檢查函數 | Account balance check function |
| 賬戶餘額檢查觸發器 | Account balance check trigger |
| 視圖定義 | View Definitions |
| 用戶賬戶餘額視圖 | User account balance view |
| 交易對統計視圖 | Trading pair statistics view |
| 用戶交易統計視圖 | User trading statistics view |
| 索引創建 | Index Creation |
| Schema 創建完成 | Schema Creation Complete |

### 6. Business Logic Comments

| Chinese | English |
|---------|---------|
| 0: 未認證, 1: 基礎認證, 2: 高級認證 | 0: Not verified, 1: Basic verification, 2: Advanced verification |
| 按月分區 | partition by month |
| 賬戶餘額不能為負數 | Account balance cannot be negative |

### 7. Role and Permission Data

| Chinese | English |
|---------|---------|
| 普通用戶 | Regular User |
| VIP用戶 | VIP User |
| 管理員 | Administrator |
| 超級管理員 | Super Administrator |
| 查看用戶信息 | View user information |
| 修改用戶信息 | Modify user information |
| 查看訂單 | View orders |
| 創建/修改訂單 | Create/modify orders |
| 查看賬戶 | View accounts |
| 修改賬戶 | Modify accounts |
| 管理員查看權限 | Admin view permissions |
| 管理員操作權限 | Admin operation permissions |

### 8. Risk Management Data

| Chinese | English |
|---------|---------|
| 日交易額度限制 | Daily Trading Volume Limit |
| 單筆訂單限制 | Single Order Limit |

### 9. Notification Templates

| Chinese | English |
|---------|---------|
| 訂單成交通知 | Order Filled Notification |
| 您的訂單 {{order_id}} 已成交，成交價格: {{price}}，成交數量: {{amount}} | Your order {{order_id}} has been filled at price: {{price}}, quantity: {{amount}} |
| 充值確認通知 | Deposit Confirmation |
| 您的 {{asset}} 充值已確認，金額: {{amount}} | Your {{asset}} deposit has been confirmed, amount: {{amount}} |

## Data.sql Translation Reference

### Test User Data

| Chinese | English |
|---------|---------|
| 管理員用戶 | Admin user |
| 測試用戶1 | Test user 1 |
| 測試用戶2 | Test user 2 |
| VIP用戶 | VIP user |
| 分配管理員角色 | Assign admin role |
| 分配普通用戶角色 | Assign regular user role |
| 分配VIP用戶角色 | Assign VIP user role |
| 創建測試用戶 | Create test users |
| 創建用戶賬戶並充值測試資金 | Create user accounts and deposit test funds |
| 為所有用戶創建所有資產的賬戶 | Create accounts for all assets for all users |
| 創建交易記錄 (模擬初始資金入賬) | Create transaction records (simulate initial fund deposits) |
| 為測試用戶創建初始充值記錄 | Create initial deposit records for test users |
| 初始測試資金 | Initial test funds |
| 創建測試訂單數據 | Create test order data |
| 獲取交易對ID | Get trading pair IDs |
| 插入一些測試訂單 | Insert some test orders |
| 創建市場數據 (K線數據) | Create market data (K-line data) |
| 插入BTCUSDT的日K線數據 (最近7天) | Insert BTCUSDT daily K-line data (last 7 days) |
| 初始化24小時統計數據 | Initialize 24-hour statistics data |
| 創建風控事件示例 | Create risk event examples |
| 插入一個低風險事件 | Insert a low-risk event |
| 創建通知記錄示例 | Create notification record examples |
| 為用戶創建歡迎通知 | Create welcome notifications for users |
| 歡迎使用 Mini Exchange | Welcome to Mini Exchange |
| 歡迎 {{name}} 加入 Mini Exchange！您的賬戶已成功創建。 | Welcome {{name}} to Mini Exchange! Your account has been successfully created. |
| 創建審計日誌示例 | Create audit log examples |
| 記錄用戶註冊事件 | Record user registration events |
| 記錄賬戶創建事件 | Record account creation events |
| 創建 Outbox 事件示例 | Create Outbox event examples |
| 創建用戶註冊事件 | Create user registration events |
| 創建賬戶創建事件 | Create account creation events |
| 創建清算批次示例 | Create settlement batch examples |
| 創建昨天的清算批次 | Create yesterday's settlement batch |
| 為清算批次創建明細 | Create details for settlement batch |
| 更新 net_amount | Update net_amount |
| 初始數據插入完成 | Initial data insertion complete |
| 顯示統計信息 | Display statistics |

### Comments in SQL Code

| Chinese | English |
|---------|---------|
| 清理現有數據 (僅用於開發環境) | Clean existing data (for development environment only) |
| 密碼: admin123 | Password: admin123 |
| 密碼: user123 | Password: user123 |
| 密碼: vip123 | Password: vip123 |

## Usage Guidelines

1. **Consistency**: Always use the English translations provided in this reference when updating or creating new database content.

2. **Context Awareness**: Consider the business context when translating. For example, "風控" specifically refers to "Risk Management" in financial contexts.

3. **Template Variables**: When translating notification templates, preserve the template variable syntax `{{variable_name}}`.

4. **Technical Terms**: Maintain technical accuracy. For example, "複式記帳" is specifically "Double-entry bookkeeping" in accounting.

5. **Database Comments**: Use the exact English translations for SQL comments to maintain consistency across the codebase.

## Maintenance

This translation reference should be updated whenever:
- New Chinese content is added to the schema
- Business terminology changes
- New domains or modules are added to the system

Last Updated: August 18, 2025
