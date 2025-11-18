# POLARIS EMAIL/MAIL NOTIFICATION MONITORING PACKAGE - DESIGN DOCUMENT

**Date:** November 18, 2025
**Package Name:** `dcplibrary/notices-polaris`
**Purpose:** Laravel package for monitoring and logging email and mail notifications sent by Polaris ILS
**Pattern:** Mirrors `dcplibrary/notices-shoutbomb` structure and conventions

---

## TABLE OF CONTENTS

- [Overview](#overview)
- [Package Goals](#package-goals)
- [Data Sources](#data-sources)
- [Delivery Channel Mapping](#delivery-channel-mapping)
- [Package Structure](#package-structure)
- [Data Export Patterns](#data-export-patterns)
- [Database Schema Design](#database-schema-design)
- [Naming Conventions](#naming-conventions)
- [Integration with Shoutbomb Package](#integration-with-shoutbomb-package)
- [Unified Dashboard Design](#unified-dashboard-design)
- [Next Steps](#next-steps)

---

## OVERVIEW

This package monitors and logs **email** and **mail** (physical mail) notifications sent by Polaris ILS, providing a consistent API with the existing `dcplibrary/notices-shoutbomb` package. Together, these packages will power a **unified notification monitoring dashboard** covering all four delivery channels: Email, Mail, Voice, and Text.

### Key Principles

1. **Consistency:** Mirror Shoutbomb package structure, naming, and patterns
2. **Unified Data Model:** Overlapping fields use identical names across packages
3. **Channel Separation:** Email and Mail tracked separately but follow same patterns
4. **Status Tracking:** Monitor delivery status (sent, failed, bounced) where available
5. **Audit Trail:** Maintain complete history of all notifications sent

---

## PACKAGE GOALS

### Primary Goals
- ✅ Track all email notifications sent by Polaris ILS
- ✅ Track all mail notifications sent by Polaris ILS
- ✅ Monitor delivery status and failures
- ✅ Provide consistent API with Shoutbomb package
- ✅ Enable unified notification dashboard

### Secondary Goals
- ✅ Import notification history from Polaris database
- ✅ Support multiple notification types (holds, overdue, renew, bills, etc.)
- ✅ Provide failure alerting and reporting
- ✅ Maintain patron contact information for audit trail

---

## DATA SOURCES

### Polaris Database Tables

| Table | Schema | Purpose |
|-------|--------|---------|
| **NotificationQueue** | Results.Polaris | Pending notifications to be sent |
| **NotificationHistory** | Results.Polaris | Sent notifications with status tracking |
| **OverdueNotices** | Results.Polaris | Overdue notice details |
| **SysHoldRequests** | Results.Polaris | Hold request details |
| **Patrons** | Polaris.Polaris | Patron contact information |
| **NotificationStatuses** | Polaris.Polaris | Status code lookup (sent, failed, bounced) |

### Key Differences from Shoutbomb

| Aspect | Shoutbomb Package | Email/Mail Package |
|--------|------------------|-------------------|
| **Data Flow** | Export → Shoutbomb → Email Reports | Polaris Sends → Import History |
| **External Service** | Shoutbomb FTP integration | None (Polaris handles sending) |
| **Failure Reporting** | Email reports from Shoutbomb | NotificationHistory status codes |
| **Patron Mapping** | Required (phone to barcode) | Optional (audit trail only) |

---

## DELIVERY CHANNEL MAPPING

### DeliveryOptionID Values

From `Polaris.Polaris.SA_DeliveryOptions`:

| ID | Channel | Handled By | Package |
|----|---------|------------|---------|
| **1** | **Mailing Address** | Polaris ILS | **notices-polaris** |
| **2** | **Email Address** | Polaris ILS | **notices-polaris** |
| **3** | Phone1 (Voice) | Shoutbomb | notices-shoutbomb |
| **8** | TXT Messaging (SMS) | Shoutbomb | notices-shoutbomb |

### SQL Filter for Email/Mail

```sql
WHERE DeliveryOptionID IN (1, 2)  -- Mail and Email
```

---

## PACKAGE STRUCTURE

### Directory Layout

```
dcplibrary/notices-polaris/
├── src/
│   ├── Models/
│   │   ├── EmailPatron.php          # Email patron mapping
│   │   ├── MailPatron.php           # Mail patron mapping
│   │   ├── EmailHold.php            # Email hold notifications
│   │   ├── EmailOverdue.php         # Email overdue notifications
│   │   ├── EmailRenew.php           # Email renewal notifications
│   │   ├── MailHold.php             # Mail hold notifications
│   │   ├── MailOverdue.php          # Mail overdue notifications
│   │   ├── MailRenew.php            # Mail renewal notifications
│   │   ├── NotificationHistory.php  # Import from Polaris history
│   │   └── PolarisSettings.php      # Database config overrides
│   ├── Services/
│   │   ├── EmailImportService.php   # Import email notifications
│   │   ├── MailImportService.php    # Import mail notifications
│   │   └── HistoryImportService.php # Import notification history
│   ├── Commands/
│   │   ├── ImportEmailHolds.php
│   │   ├── ImportEmailOverdue.php
│   │   ├── ImportMailHolds.php
│   │   └── ImportNotificationHistory.php
│   └── NoticesPolarisServiceProvider.php
├── database/
│   ├── migrations/
│   │   ├── create_email_patrons_table.php
│   │   ├── create_mail_patrons_table.php
│   │   ├── create_email_holds_table.php
│   │   ├── create_email_overdue_table.php
│   │   ├── create_mail_holds_table.php
│   │   ├── create_mail_overdue_table.php
│   │   └── create_notification_history_table.php
│   └── seeders/
├── tests/
├── polaris-docs/
│   ├── POLARIS_EMAIL_MAIL_PACKAGE_DESIGN.md (this file)
│   ├── POLARIS_EMAIL_NOTIFICATIONS.md
│   ├── POLARIS_MAIL_NOTIFICATIONS.md
│   └── SQL_QUERIES.md
├── composer.json
└── README.md
```

---

## DATA EXPORT PATTERNS

### Pattern 1: Notification History Import (Primary)

**Recommended approach** - Import directly from `Results.Polaris.NotificationHistory`

**Why:**
- Polaris already tracks sent notifications with status
- No need for separate export files
- Real-time status updates available
- Consistent with how Polaris works (unlike Shoutbomb's external service)

**Query Pattern:**
```sql
SELECT
    nh.PatronId,
    nh.ItemRecordId,
    nh.TxnId,
    nh.NotificationTypeId,
    nh.ReportingOrgId,
    nh.DeliveryOptionId,
    nh.NoticeDate,
    nh.Amount,
    nh.NotificationStatusId,
    nh.Title,
    -- Join to Patrons for contact info
    p.Barcode AS PatronBarcode,
    p.NameFirst,
    p.NameLast,
    p.EmailAddress,
    p.StreetOne,
    p.City,
    p.State,
    p.PostalCode
FROM [Results].[Polaris].[NotificationHistory] nh
JOIN [Polaris].[Polaris].[Patrons] p ON nh.PatronId = p.PatronId
WHERE nh.NoticeDate >= DATEADD(day, -7, GETDATE())
  AND nh.DeliveryOptionId IN (1, 2)  -- Mail and Email
ORDER BY nh.NoticeDate DESC
```

### Pattern 2: Separate Export Files (Alternative)

If you prefer to match Shoutbomb's file-based pattern:

**Email Notifications:**
- `email_holds_submitted_YYYY-MM-DD_HH-MM-SS.txt`
- `email_overdue_submitted_YYYY-MM-DD_HH-MM-SS.txt`
- `email_renew_submitted_YYYY-MM-DD_HH-MM-SS.txt`

**Mail Notifications:**
- `mail_holds_submitted_YYYY-MM-DD_HH-MM-SS.txt`
- `mail_overdue_submitted_YYYY-MM-DD_HH-MM-SS.txt`
- `mail_renew_submitted_YYYY-MM-DD_HH-MM-SS.txt`

**Delimiter:** Pipe (`|`) to match Shoutbomb pattern

---

## DATABASE SCHEMA DESIGN

### Core Tables (Mirroring Shoutbomb)

#### 1. `polaris_email_patrons`

Maps PatronID to email address (like `shoutbomb_voice_patrons`)

| Column | Type | Description |
|--------|------|-------------|
| id | bigint | Primary key |
| patron_id | int | Polaris PatronID |
| patron_barcode | varchar(20) | Patron barcode |
| email_address | varchar(255) | Patron email |
| name_first | varchar(50) | First name |
| name_last | varchar(50) | Last name |
| import_date | datetime | When imported |
| is_active | boolean | Active email? |
| created_at | timestamp | |
| updated_at | timestamp | |

**Indexes:**
- `patron_id` (unique)
- `patron_barcode`
- `email_address`

#### 2. `polaris_mail_patrons`

Maps PatronID to mailing address

| Column | Type | Description |
|--------|------|-------------|
| id | bigint | Primary key |
| patron_id | int | Polaris PatronID |
| patron_barcode | varchar(20) | Patron barcode |
| street_one | varchar(255) | Street address |
| street_two | varchar(255) | Apt/Suite |
| city | varchar(100) | City |
| state | varchar(2) | State code |
| postal_code | varchar(10) | ZIP code |
| country | varchar(50) | Country |
| name_first | varchar(50) | First name |
| name_last | varchar(50) | Last name |
| import_date | datetime | When imported |
| is_active | boolean | Active address? |
| created_at | timestamp | |
| updated_at | timestamp | |

**Indexes:**
- `patron_id` (unique)
- `patron_barcode`
- `postal_code`

#### 3. `polaris_email_holds`

Email hold notifications (like `shoutbomb_holds`)

| Column | Type | Description |
|--------|------|-------------|
| id | bigint | Primary key |
| sys_hold_request_id | int | Hold request ID |
| patron_id | int | Polaris PatronID |
| patron_barcode | varchar(20) | Patron barcode |
| patron_email | varchar(255) | Email sent to |
| item_record_id | int | Item record ID |
| item_barcode | varchar(20) | Item barcode |
| browse_title | varchar(255) | Title |
| creation_date | date | Hold created date |
| hold_till_date | date | Pickup expiration |
| notification_date | datetime | When notification sent |
| notification_status_id | int | Status (sent/failed) |
| pickup_branch_id | int | Pickup location |
| import_date | datetime | When imported to Laravel |
| created_at | timestamp | |
| updated_at | timestamp | |

**Indexes:**
- `sys_hold_request_id` (unique)
- `patron_id`
- `patron_barcode`
- `notification_date`
- `notification_status_id`

#### 4. `polaris_email_overdue`

Email overdue notifications (like `shoutbomb_overdue`)

| Column | Type | Description |
|--------|------|-------------|
| id | bigint | Primary key |
| overdue_notice_id | int | Polaris OverdueNoticeID |
| patron_id | int | Polaris PatronID |
| patron_barcode | varchar(20) | Patron barcode |
| patron_email | varchar(255) | Email sent to |
| item_record_id | int | Item record ID |
| item_barcode | varchar(20) | Item barcode |
| browse_title | varchar(255) | Title |
| browse_author | varchar(255) | Author |
| due_date | date | Original due date |
| notification_date | datetime | When notification sent |
| notification_type_id | int | 1=1st, 12=2nd, 13=3rd |
| notification_status_id | int | Status (sent/failed) |
| loaning_org_id | int | Owning branch |
| reporting_org_id | int | Sending branch |
| import_date | datetime | When imported to Laravel |
| created_at | timestamp | |
| updated_at | timestamp | |

**Indexes:**
- `overdue_notice_id` (unique)
- `patron_id`
- `patron_barcode`
- `due_date`
- `notification_date`
- `notification_type_id`

#### 5. `polaris_notification_history`

Complete notification history import

| Column | Type | Description |
|--------|------|-------------|
| id | bigint | Primary key |
| patron_id | int | Polaris PatronID |
| patron_barcode | varchar(20) | Patron barcode |
| item_record_id | int | Item record ID (nullable) |
| txn_id | int | Transaction ID (nullable) |
| notification_type_id | int | Type of notification |
| reporting_org_id | int | Sending branch |
| delivery_option_id | int | 1=Mail, 2=Email |
| notice_date | datetime | When sent |
| amount | decimal(10,2) | Amount (for bills) |
| notification_status_id | int | Delivery status |
| title | varchar(255) | Item title |
| import_date | datetime | When imported |
| created_at | timestamp | |
| updated_at | timestamp | |

**Indexes:**
- `patron_id`
- `delivery_option_id`
- `notification_type_id`
- `notification_status_id`
- `notice_date`

---

## NAMING CONVENTIONS

### Consistency with Shoutbomb Package

| Concept | Shoutbomb Package | Email/Mail Package | Notes |
|---------|------------------|-------------------|-------|
| **Table Prefix** | `shoutbomb_` | `polaris_` | Different service |
| **Field: Patron ID** | `patron_id` | `patron_id` | ✅ Identical |
| **Field: Barcode** | `patron_barcode` | `patron_barcode` | ✅ Identical |
| **Field: Item Barcode** | `item_barcode` | `item_barcode` | ✅ Identical |
| **Field: Title** | `browse_title` | `browse_title` | ✅ Identical |
| **Field: Import Date** | `import_date` | `import_date` | ✅ Identical |
| **Model Names** | `VoicePatron`, `HoldsExport` | `EmailPatron`, `EmailHold` | Channel prefix |
| **File Pattern** | `{type}_submitted_{timestamp}.txt` | `{channel}_{type}_submitted_{timestamp}.txt` | Add channel |

### Overlapping Fields (Must Match!)

These fields appear in both packages and **MUST** use identical names:

- `patron_id` (int)
- `patron_barcode` (varchar(20))
- `item_record_id` (int, nullable)
- `item_barcode` (varchar(20))
- `browse_title` (varchar(255))
- `browse_author` (varchar(255))
- `notification_type_id` (int)
- `reporting_org_id` (int)
- `import_date` (datetime)
- `created_at` (timestamp)
- `updated_at` (timestamp)

### Channel-Specific Fields

**Email-specific:**
- `patron_email` (varchar(255))

**Mail-specific:**
- `street_one`, `street_two`, `city`, `state`, `postal_code`

**Voice/Text-specific (Shoutbomb):**
- `phone_number` (varchar(10))

---

## INTEGRATION WITH SHOUTBOMB PACKAGE

### Shared Configuration Pattern

Both packages use database override system via settings table:

**Shoutbomb:**
```php
DB::table('shoutbomb_settings')->insert([
    'key' => 'ftp.host',
    'value' => 'ftp.example.com',
]);
```

**Polaris:**
```php
DB::table('polaris_settings')->insert([
    'key' => 'database.host',
    'value' => 'polaris-sql.example.com',
]);
```

### Shared Service Providers

Both packages should register with consistent patterns:

```php
// Shoutbomb
$this->app->register(NoticesShoutbombServiceProvider::class);

// Polaris
$this->app->register(NoticesPolarisServiceProvider::class);
```

---

## UNIFIED DASHBOARD DESIGN

### Dashboard Data Model

The unified dashboard will aggregate data from both packages:

#### Notification Summary View

```sql
-- Email notifications (this package)
SELECT
    'Email' as channel,
    notification_type_id,
    notification_status_id,
    COUNT(*) as count,
    DATE(notice_date) as date
FROM polaris_notification_history
WHERE delivery_option_id = 2
GROUP BY DATE(notice_date), notification_type_id, notification_status_id

UNION ALL

-- Mail notifications (this package)
SELECT
    'Mail' as channel,
    notification_type_id,
    notification_status_id,
    COUNT(*) as count,
    DATE(notice_date) as date
FROM polaris_notification_history
WHERE delivery_option_id = 1
GROUP BY DATE(notice_date), notification_type_id, notification_status_id

UNION ALL

-- Voice notifications (shoutbomb package)
SELECT
    'Voice' as channel,
    -- ... from shoutbomb tables

UNION ALL

-- Text notifications (shoutbomb package)
SELECT
    'Text' as channel,
    -- ... from shoutbomb tables
```

#### Dashboard Metrics

1. **Total Notifications by Channel** (Email, Mail, Voice, Text)
2. **Success/Failure Rates** by Channel
3. **Notification Types** (Holds, Overdue, Renew, Bills)
4. **Daily Volume Trends**
5. **Failure Alerts** (Email bounces, invalid phones, bad addresses)
6. **Patron Contact Method Distribution**

---

## NEXT STEPS

### Phase 1: Data Mapping (CURRENT)
- ✅ Study Shoutbomb package structure
- ✅ Analyze Polaris database schemas
- ⏳ **Map Patrons table fields** (email, address, name)
- ⏳ **Get NotificationStatuses lookup** (status meanings)
- ⏳ Design SQL export queries

### Phase 2: Package Implementation
- Create Laravel package structure
- Write migrations for all tables
- Create models with relationships
- Implement import services
- Write Artisan commands

### Phase 3: Testing
- Create test data seeders
- Write unit tests
- Write integration tests
- Test import from sample exports

### Phase 4: Documentation
- SQL query documentation
- API documentation
- Installation guide
- Usage examples

### Phase 5: Dashboard Integration
- Unified data model
- Dashboard views
- Reporting tools
- Alerting system

---

## REQUIRED INFORMATION

To complete Phase 1, I still need:

### 1. Patrons Table Field Names

Please provide the actual field names in `Polaris.Polaris.Patrons`:

```sql
-- What are the field names for:
-- Email address: EmailAddress? Email1? EmailAddr?
-- First name: NameFirst? FirstName? PatronFirstName?
-- Last name: NameLast? LastName? PatronLastName?
-- Barcode: Barcode? PatronBarcode?
-- Street: StreetOne? Address1? MailingAddress1?
-- City: City?
-- State: State? StateAbbr?
-- Zip: PostalCode? Zip? ZipCode?
```

### 2. NotificationStatuses Lookup

```sql
SELECT NotificationStatusID, StatusLabel, Description
FROM Polaris.Polaris.NotificationStatuses
ORDER BY NotificationStatusID;
```

This will tell me:
- Which status = Successfully sent
- Which status = Failed
- Which status = Bounced
- Which status = Pending

### 3. Export Strategy Decision

**Option A:** Import from NotificationHistory table (recommended)
- Real-time import via scheduled command
- No export files needed
- Direct database connection

**Option B:** Export to files (matching Shoutbomb pattern)
- Generate daily export files
- Archive submitted files
- Import from files

**Which do you prefer?**

---

**Status:** Design phase - awaiting Patrons field names and NotificationStatuses lookup
**Next Action:** Provide requested information to proceed with SQL query design
**Estimated Completion:** Phase 1 = 1 day after info provided, Full package = 1 week

