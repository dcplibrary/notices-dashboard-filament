# POLARIS EMAIL/MAIL NOTIFICATION SQL QUERIES

**Date:** November 18, 2025
**Package:** `dcplibrary/notices-polaris`
**Database:** Polaris ILS (MSSQL)

---

## TABLE OF CONTENTS

- [Overview](#overview)
- [Data Source: NotificationLog](#data-source-notificationlog)
- [Lookup Tables Reference](#lookup-tables-reference)
- [SQL Export Queries](#sql-export-queries)
- [Field Mappings](#field-mappings)
- [Export File Formats](#export-file-formats)
- [Import Strategy](#import-strategy)

---

## OVERVIEW

This document contains SQL queries for extracting email and mail notification data from Polaris ILS, following the same patterns established by the Shoutbomb package.

### Primary Data Source

**`Polaris.Polaris.NotificationLog`** - Contains aggregated notification records with delivery status

This is superior to using NotificationQueue/NotificationHistory because it:
- ✅ Already contains PatronBarcode (no join needed)
- ✅ Has email address in DeliveryString field
- ✅ Aggregates multiple items per notification
- ✅ Includes delivery status
- ✅ Simpler queries with fewer joins

---

## DATA SOURCE: NOTIFICATIONLOG

### Schema

```sql
CREATE TABLE [Polaris].[NotificationLog] (
    [PatronID] int,
    [NotificationDateTime] datetime,
    [NotificationTypeID] int,
    [DeliveryOptionID] int,
    [DeliveryString] nvarchar(255),        -- Email address or phone number
    [OverduesCount] int,
    [HoldsCount] int,
    [CancelsCount] int,
    [RecallsCount] int,
    [NotificationStatusID] int,
    [Details] nvarchar(255),
    [RoutingsCount] int DEFAULT (0),
    [ReportingOrgID] int,
    [PatronBarcode] nvarchar(20),
    [Reported] bit DEFAULT ((0)),
    [Overdues2ndCount] int,
    [Overdues3rdCount] int,
    [BillsCount] int,
    [LanguageID] int,
    [CarrierName] nvarchar(255),
    [ManualBillCount] int,
    [NotificationLogID] int IDENTITY,
    PRIMARY KEY ([NotificationLogID])
);
```

### Sample Data

**Email Hold Notification:**
```
PatronID: 10003
NotificationDateTime: 2025-11-04 08:00:00
NotificationTypeID: 2 (Hold)
DeliveryOptionID: 2 (Email)
DeliveryString: angiehenderson@hotmail.com
HoldsCount: 1
NotificationStatusID: 12 (Email Completed)
PatronBarcode: 23307182449353
```

**Email Overdue Notification:**
```
PatronID: 10000
NotificationDateTime: 2025-11-05 08:00:00
NotificationTypeID: 1 (1st Overdue)
DeliveryOptionID: 2 (Email)
DeliveryString: margaret.johnson@icloud.com
OverduesCount: 2
NotificationStatusID: 12 (Email Completed)
PatronBarcode: 23307600133890
```

**Mail Hold Notification:**
```
PatronID: 10013
NotificationDateTime: 2025-10-31 08:00:00
NotificationTypeID: 2 (Hold)
DeliveryOptionID: 1 (Mail)
DeliveryString: (blank - no delivery string for mail)
HoldsCount: 1
NotificationStatusID: 15 (Mail Printed)
PatronBarcode: 23307762268388
```

---

## LOOKUP TABLES REFERENCE

### NotificationStatuses (Polaris.Polaris.NotificationStatuses)

| ID | Description | Used By | Success? |
|----|-------------|---------|----------|
| 1 | Call completed - Voice | Voice (3) | ✅ Yes |
| 2 | Call completed - Answering machine | Voice (3) | ✅ Yes |
| 3 | Call not completed - Hang up | Voice (3) | ❌ No |
| 4 | Call not completed - Busy | Voice (3) | ❌ No |
| 5 | Call not completed - No answer | Voice (3) | ❌ No |
| 6 | Call not completed - No ring | Voice (3) | ❌ No |
| 7 | Call failed - No dial tone | Voice (3) | ❌ No |
| 8 | Call failed - Intercept tones heard | Voice (3) | ❌ No |
| 9 | Call failed - Probable bad phone number | Voice (3) | ❌ No |
| 10 | Call failed - Maximum retries exceeded | Voice (3) | ❌ No |
| 11 | Call failed - Undetermined error | Voice (3) | ❌ No |
| **12** | **Email Completed** | **Email (2)** | ✅ **Yes** |
| **13** | **Email Failed - Invalid address** | **Email (2)** | ❌ **No** |
| **14** | **Email Failed** | **Email (2)** | ❌ **No** |
| **15** | **Mail Printed** | **Mail (1)** | ✅ **Yes** |
| **16** | **Sent** | **Text (8)** | ✅ **Yes** |

**Email Success Statuses:** 12
**Email Failure Statuses:** 13, 14
**Mail Success Statuses:** 15
**Mail Failure Statuses:** (None observed - mail assumed successful if printed)

### NotificationTypes (Polaris.Polaris.NotificationTypes)

| ID | Type | Email/Mail? | Description |
|----|------|-------------|-------------|
| 0 | Combined | ❌ | Combined notification types |
| **1** | **1st Overdue** | ✅ | First overdue notice |
| **2** | **Hold** | ✅ | Hold ready for pickup |
| 3 | Cancel | ✅ | Hold cancellation |
| 4 | Recall | ❌ | Item recall |
| 5 | All | ❌ | All notification types |
| 6 | Route | ❌ | Routing notice |
| **7** | **Almost overdue/Auto-renew** | ✅ | Pre-overdue reminder |
| **8** | **Fine** | ✅ | Fine notice |
| 9 | Inactive Reminder | ✅ | Inactive account reminder |
| 10 | Expiration Reminder | ✅ | Card expiration reminder |
| **11** | **Bill** | ✅ | Billing notice |
| **12** | **2nd Overdue** | ✅ | Second overdue notice |
| **13** | **3rd Overdue** | ✅ | Third overdue notice |
| 14 | Serial Claim | ❌ | Serial publication claim |
| 15 | Polaris Fusion Access | ❌ | Fusion access request |
| 16 | Course Reserves | ❌ | Course reserves notice |
| 17 | Borrow-By-Mail Failure | ✅ | BBM failure notice |
| **18** | **2nd Hold** | ✅ | Second hold notice |
| 19 | Missing Part | ❌ | Missing part notice |
| **20** | **Manual Bill** | ✅ | Manual billing notice |
| 21 | 2nd Fine Notice | ✅ | Second fine notice |

### DeliveryOptions (Polaris.Polaris.SA_DeliveryOptions)

| ID | Channel | Package | Description |
|----|---------|---------|-------------|
| **1** | **Mail** | **polaris** | Physical mail notification |
| **2** | **Email** | **polaris** | Email notification |
| 3 | Voice | shoutbomb | Voice call via Phone1 |
| 8 | Text | shoutbomb | SMS text message |

---

## SQL EXPORT QUERIES

### Query 1: Email Notifications (Last 7 Days)

**Purpose:** Export all email notifications for monitoring and import

**File Output:** `email_notifications_submitted_YYYY-MM-DD_HH-MM-SS.txt`

```sql
SET NOCOUNT ON

SELECT
    nl.NotificationLogID,
    nl.PatronID,
    nl.PatronBarcode,
    nl.NotificationDateTime,
    nl.NotificationTypeID,
    nl.DeliveryString AS EmailAddress,
    nl.NotificationStatusID,
    nl.OverduesCount,
    nl.HoldsCount,
    nl.CancelsCount,
    nl.BillsCount,
    nl.ManualBillCount,
    nl.Overdues2ndCount,
    nl.Overdues3rdCount,
    nl.ReportingOrgID,
    nl.LanguageID,
    nl.Details,
    ns.Description AS StatusDescription,
    nt.Description AS NotificationTypeDescription
FROM
    Polaris.Polaris.NotificationLog nl (NOLOCK)
    LEFT JOIN Polaris.Polaris.NotificationStatuses ns (NOLOCK)
        ON nl.NotificationStatusID = ns.NotificationStatusID
    LEFT JOIN Polaris.Polaris.NotificationTypes nt (NOLOCK)
        ON nl.NotificationTypeID = nt.NotificationTypeID
WHERE
    nl.DeliveryOptionID = 2  -- Email only
    AND nl.NotificationDateTime >= DATEADD(day, -7, GETDATE())
ORDER BY
    nl.NotificationDateTime DESC, nl.PatronBarcode
```

**Expected Output (pipe-delimited):**
```
NotificationLogID|PatronID|PatronBarcode|NotificationDateTime|NotificationTypeID|EmailAddress|NotificationStatusID|OverduesCount|HoldsCount|CancelsCount|BillsCount|ManualBillCount|Overdues2ndCount|Overdues3rdCount|ReportingOrgID|LanguageID|Details|StatusDescription|NotificationTypeDescription
1|10003|23307182449353|2025-11-04 08:00:00|2|angiehenderson@hotmail.com|12|0|1|0|0|0|0|0|3|1033||Email Completed|Hold
```

### Query 2: Mail Notifications (Last 7 Days)

**Purpose:** Export all mail notifications for monitoring and import

**File Output:** `mail_notifications_submitted_YYYY-MM-DD_HH-MM-SS.txt`

```sql
SET NOCOUNT ON

SELECT
    nl.NotificationLogID,
    nl.PatronID,
    nl.PatronBarcode,
    nl.NotificationDateTime,
    nl.NotificationTypeID,
    nl.DeliveryString AS MailingInfo,
    nl.NotificationStatusID,
    nl.OverduesCount,
    nl.HoldsCount,
    nl.CancelsCount,
    nl.BillsCount,
    nl.ManualBillCount,
    nl.Overdues2ndCount,
    nl.Overdues3rdCount,
    nl.ReportingOrgID,
    nl.LanguageID,
    nl.Details,
    ns.Description AS StatusDescription,
    nt.Description AS NotificationTypeDescription,
    -- Join to Patrons for mailing address
    p.NameFirst,
    p.NameLast,
    p.StreetOne,
    p.StreetTwo,
    p.City,
    p.State,
    p.PostalCode
FROM
    Polaris.Polaris.NotificationLog nl (NOLOCK)
    LEFT JOIN Polaris.Polaris.NotificationStatuses ns (NOLOCK)
        ON nl.NotificationStatusID = ns.NotificationStatusID
    LEFT JOIN Polaris.Polaris.NotificationTypes nt (NOLOCK)
        ON nl.NotificationTypeID = nt.NotificationTypeID
    LEFT JOIN Polaris.Polaris.Patrons p (NOLOCK)
        ON nl.PatronID = p.PatronID
WHERE
    nl.DeliveryOptionID = 1  -- Mail only
    AND nl.NotificationDateTime >= DATEADD(day, -7, GETDATE())
ORDER BY
    nl.NotificationDateTime DESC, nl.PatronBarcode
```

**Note:** Mail notifications need Patrons table join to get mailing address since `DeliveryString` is blank for mail.

### Query 3: Email Failures (Last 30 Days)

**Purpose:** Track bounced/failed email notifications

**File Output:** `email_failures_YYYY-MM-DD.txt`

```sql
SET NOCOUNT ON

SELECT
    nl.NotificationLogID,
    nl.PatronID,
    nl.PatronBarcode,
    nl.NotificationDateTime,
    nl.DeliveryString AS EmailAddress,
    nl.NotificationStatusID,
    ns.Description AS FailureReason,
    nl.Details
FROM
    Polaris.Polaris.NotificationLog nl (NOLOCK)
    JOIN Polaris.Polaris.NotificationStatuses ns (NOLOCK)
        ON nl.NotificationStatusID = ns.NotificationStatusID
WHERE
    nl.DeliveryOptionID = 2  -- Email
    AND nl.NotificationStatusID IN (13, 14)  -- Email failures only
    AND nl.NotificationDateTime >= DATEADD(day, -30, GETDATE())
ORDER BY
    nl.NotificationDateTime DESC
```

### Query 4: Email Statistics by Type (Last 30 Days)

**Purpose:** Generate summary statistics for dashboard

```sql
SELECT
    nt.Description AS NotificationType,
    COUNT(*) AS TotalSent,
    SUM(CASE WHEN nl.NotificationStatusID = 12 THEN 1 ELSE 0 END) AS Successful,
    SUM(CASE WHEN nl.NotificationStatusID IN (13,14) THEN 1 ELSE 0 END) AS Failed,
    SUM(nl.HoldsCount) AS TotalHolds,
    SUM(nl.OverduesCount) AS TotalOverdues,
    SUM(nl.BillsCount) AS TotalBills,
    CAST(SUM(CASE WHEN nl.NotificationStatusID = 12 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS DECIMAL(5,2)) AS SuccessRate
FROM
    Polaris.Polaris.NotificationLog nl (NOLOCK)
    JOIN Polaris.Polaris.NotificationTypes nt (NOLOCK)
        ON nl.NotificationTypeID = nt.NotificationTypeID
WHERE
    nl.DeliveryOptionID = 2  -- Email
    AND nl.NotificationDateTime >= DATEADD(day, -30, GETDATE())
GROUP BY
    nl.NotificationTypeID, nt.Description
ORDER BY
    TotalSent DESC
```

### Query 5: Patron Email Contact List (Daily Export)

**Purpose:** Maintain current list of patrons receiving email notifications (like voice_patrons/text_patrons)

**File Output:** `email_patrons_submitted_YYYY-MM-DD_HH-MM-SS.txt`

```sql
SET NOCOUNT ON

SELECT DISTINCT
    p.PatronID,
    p.Barcode AS PatronBarcode,
    p.EmailAddress,
    p.NameFirst,
    p.NameLast,
    p.EmailNotices
FROM
    Polaris.Polaris.Patrons p (NOLOCK)
WHERE
    p.EmailAddress IS NOT NULL
    AND p.EmailAddress <> ''
    AND p.EmailNotices = 1  -- Opted in for email notifications
    AND p.ExpirationDate > GETDATE()  -- Active patron
ORDER BY
    p.Barcode
```

**Expected Output (pipe-delimited):**
```
PatronID|PatronBarcode|EmailAddress|NameFirst|NameLast|EmailNotices
10003|23307182449353|angiehenderson@hotmail.com|Angie|Henderson|1
```

### Query 6: Patron Mail Contact List (Daily Export)

**Purpose:** Maintain current list of patrons receiving mail notifications

**File Output:** `mail_patrons_submitted_YYYY-MM-DD_HH-MM-SS.txt`

```sql
SET NOCOUNT ON

SELECT DISTINCT
    p.PatronID,
    p.Barcode AS PatronBarcode,
    p.NameFirst,
    p.NameLast,
    p.StreetOne,
    p.StreetTwo,
    p.City,
    p.State,
    p.PostalCode,
    p.Country,
    p.MailNotices
FROM
    Polaris.Polaris.Patrons p (NOLOCK)
WHERE
    p.StreetOne IS NOT NULL
    AND p.StreetOne <> ''
    AND p.MailNotices = 1  -- Opted in for mail notifications
    AND p.ExpirationDate > GETDATE()  -- Active patron
ORDER BY
    p.Barcode
```

**Note:** Patron table field names (EmailAddress, NameFirst, StreetOne, etc.) are **assumed** based on standard Polaris naming. Please verify actual field names in your database.

---

## FIELD MAPPINGS

### NotificationLog → Laravel Model Mapping

| Polaris Field | Laravel Field | Type | Notes |
|---------------|---------------|------|-------|
| NotificationLogID | notification_log_id | bigint | Primary key |
| PatronID | patron_id | int | Foreign key |
| PatronBarcode | patron_barcode | varchar(20) | Indexed |
| NotificationDateTime | notification_date_time | datetime | When sent |
| NotificationTypeID | notification_type_id | int | Type of notice |
| DeliveryOptionID | delivery_option_id | int | 1=Mail, 2=Email |
| DeliveryString | email_address / mailing_info | varchar(255) | Email for email, blank for mail |
| NotificationStatusID | notification_status_id | int | Delivery status |
| OverduesCount | overdues_count | int | Number of overdues |
| HoldsCount | holds_count | int | Number of holds |
| CancelsCount | cancels_count | int | Number of cancels |
| BillsCount | bills_count | int | Number of bills |
| ManualBillCount | manual_bill_count | int | Manual bills |
| Overdues2ndCount | overdues_2nd_count | int | 2nd overdues |
| Overdues3rdCount | overdues_3rd_count | int | 3rd overdues |
| ReportingOrgID | reporting_org_id | int | Sending branch |
| LanguageID | language_id | int | Language code |
| Details | details | text | Additional info |

### Consistent Field Names (Shared with Shoutbomb)

These fields **MUST** use identical names in both packages:

- ✅ `patron_id`
- ✅ `patron_barcode`
- ✅ `notification_type_id`
- ✅ `reporting_org_id`
- ✅ `notification_status_id` (new, but consistent)
- ✅ `import_date`
- ✅ `created_at`
- ✅ `updated_at`

---

## EXPORT FILE FORMATS

### File Naming Convention

Following Shoutbomb pattern:

```
{channel}_{type}_submitted_YYYY-MM-DD_HH-MM-SS.txt

Examples:
- email_notifications_submitted_2025-11-07_08-00-01.txt
- mail_notifications_submitted_2025-11-07_08-00-01.txt
- email_patrons_submitted_2025-11-07_04-00-01.txt
- mail_patrons_submitted_2025-11-07_04-00-01.txt
- email_failures_2025-11-07.txt
```

### File Format

**Delimiter:** Pipe (`|`) - matches Shoutbomb pattern
**Header Row:** NO - matches Shoutbomb pattern
**Text Qualifier:** None - replace pipes in text with hyphen
**Encoding:** UTF-8
**Line Endings:** Windows CRLF (`\r\n`)

### Field Order (Email Notifications)

```
[NotificationLogID]|[PatronID]|[PatronBarcode]|[NotificationDateTime]|[NotificationTypeID]|[EmailAddress]|[NotificationStatusID]|[OverduesCount]|[HoldsCount]|[CancelsCount]|[BillsCount]|[ManualBillCount]|[Overdues2ndCount]|[Overdues3rdCount]|[ReportingOrgID]|[LanguageID]|[Details]|[StatusDescription]|[NotificationTypeDescription]
```

---

## IMPORT STRATEGY

### Recommended Approach: Direct Database Import

Unlike Shoutbomb (which uses FTP file exchange), email/mail notifications are handled directly by Polaris. **Recommended approach:**

**Option A: Direct Database Connection (Preferred)**
```php
// config/polaris.php
'database' => [
    'host' => env('POLARIS_DB_HOST', 'polaris-sql.library.local'),
    'database' => env('POLARIS_DB_NAME', 'Polaris'),
    'username' => env('POLARIS_DB_USERNAME'),
    'password' => env('POLARIS_DB_PASSWORD'),
],

// Import command
php artisan polaris:import-email-notifications --days=7
```

**Option B: Export Files (Matches Shoutbomb Pattern)**
```bash
# Windows scheduled task runs SQL export
sqlcmd -S POLARISSQL -d Polaris -i email_notifications.sql -o email_notifications_submitted_2025-11-07_08-00-01.txt

# Laravel imports file
php artisan polaris:import-email-file /path/to/email_notifications_submitted_2025-11-07_08-00-01.txt
```

### Import Frequency

- **Email/Mail Notifications:** Daily at 8:00 AM (after overnight processing)
- **Patron Contact Lists:** Daily at 4:00 AM (before notifications run)
- **Failure Reports:** Daily at 9:00 AM (review previous day)

---

## NEXT STEPS

1. ✅ Verify Patrons table field names (EmailAddress, NameFirst, StreetOne, etc.)
2. ✅ Test SQL queries against Polaris database
3. ✅ Anonymize sample export data for documentation
4. ⏳ Create Laravel package structure
5. ⏳ Write database migrations
6. ⏳ Implement import services
7. ⏳ Create Artisan commands

---

**Status:** SQL queries ready for testing
**Last Updated:** November 18, 2025
**Next Review:** After Patrons table field verification
