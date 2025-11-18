# EMAIL/MAIL NOTIFICATION MONITORING PACKAGE - DESIGN SUMMARY

**Date:** November 18, 2025
**Package:** `dcplibrary/notices-polaris`
**Status:** Design Complete - Ready for Implementation

---

## EXECUTIVE SUMMARY

This document summarizes the complete design for the Polaris Email/Mail notification monitoring package, which will mirror the structure and conventions of the existing `dcplibrary/notices-shoutbomb` package to enable a **unified notification monitoring dashboard** covering all four delivery channels: Email, Mail, Voice, and Text.

---

## PACKAGE OVERVIEW

### Purpose
Monitor and log email and mail notifications sent by Polaris ILS with consistent API and data structures that align with the Shoutbomb package.

### Key Design Decisions

1. **Primary Data Source:** `Polaris.Polaris.NotificationLog` table
   - Simpler than joining multiple tables
   - Already contains PatronBarcode and email addresses
   - Aggregates multiple items per notification
   - Includes delivery status tracking

2. **Package Name:** `dcplibrary/notices-polaris`
   - Parallels `dcplibrary/notices-shoutbomb`
   - Handles both email (DeliveryOptionID=2) and mail (DeliveryOptionID=1)

3. **Data Model:** Single unified package for both channels
   - Consistent field names with Shoutbomb package
   - Channel-specific tables (email_*, mail_*)
   - Shared notification_log base table

---

## DATA STRUCTURE ANALYSIS

### Delivery Channels (DeliveryOptionID)

| ID | Channel | Polaris Handles? | Package | DeliveryString Contains |
|----|---------|------------------|---------|------------------------|
| **1** | **Mail** | Yes | **polaris** | (blank) |
| **2** | **Email** | Yes | **polaris** | Email address |
| 3 | Voice | No (Shoutbomb) | shoutbomb | Phone number |
| 8 | Text | No (Shoutbomb) | shoutbomb | Phone number |

### Notification Status Codes

**Email Statuses:**
- ✅ **12** = Email Completed (success)
- ❌ **13** = Email Failed - Invalid address
- ❌ **14** = Email Failed (general)

**Mail Statuses:**
- ✅ **15** = Mail Printed (success)

**Success Rate Calculation:**
```sql
Success Rate = (Status 12 count / Total email notifications) * 100
Failure Rate = ((Status 13 + 14 count) / Total) * 100
```

### Common Notification Types

| ID | Type | Email Common? | Mail Common? | Notes |
|----|------|--------------|--------------|-------|
| 1 | 1st Overdue | ✅ Very Common | ✅ Common | First overdue notice |
| 2 | Hold | ✅ Very Common | ⚠️ Rare | Hold ready pickup |
| 7 | Almost Overdue | ✅ Common | ⚠️ Rare | Pre-overdue reminder |
| 8 | Fine | ✅ Common | ✅ Common | Fine notice |
| 11 | Bill | ✅ Common | ✅ Very Common | Billing notice |
| 12 | 2nd Overdue | ✅ Common | ✅ Common | Second overdue |
| 13 | 3rd Overdue | ✅ Common | ✅ Very Common | Third overdue (final) |
| 18 | 2nd Hold | ✅ Common | ❌ Rare | Second hold notice |
| 20 | Manual Bill | ✅ Occasional | ✅ Occasional | Manual billing |

---

## DATABASE SCHEMA DESIGN

### Core Tables

#### 1. `polaris_notification_log`
**Purpose:** Main import table from Polaris NotificationLog

```sql
CREATE TABLE polaris_notification_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    notification_log_id INT NOT NULL UNIQUE,
    patron_id INT NOT NULL,
    patron_barcode VARCHAR(20),
    notification_date_time DATETIME NOT NULL,
    notification_type_id INT NOT NULL,
    delivery_option_id INT NOT NULL,  -- 1=Mail, 2=Email
    email_address VARCHAR(255),        -- For email notifications
    notification_status_id INT NOT NULL,
    overdues_count INT DEFAULT 0,
    holds_count INT DEFAULT 0,
    cancels_count INT DEFAULT 0,
    bills_count INT DEFAULT 0,
    manual_bill_count INT DEFAULT 0,
    overdues_2nd_count INT DEFAULT 0,
    overdues_3rd_count INT DEFAULT 0,
    reporting_org_id INT,
    language_id INT,
    details TEXT,
    import_date DATETIME NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_patron_id (patron_id),
    INDEX idx_patron_barcode (patron_barcode),
    INDEX idx_delivery_option (delivery_option_id),
    INDEX idx_notification_date (notification_date_time),
    INDEX idx_notification_type (notification_type_id),
    INDEX idx_notification_status (notification_status_id),
    INDEX idx_email (email_address)
);
```

#### 2. `polaris_email_patrons`
**Purpose:** Active email patron mapping (like shoutbomb_voice_patrons)

```sql
CREATE TABLE polaris_email_patrons (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    patron_id INT NOT NULL UNIQUE,
    patron_barcode VARCHAR(20),
    email_address VARCHAR(255) NOT NULL,
    name_first VARCHAR(50),
    name_last VARCHAR(50),
    email_notices BOOLEAN DEFAULT 1,
    is_active BOOLEAN DEFAULT 1,
    import_date DATETIME NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_patron_barcode (patron_barcode),
    INDEX idx_email (email_address),
    INDEX idx_active (is_active)
);
```

#### 3. `polaris_mail_patrons`
**Purpose:** Active mail patron mapping

```sql
CREATE TABLE polaris_mail_patrons (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    patron_id INT NOT NULL UNIQUE,
    patron_barcode VARCHAR(20),
    name_first VARCHAR(50),
    name_last VARCHAR(50),
    street_one VARCHAR(255),
    street_two VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(2),
    postal_code VARCHAR(10),
    country VARCHAR(50),
    mail_notices BOOLEAN DEFAULT 1,
    is_active BOOLEAN DEFAULT 1,
    import_date DATETIME NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_patron_barcode (patron_barcode),
    INDEX idx_postal_code (postal_code),
    INDEX idx_active (is_active)
);
```

#### 4. `polaris_settings`
**Purpose:** Database configuration overrides (matches shoutbomb_settings pattern)

```sql
CREATE TABLE polaris_settings (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    key VARCHAR(255) NOT NULL UNIQUE,
    value TEXT,
    type VARCHAR(50) DEFAULT 'string',
    is_active BOOLEAN DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_key (key),
    INDEX idx_active (is_active)
);
```

### Reference/Lookup Tables

These mirror Polaris lookup tables for offline queries:

#### 5. `polaris_notification_statuses`
```sql
CREATE TABLE polaris_notification_statuses (
    notification_status_id INT PRIMARY KEY,
    description VARCHAR(255) NOT NULL,
    is_success BOOLEAN DEFAULT 0,
    channel VARCHAR(20),  -- 'Email', 'Mail', 'Voice', 'Text'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

**Seed Data:**
```sql
INSERT INTO polaris_notification_statuses VALUES
(12, 'Email Completed', 1, 'Email'),
(13, 'Email Failed - Invalid address', 0, 'Email'),
(14, 'Email Failed', 0, 'Email'),
(15, 'Mail Printed', 1, 'Mail');
```

#### 6. `polaris_notification_types`
```sql
CREATE TABLE polaris_notification_types (
    notification_type_id INT PRIMARY KEY,
    description VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

**Seed Data:**
```sql
INSERT INTO polaris_notification_types VALUES
(1, '1st Overdue'),
(2, 'Hold'),
(7, 'Almost overdue/Auto-renew reminder'),
(8, 'Fine'),
(11, 'Bill'),
(12, '2nd Overdue'),
(13, '3rd Overdue'),
(18, '2nd Hold'),
(20, 'Manual Bill'),
(21, '2nd Fine Notice');
```

---

## CONSISTENT FIELD NAMING

### Fields That MUST Match Shoutbomb Package

| Field Name | Type | Notes |
|------------|------|-------|
| patron_id | INT | Polaris PatronID |
| patron_barcode | VARCHAR(20) | Patron barcode |
| notification_type_id | INT | Type of notification |
| notification_status_id | INT | Delivery status |
| reporting_org_id | INT | Sending branch |
| import_date | DATETIME | When imported to Laravel |
| created_at | TIMESTAMP | Record creation |
| updated_at | TIMESTAMP | Record modification |

### Channel-Specific Fields

**Email Package:**
- `email_address` (VARCHAR(255))

**Mail Package:**
- `street_one`, `street_two`, `city`, `state`, `postal_code` (VARCHAR)

**Shoutbomb Package:**
- `phone_number` (VARCHAR(10))

---

## SQL EXPORT QUERIES

### Daily Email Notifications
```sql
SELECT * FROM Polaris.Polaris.NotificationLog
WHERE DeliveryOptionID = 2
  AND NotificationDateTime >= DATEADD(day, -7, GETDATE())
```

### Daily Mail Notifications
```sql
SELECT nl.*, p.StreetOne, p.City, p.State, p.PostalCode
FROM Polaris.Polaris.NotificationLog nl
JOIN Polaris.Polaris.Patrons p ON nl.PatronID = p.PatronID
WHERE nl.DeliveryOptionID = 1
  AND nl.NotificationDateTime >= DATEADD(day, -7, GETDATE())
```

### Email Failures (Monitoring)
```sql
SELECT * FROM Polaris.Polaris.NotificationLog
WHERE DeliveryOptionID = 2
  AND NotificationStatusID IN (13, 14)  -- Failed statuses
  AND NotificationDateTime >= DATEADD(day, -30, GETDATE())
```

See `POLARIS_SQL_QUERIES.md` for complete query documentation.

---

## LARAVEL PACKAGE STRUCTURE

```
dcplibrary/notices-polaris/
├── src/
│   ├── Models/
│   │   ├── NotificationLog.php
│   │   ├── EmailPatron.php
│   │   ├── MailPatron.php
│   │   ├── NotificationStatus.php
│   │   ├── NotificationType.php
│   │   └── PolarisSettings.php
│   ├── Services/
│   │   ├── NotificationImportService.php
│   │   ├── EmailPatronService.php
│   │   └── MailPatronService.php
│   ├── Commands/
│   │   ├── ImportNotifications.php
│   │   ├── ImportEmailPatrons.php
│   │   ├── ImportMailPatrons.php
│   │   └── SyncReferenceTables.php
│   ├── Http/Controllers/
│   │   └── NotificationController.php
│   └── NoticesPolarisServiceProvider.php
├── database/
│   ├── migrations/
│   │   ├── 2025_11_18_000001_create_polaris_notification_log_table.php
│   │   ├── 2025_11_18_000002_create_polaris_email_patrons_table.php
│   │   ├── 2025_11_18_000003_create_polaris_mail_patrons_table.php
│   │   ├── 2025_11_18_000004_create_polaris_settings_table.php
│   │   ├── 2025_11_18_000005_create_polaris_notification_statuses_table.php
│   │   └── 2025_11_18_000006_create_polaris_notification_types_table.php
│   └── seeders/
│       ├── NotificationStatusesSeeder.php
│       └── NotificationTypesSeeder.php
├── polaris-docs/
│   ├── PACKAGE_DESIGN_SUMMARY.md (this file)
│   ├── POLARIS_EMAIL_MAIL_PACKAGE_DESIGN.md
│   └── POLARIS_SQL_QUERIES.md
├── composer.json
└── README.md
```

---

## ARTISAN COMMANDS

### Import Commands

```bash
# Import email/mail notifications from last 7 days
php artisan polaris:import-notifications --days=7

# Import only email notifications
php artisan polaris:import-notifications --channel=email --days=7

# Import only mail notifications
php artisan polaris:import-notifications --channel=mail --days=7

# Import patron contact lists
php artisan polaris:import-email-patrons
php artisan polaris:import-mail-patrons

# Sync reference tables (statuses, types)
php artisan polaris:sync-references
```

### Reporting Commands

```bash
# Email failure report
php artisan polaris:email-failures --days=30

# Notification statistics
php artisan polaris:stats --channel=email --days=30
php artisan polaris:stats --channel=mail --days=30
```

---

## UNIFIED DASHBOARD INTEGRATION

### Dashboard Data Aggregation

```sql
-- Combined notification summary (all 4 channels)
SELECT
    'Email' AS channel,
    notification_type_id,
    notification_status_id,
    COUNT(*) as total,
    SUM(holds_count) as holds,
    SUM(overdues_count) as overdues,
    DATE(notification_date_time) as date
FROM polaris_notification_log
WHERE delivery_option_id = 2
GROUP BY DATE(notification_date_time), notification_type_id, notification_status_id

UNION ALL

SELECT
    'Mail' AS channel,
    notification_type_id,
    notification_status_id,
    COUNT(*) as total,
    SUM(holds_count) as holds,
    SUM(overdues_count) as overdues,
    DATE(notification_date_time) as date
FROM polaris_notification_log
WHERE delivery_option_id = 1
GROUP BY DATE(notification_date_time), notification_type_id, notification_status_id

-- Add Voice and Text from shoutbomb package tables...
```

### Dashboard Metrics

1. **Total Notifications by Channel** (Email, Mail, Voice, Text)
2. **Success/Failure Rates** by Channel
3. **Notification Type Distribution**
4. **Daily Volume Trends**
5. **Patron Delivery Preferences**
6. **Failure Alerts** (bounced emails, invalid phones, etc.)

---

## IMPLEMENTATION PHASES

### ✅ Phase 1: Design & Analysis (COMPLETE)
- [x] Study Shoutbomb package structure
- [x] Analyze Polaris database schemas
- [x] Identify NotificationLog as primary source
- [x] Map notification statuses and types
- [x] Design consistent field naming
- [x] Create SQL export queries

### ⏳ Phase 2: Package Setup (NEXT)
- [ ] Verify Patrons table field names
- [ ] Create package directory structure
- [ ] Set up composer.json
- [ ] Create service provider
- [ ] Configure database connections

### ⏳ Phase 3: Database Layer
- [ ] Write migration files
- [ ] Create model classes
- [ ] Add relationships
- [ ] Create seeder files
- [ ] Test migrations

### ⏳ Phase 4: Import Services
- [ ] NotificationImportService
- [ ] EmailPatronService
- [ ] MailPatronService
- [ ] Database configuration override system

### ⏳ Phase 5: Commands & Testing
- [ ] Write Artisan commands
- [ ] Create unit tests
- [ ] Integration tests
- [ ] Test with anonymized data

### ⏳ Phase 6: Documentation
- [ ] API documentation
- [ ] Installation guide
- [ ] Usage examples
- [ ] Troubleshooting guide

### ⏳ Phase 7: Dashboard Integration
- [ ] Unified data models
- [ ] Dashboard views
- [ ] Reporting tools
- [ ] Alerting system

---

## CRITICAL NEXT STEPS

To proceed with implementation, I need to verify the **Patrons table field names**:

### Required Field Names from `Polaris.Polaris.Patrons`:

**Please confirm these field names exist and are correct:**

```
Email Address Field: EmailAddress? Email1? EmailAddr?
First Name Field: NameFirst? FirstName? PatronFirstName?
Last Name Field: NameLast? LastName? PatronLastName?
Barcode Field: Barcode? PatronBarcode?
Street Address: StreetOne? Address1? MailingAddress1?
City: City?
State: State? StateAbbr?
Postal Code: PostalCode? Zip? ZipCode?
Country: Country?
Email Notices Flag: EmailNotices? EmailNotification?
Mail Notices Flag: MailNotices? MailNotification?
Expiration Date: ExpirationDate? ExpDate?
```

### How to Get This Information:

**Option 1: Describe Table**
```sql
SELECT
    COLUMN_NAME,
    DATA_TYPE,
    CHARACTER_MAXIMUM_LENGTH
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'Polaris'
  AND TABLE_NAME = 'Patrons'
ORDER BY ORDINAL_POSITION;
```

**Option 2: CREATE TABLE Script**
```sql
-- Run in SQL Server Management Studio:
-- Right-click Polaris.Polaris.Patrons
-- Script Table As → CREATE To → New Query Editor Window
```

---

## PACKAGE GOALS REVIEW

### Primary Goals ✅
- ✅ Track all email notifications sent by Polaris ILS
- ✅ Track all mail notifications sent by Polaris ILS
- ✅ Monitor delivery status (sent, failed, bounced)
- ✅ Provide consistent API with Shoutbomb package
- ✅ Enable unified notification dashboard

### Technical Goals ✅
- ✅ Use NotificationLog as primary data source
- ✅ Consistent field naming with Shoutbomb
- ✅ Database override configuration system
- ✅ Support multiple notification types
- ✅ Import historical data

---

## CONCLUSION

The design phase is complete. The package will:

1. **Import** notifications from `Polaris.Polaris.NotificationLog`
2. **Track** email and mail delivery status
3. **Monitor** failures and bounced emails
4. **Provide** consistent API with Shoutbomb package
5. **Enable** unified dashboard across all 4 notification channels

**Next Action:** Verify Patrons table field names and proceed with Phase 2 implementation.

---

**Design Status:** ✅ Complete
**Implementation Status:** ⏳ Awaiting Patrons field verification
**Estimated Timeline:** 1 week from field verification to MVP
**Documentation:** 3 comprehensive docs created
