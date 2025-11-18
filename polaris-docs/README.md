# Polaris Email/Mail Notification Monitoring Package

**Package Name:** `dcplibrary/notices-polaris`
**Purpose:** Monitor and log email and mail notifications sent by Polaris ILS
**Status:** 📋 Design Complete - Ready for Implementation

---

## 🎯 Overview

This Laravel package monitors and logs **email** and **mail** (physical mail) notifications sent by Polaris ILS, providing a consistent API with the existing `dcplibrary/notices-shoutbomb` package. Together, these packages will power a **unified notification monitoring dashboard** covering all four delivery channels:

- 📧 **Email** (DeliveryOptionID = 2) - This package
- 📮 **Mail** (DeliveryOptionID = 1) - This package
- 📞 **Voice** (DeliveryOptionID = 3) - Shoutbomb package
- 📱 **Text/SMS** (DeliveryOptionID = 8) - Shoutbomb package

---

## 📚 Documentation Index

### Quick Reference

| Document | Purpose | Audience |
|----------|---------|----------|
| **[README.md](README.md)** (this file) | Overview & quick start | Everyone |
| **[PACKAGE_DESIGN_SUMMARY.md](PACKAGE_DESIGN_SUMMARY.md)** | Complete design overview | Developers |
| **[FINALIZED_SQL_QUERIES.md](FINALIZED_SQL_QUERIES.md)** | Production-ready SQL queries | Database admins |
| **[API_VS_SQL_ANALYSIS.md](API_VS_SQL_ANALYSIS.md)** | API vs SQL efficiency comparison | Architects |
| **[POLARIS_SQL_QUERIES.md](POLARIS_SQL_QUERIES.md)** | Initial query design | Reference |
| **[POLARIS_EMAIL_MAIL_PACKAGE_DESIGN.md](POLARIS_EMAIL_MAIL_PACKAGE_DESIGN.md)** | Detailed design document | Architects |

### Read in This Order

1. **START HERE:** [README.md](README.md) - Overview
2. **DESIGN:** [PACKAGE_DESIGN_SUMMARY.md](PACKAGE_DESIGN_SUMMARY.md) - Full design
3. **QUERIES:** [FINALIZED_SQL_QUERIES.md](FINALIZED_SQL_QUERIES.md) - SQL exports

---

## 🔑 Key Features

- ✅ Track all email notifications sent by Polaris
- ✅ Track all mail notifications sent by Polaris
- ✅ Monitor delivery status (sent, failed, bounced)
- ✅ Consistent API and naming with Shoutbomb package
- ✅ Unified dashboard across all 4 notification channels
- ✅ Email failure tracking and alerting
- ✅ Patron contact list management
- ✅ Comprehensive statistics and reporting

---

## 🗄️ Data Source

### Primary Table: `Polaris.Polaris.NotificationLog`

This is the **simplest and most efficient** data source because it already contains:

- ✅ PatronID and PatronBarcode (no extra joins!)
- ✅ Email addresses in `DeliveryString` field
- ✅ Notification counts (holds, overdues, bills, etc.)
- ✅ Delivery status codes
- ✅ Timestamps for tracking
- ✅ Notification type IDs

### Supporting Tables

```
PatronRegistration → Names, email preferences
PatronAddresses → Linking table for mailing addresses
Addresses → Street addresses
PostalCodes → City, state, ZIP
NotificationStatuses → Status code meanings
NotificationTypes → Notification type descriptions
```

---

## 📊 Notification Status Codes

### Email Statuses

| Status ID | Description | Success? |
|-----------|-------------|----------|
| **12** | Email Completed | ✅ Yes |
| **13** | Email Failed - Invalid address | ❌ No |
| **14** | Email Failed | ❌ No |

### Mail Statuses

| Status ID | Description | Success? |
|-----------|-------------|----------|
| **15** | Mail Printed | ✅ Yes |

---

## 📦 Database Schema

### Core Tables

1. **`polaris_notification_log`** - Main import from Polaris NotificationLog
2. **`polaris_email_patrons`** - Active email patron mapping (like shoutbomb_voice_patrons)
3. **`polaris_mail_patrons`** - Active mail patron mapping with addresses
4. **`polaris_settings`** - Database configuration overrides
5. **`polaris_notification_statuses`** - Lookup table for status codes
6. **`polaris_notification_types`** - Lookup table for notification types

### Field Naming Consistency

Fields that **MUST** match between Polaris and Shoutbomb packages:

- `patron_id` (INT)
- `patron_barcode` (VARCHAR(20))
- `notification_type_id` (INT)
- `notification_status_id` (INT)
- `reporting_org_id` (INT)
- `import_date` (DATETIME)
- `created_at`, `updated_at` (TIMESTAMP)

---

## 🔄 Data Flow

### Email Notifications

```
Polaris ILS
  ↓
Sends Email via SMTP
  ↓
Logs to NotificationLog table
  ↓ (SQL export or direct import)
Laravel Package
  ↓
polaris_notification_log table
  ↓
Unified Dashboard
```

### Mail Notifications

```
Polaris ILS
  ↓
Prints Mail Notice
  ↓
Logs to NotificationLog table
  ↓ (SQL export or direct import)
Laravel Package
  ↓
polaris_notification_log table (with mailing address)
  ↓
Unified Dashboard
```

---

## 🚀 Quick Start

### Phase 1: Test SQL Queries ✅ READY

1. Review [FINALIZED_SQL_QUERIES.md](FINALIZED_SQL_QUERIES.md)
2. Test queries against your Polaris database
3. Verify AddressTypeID values for your system
4. Export sample data and anonymize for testing

### Phase 2: Package Setup ⏳ PENDING

```bash
# Create package directory
mkdir dcplibrary/notices-polaris

# Set up composer.json
composer init

# Create directory structure
mkdir -p src/{Models,Services,Commands,Http/Controllers}
mkdir -p database/{migrations,seeders}
mkdir -p tests
```

### Phase 3: Database Migrations ⏳ PENDING

Create migration files for:
- polaris_notification_log
- polaris_email_patrons
- polaris_mail_patrons
- polaris_settings
- polaris_notification_statuses
- polaris_notification_types

### Phase 4: Models & Services ⏳ PENDING

Implement:
- NotificationLog model
- EmailPatron model
- MailPatron model
- NotificationImportService
- EmailPatronService
- MailPatronService

### Phase 5: Artisan Commands ⏳ PENDING

```bash
php artisan polaris:import-notifications --days=7
php artisan polaris:import-email-patrons
php artisan polaris:import-mail-patrons
php artisan polaris:email-failures --days=30
php artisan polaris:stats --channel=email
```

### Phase 6: Dashboard Integration ⏳ PENDING

Build unified dashboard combining:
- Email notifications (this package)
- Mail notifications (this package)
- Voice notifications (shoutbomb package)
- Text notifications (shoutbomb package)

---

## 📈 Sample Queries

### Email Notifications (Last 7 Days)

```sql
SELECT * FROM Polaris.Polaris.NotificationLog
WHERE DeliveryOptionID = 2
  AND NotificationDateTime >= DATEADD(day, -7, GETDATE())
ORDER BY NotificationDateTime DESC
```

### Email Failures (Last 30 Days)

```sql
SELECT * FROM Polaris.Polaris.NotificationLog
WHERE DeliveryOptionID = 2
  AND NotificationStatusID IN (13, 14)
  AND NotificationDateTime >= DATEADD(day, -30, GETDATE())
```

### Email Success Rate

```sql
SELECT
    COUNT(*) AS Total,
    SUM(CASE WHEN NotificationStatusID = 12 THEN 1 ELSE 0 END) AS Success,
    SUM(CASE WHEN NotificationStatusID IN (13,14) THEN 1 ELSE 0 END) AS Failed,
    CAST(SUM(CASE WHEN NotificationStatusID = 12 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS DECIMAL(5,2)) AS SuccessRate
FROM Polaris.Polaris.NotificationLog
WHERE DeliveryOptionID = 2
  AND NotificationDateTime >= DATEADD(day, -30, GETDATE())
```

See [FINALIZED_SQL_QUERIES.md](FINALIZED_SQL_QUERIES.md) for complete production queries.

---

## 🎨 Dashboard Mockup

### Unified Notification Dashboard

```
╔══════════════════════════════════════════════════════════════╗
║  POLARIS NOTIFICATION MONITORING DASHBOARD                   ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  Last 30 Days Summary                                       ║
║  ┌────────────┬─────────┬─────────┬────────────┐           ║
║  │ Channel    │ Sent    │ Success │ Failed (%) │           ║
║  ├────────────┼─────────┼─────────┼────────────┤           ║
║  │ 📧 Email   │  5,234  │  5,187  │   47 (0.9%)│           ║
║  │ 📮 Mail    │    456  │    456  │    0 (0.0%)│           ║
║  │ 📞 Voice   │  1,892  │  1,745  │  147 (7.8%)│           ║
║  │ 📱 Text    │  3,567  │  3,512  │   55 (1.5%)│           ║
║  └────────────┴─────────┴─────────┴────────────┘           ║
║                                                              ║
║  Notification Types (Email)                                 ║
║  ┌─────────────────────┬─────────┬────────────┐            ║
║  │ Type                │ Count   │ Success %  │            ║
║  ├─────────────────────┼─────────┼────────────┤            ║
║  │ 1st Overdue         │  2,345  │    99.2%   │            ║
║  │ Hold Ready          │  1,876  │    99.8%   │            ║
║  │ Almost Overdue      │    678  │    98.5%   │            ║
║  │ Bills               │    234  │    97.4%   │            ║
║  │ 2nd Overdue         │    101  │    99.0%   │            ║
║  └─────────────────────┴─────────┴────────────┘            ║
║                                                              ║
║  Recent Failures (Email)                                    ║
║  ┌──────────────┬──────────────────────────┬──────────────┐║
║  │ Date/Time    │ Patron                   │ Reason       │║
║  ├──────────────┼──────────────────────────┼──────────────┤║
║  │ 11/18 08:45  │ Smith, John (233070...)  │ Invalid addr │║
║  │ 11/18 08:32  │ Doe, Jane (233070...)    │ Failed       │║
║  └──────────────┴──────────────────────────┴──────────────┘║
╚══════════════════════════════════════════════════════════════╝
```

---

## 🔧 Configuration

### Database Connection

```php
// config/polaris.php
return [
    'database' => [
        'host' => env('POLARIS_DB_HOST', 'polaris-sql.library.local'),
        'port' => env('POLARIS_DB_PORT', 1433),
        'database' => env('POLARIS_DB_NAME', 'Polaris'),
        'username' => env('POLARIS_DB_USERNAME'),
        'password' => env('POLARIS_DB_PASSWORD'),
    ],

    'import' => [
        'days_back' => env('POLARIS_IMPORT_DAYS', 7),
        'chunk_size' => env('POLARIS_CHUNK_SIZE', 1000),
    ],

    'address_type_id' => env('POLARIS_MAIL_ADDRESS_TYPE_ID', 1),
];
```

### Environment Variables

```env
# Polaris Database Connection
POLARIS_DB_HOST=polaris-sql.library.local
POLARIS_DB_PORT=1433
POLARIS_DB_NAME=Polaris
POLARIS_DB_USERNAME=polaris_readonly
POLARIS_DB_PASSWORD=your_secure_password

# Import Settings
POLARIS_IMPORT_DAYS=7
POLARIS_CHUNK_SIZE=1000
POLARIS_MAIL_ADDRESS_TYPE_ID=1
```

---

## 📋 Implementation Checklist

### ✅ Phase 1: Design & Analysis (COMPLETE)
- [x] Study Shoutbomb package structure
- [x] Analyze Polaris database schemas
- [x] Identify NotificationLog as primary source
- [x] Map notification statuses and types
- [x] Design consistent field naming
- [x] Create production SQL queries
- [x] Verify all Polaris table field names
- [x] Document complete data model

### ⏳ Phase 2: Package Setup (READY TO START)
- [ ] Create package directory structure
- [ ] Set up composer.json
- [ ] Create service provider
- [ ] Configure database connections
- [ ] Set up testing framework

### ⏳ Phase 3: Database Layer
- [ ] Write migration files (6 tables)
- [ ] Create model classes
- [ ] Add eloquent relationships
- [ ] Create seeder files
- [ ] Test migrations

### ⏳ Phase 4: Import Services
- [ ] NotificationImportService
- [ ] EmailPatronService
- [ ] MailPatronService
- [ ] Database configuration override system
- [ ] Batch processing utilities

### ⏳ Phase 5: Commands & Testing
- [ ] Write Artisan commands (6 commands)
- [ ] Create unit tests
- [ ] Integration tests
- [ ] Test with anonymized production data
- [ ] Performance testing

### ⏳ Phase 6: Documentation
- [ ] API documentation
- [ ] Installation guide
- [ ] Usage examples
- [ ] Troubleshooting guide
- [ ] Dashboard mockups

### ⏳ Phase 7: Dashboard Integration
- [ ] Unified data models
- [ ] Dashboard views (Laravel Livewire/Vue)
- [ ] Reporting tools
- [ ] Alerting system
- [ ] Export capabilities

---

## 🤝 Integration with Shoutbomb Package

### Shared Patterns

Both packages follow identical patterns:

| Feature | Shoutbomb | Polaris |
|---------|-----------|---------|
| **Prefix** | `shoutbomb_` | `polaris_` |
| **Patron Mapping** | `shoutbomb_voice_patrons`, `shoutbomb_text_patrons` | `polaris_email_patrons`, `polaris_mail_patrons` |
| **Settings Table** | `shoutbomb_settings` | `polaris_settings` |
| **Commands** | `shoutbomb:import-*` | `polaris:import-*` |
| **Service Provider** | `NoticesShoutbombServiceProvider` | `NoticesPolarisServiceProvider` |

### Unified Dashboard Query

```sql
-- Combined notifications across all 4 channels
SELECT 'Email' as channel, notification_type_id, COUNT(*) as total
FROM polaris_notification_log
WHERE delivery_option_id = 2
GROUP BY notification_type_id

UNION ALL

SELECT 'Mail' as channel, notification_type_id, COUNT(*) as total
FROM polaris_notification_log
WHERE delivery_option_id = 1
GROUP BY notification_type_id

UNION ALL

SELECT 'Voice' as channel, notification_type_id, COUNT(*) as total
FROM shoutbomb_holds
-- ... similar for voice

UNION ALL

SELECT 'Text' as channel, notification_type_id, COUNT(*) as total
FROM shoutbomb_holds
-- ... similar for text
```

---

## 📞 Support & Contact

**Package Maintainer:** DC Public Library
**Documentation:** See `/polaris-docs/` directory
**Issues:** GitHub Issues
**Questions:** Contact library IT department

---

## 📄 License

MIT License - See LICENSE file

---

## 🎉 Status

**Design:** ✅ 100% Complete
**SQL Queries:** ✅ Production Ready
**Implementation:** ⏳ Ready to Begin
**Estimated Timeline:** 1 week to MVP

---

**Last Updated:** November 18, 2025
**Version:** 0.1.0-design
**Next Milestone:** Package Setup & Database Migrations
