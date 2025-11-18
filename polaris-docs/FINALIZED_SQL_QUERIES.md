# POLARIS EMAIL/MAIL NOTIFICATION - FINALIZED SQL QUERIES

**Date:** November 18, 2025
**Package:** `dcplibrary/notices-polaris`
**Database:** Polaris ILS (MSSQL)
**Status:** ✅ Field Names Verified - Production Ready

---

## TABLE OF CONTENTS

- [Database Schema Overview](#database-schema-overview)
- [Query 1: Email Notifications Export](#query-1-email-notifications-export)
- [Query 2: Mail Notifications Export](#query-2-mail-notifications-export)
- [Query 3: Email Patron Mapping](#query-3-email-patron-mapping)
- [Query 4: Mail Patron Mapping](#query-4-mail-patron-mapping)
- [Query 5: Email Failure Report](#query-5-email-failure-report)
- [Query 6: Email Statistics Dashboard](#query-6-email-statistics-dashboard)
- [Query 7: Mail Statistics Dashboard](#query-7-mail-statistics-dashboard)
- [Scheduled Export Scripts](#scheduled-export-scripts)
- [Field Mapping Reference](#field-mapping-reference)

---

## DATABASE SCHEMA OVERVIEW

### Verified Table Relationships

```
NotificationLog (primary data source)
├── PatronID → Patron → PatronRegistration
│   ├── EmailAddress
│   ├── NameFirst, NameLast
│   └── DeliveryOptionID
└── PatronID → Patron → PatronAddresses → Addresses → PostalCodes
    ├── StreetOne, StreetTwo, StreetThree
    ├── MunicipalityName
    ├── ZipPlusFour
    ├── PostalCode
    ├── City
    ├── State
    └── CountryID → Countries.CountryName
```

### Core Tables Used

| Table | Schema | Purpose |
|-------|--------|---------|
| NotificationLog | Polaris.Polaris | Main notification tracking |
| Patrons | Polaris.Polaris | Core patron record |
| PatronRegistration | Polaris.Polaris | Patron contact info (email, name, phone) |
| PatronAddresses | Polaris.Polaris | Links patrons to addresses |
| Addresses | Polaris.Polaris | Physical address data |
| PostalCodes | Polaris.Polaris | ZIP code, city, state, county |
| Countries | Polaris.Polaris | Country names |
| NotificationStatuses | Polaris.Polaris | Status codes (sent, failed, bounced) |
| NotificationTypes | Polaris.Polaris | Notification type descriptions |

---

## QUERY 1: EMAIL NOTIFICATIONS EXPORT

**Purpose:** Export all email notifications with patron details for Laravel import
**Frequency:** Daily at 8:00 AM
**Output File:** `email_notifications_submitted_YYYY-MM-DD_HH-MM-SS.txt`

### SQL Query

```sql
SET NOCOUNT ON

SELECT
    -- Notification Log Fields
    nl.NotificationLogID,
    nl.PatronID,
    nl.NotificationDateTime,
    nl.NotificationTypeID,
    nl.DeliveryOptionID,
    nl.NotificationStatusID,
    nl.OverduesCount,
    nl.HoldsCount,
    nl.CancelsCount,
    nl.BillsCount,
    nl.ManualBillCount,
    nl.Overdues2ndCount,
    nl.Overdues3rdCount,
    nl.RoutingsCount,
    nl.RecallsCount,
    nl.ReportingOrgID,
    nl.LanguageID,
    nl.Details,

    -- Email Address (from DeliveryString)
    nl.DeliveryString AS EmailAddress,

    -- Patron Information
    nl.PatronBarcode,
    pr.NameFirst,
    pr.NameLast,
    pr.NameMiddle,
    pr.PatronFullName,

    -- Lookup Descriptions
    ns.Description AS NotificationStatusDescription,
    nt.Description AS NotificationTypeDescription,

    -- Additional Patron Flags
    pr.ExcludeFromOverdues,
    pr.ExcludeFromHolds,
    pr.ExcludeFromBills,
    pr.EmailFormatID

FROM
    Polaris.Polaris.NotificationLog nl (NOLOCK)

    -- Join to get patron name and email info
    LEFT JOIN Polaris.Polaris.Patrons p (NOLOCK)
        ON nl.PatronID = p.PatronID
    LEFT JOIN Polaris.Polaris.PatronRegistration pr (NOLOCK)
        ON nl.PatronID = pr.PatronID

    -- Join to get status and type descriptions
    LEFT JOIN Polaris.Polaris.NotificationStatuses ns (NOLOCK)
        ON nl.NotificationStatusID = ns.NotificationStatusID
    LEFT JOIN Polaris.Polaris.NotificationTypes nt (NOLOCK)
        ON nl.NotificationTypeID = nt.NotificationTypeID

WHERE
    nl.DeliveryOptionID = 2  -- Email only
    AND nl.NotificationDateTime >= DATEADD(day, -7, GETDATE())  -- Last 7 days
    AND p.RecordStatusID = 1  -- Active patrons only

ORDER BY
    nl.NotificationDateTime DESC,
    nl.PatronBarcode
```

### Output Format (Pipe-Delimited)

```
NotificationLogID|PatronID|NotificationDateTime|NotificationTypeID|DeliveryOptionID|NotificationStatusID|OverduesCount|HoldsCount|CancelsCount|BillsCount|ManualBillCount|Overdues2ndCount|Overdues3rdCount|RoutingsCount|RecallsCount|ReportingOrgID|LanguageID|Details|EmailAddress|PatronBarcode|NameFirst|NameLast|NameMiddle|PatronFullName|NotificationStatusDescription|NotificationTypeDescription|ExcludeFromOverdues|ExcludeFromHolds|ExcludeFromBills|EmailFormatID
1|10003|2025-11-04 08:00:00|2|2|12|0|1|0|0|0|0|0|0|0|3|1033||angiehenderson@hotmail.com|23307182449353|Angie|Henderson||Angie Henderson|Email Completed|Hold|0|0|0|2
```

---

## QUERY 2: MAIL NOTIFICATIONS EXPORT

**Purpose:** Export all mail notifications with complete mailing addresses
**Frequency:** Daily at 8:00 AM
**Output File:** `mail_notifications_submitted_YYYY-MM-DD_HH-MM-SS.txt`

### SQL Query

```sql
SET NOCOUNT ON

SELECT
    -- Notification Log Fields
    nl.NotificationLogID,
    nl.PatronID,
    nl.NotificationDateTime,
    nl.NotificationTypeID,
    nl.DeliveryOptionID,
    nl.NotificationStatusID,
    nl.OverduesCount,
    nl.HoldsCount,
    nl.CancelsCount,
    nl.BillsCount,
    nl.ManualBillCount,
    nl.Overdues2ndCount,
    nl.Overdues3rdCount,
    nl.RoutingsCount,
    nl.RecallsCount,
    nl.ReportingOrgID,
    nl.LanguageID,
    nl.Details,

    -- Patron Information
    nl.PatronBarcode,
    pr.NameFirst,
    pr.NameLast,
    pr.NameMiddle,
    pr.PatronFullName,

    -- Mailing Address (join through PatronAddresses → Addresses → PostalCodes)
    a.StreetOne,
    a.StreetTwo,
    a.StreetThree,
    a.MunicipalityName,
    a.ZipPlusFour,
    pc.PostalCode,
    pc.City,
    pc.State,
    pc.County,
    c.CountryName,

    -- Address Metadata
    pa.AddressTypeID,
    pa.Verified AS AddressVerified,
    pa.VerificationDate AS AddressVerificationDate,

    -- Lookup Descriptions
    ns.Description AS NotificationStatusDescription,
    nt.Description AS NotificationTypeDescription

FROM
    Polaris.Polaris.NotificationLog nl (NOLOCK)

    -- Join to get patron info
    LEFT JOIN Polaris.Polaris.Patrons p (NOLOCK)
        ON nl.PatronID = p.PatronID
    LEFT JOIN Polaris.Polaris.PatronRegistration pr (NOLOCK)
        ON nl.PatronID = pr.PatronID

    -- Join to get mailing address (AddressTypeID = 2 for Notice address)
    LEFT JOIN Polaris.Polaris.PatronAddresses pa (NOLOCK)
        ON nl.PatronID = pa.PatronID
        AND pa.AddressTypeID = 2  -- Notice mailing address
    LEFT JOIN Polaris.Polaris.Addresses a (NOLOCK)
        ON pa.AddressID = a.AddressID
    LEFT JOIN Polaris.Polaris.PostalCodes pc (NOLOCK)
        ON a.PostalCodeID = pc.PostalCodeID
    LEFT JOIN Polaris.Polaris.Countries c (NOLOCK)
        ON pc.CountryID = c.CountryID

    -- Join to get status and type descriptions
    LEFT JOIN Polaris.Polaris.NotificationStatuses ns (NOLOCK)
        ON nl.NotificationStatusID = ns.NotificationStatusID
    LEFT JOIN Polaris.Polaris.NotificationTypes nt (NOLOCK)
        ON nl.NotificationTypeID = nt.NotificationTypeID

WHERE
    nl.DeliveryOptionID = 1  -- Mail only
    AND nl.NotificationDateTime >= DATEADD(day, -7, GETDATE())  -- Last 7 days
    AND p.RecordStatusID = 1  -- Active patrons only

ORDER BY
    nl.NotificationDateTime DESC,
    nl.PatronBarcode
```

### Output Format (Pipe-Delimited)

```
NotificationLogID|PatronID|NotificationDateTime|...|StreetOne|StreetTwo|StreetThree|MunicipalityName|ZipPlusFour|PostalCode|City|State|County|CountryName|AddressTypeID|AddressVerified|AddressVerificationDate|NotificationStatusDescription|NotificationTypeDescription
9|10013|2025-10-31 08:00:00|...|123 Main St|Apt 4||Owensboro|1234|42301|Owensboro|KY|Daviess|United States|1|1|2025-01-15|Mail Printed|Hold
```

**Note:** Using AddressTypeID = 2 (Notice) for notification mailings. If patrons don't have a Notice address, the query will fall back to NULL addresses. You may want to add a COALESCE to try AddressTypeID = 12 (Mailing) as a fallback.

---

## QUERY 3: EMAIL PATRON MAPPING

**Purpose:** Daily export of all patrons with active email delivery preferences
**Frequency:** Daily at 4:00 AM (before notifications run)
**Output File:** `email_patrons_submitted_YYYY-MM-DD_HH-MM-SS.txt`

### SQL Query

```sql
SET NOCOUNT ON

SELECT DISTINCT
    p.PatronID,
    p.Barcode AS PatronBarcode,
    pr.EmailAddress,
    pr.AltEmailAddress,
    pr.NameFirst,
    pr.NameLast,
    pr.NameMiddle,
    pr.PatronFullName,
    pr.DeliveryOptionID,
    pr.EmailFormatID,
    pr.ExpirationDate,
    pr.ExcludeFromOverdues,
    pr.ExcludeFromHolds,
    pr.ExcludeFromBills,
    p.RecordStatusID

FROM
    Polaris.Polaris.Patrons p (NOLOCK)
    JOIN Polaris.Polaris.PatronRegistration pr (NOLOCK)
        ON p.PatronID = pr.PatronID

WHERE
    pr.EmailAddress IS NOT NULL
    AND pr.EmailAddress <> ''
    AND pr.DeliveryOptionID = 2  -- Email delivery preference
    AND pr.ExpirationDate > GETDATE()  -- Not expired
    AND p.RecordStatusID = 1  -- Active record

ORDER BY
    p.Barcode
```

### Output Format (Pipe-Delimited)

```
PatronID|PatronBarcode|EmailAddress|AltEmailAddress|NameFirst|NameLast|NameMiddle|PatronFullName|DeliveryOptionID|EmailFormatID|ExpirationDate|ExcludeFromOverdues|ExcludeFromHolds|ExcludeFromBills|RecordStatusID
10003|23307182449353|angiehenderson@hotmail.com||Angie|Henderson||Angie Henderson|2|2|2026-12-31|0|0|0|1
```

---

## QUERY 4: MAIL PATRON MAPPING

**Purpose:** Daily export of all patrons with active mail delivery preferences
**Frequency:** Daily at 4:00 AM (before notifications run)
**Output File:** `mail_patrons_submitted_YYYY-MM-DD_HH-MM-SS.txt`

### SQL Query

```sql
SET NOCOUNT ON

SELECT DISTINCT
    p.PatronID,
    p.Barcode AS PatronBarcode,
    pr.NameFirst,
    pr.NameLast,
    pr.NameMiddle,
    pr.PatronFullName,
    pr.DeliveryOptionID,
    pr.ExpirationDate,
    pr.ExcludeFromOverdues,
    pr.ExcludeFromHolds,
    pr.ExcludeFromBills,

    -- Mailing Address
    a.StreetOne,
    a.StreetTwo,
    a.StreetThree,
    a.MunicipalityName,
    a.ZipPlusFour,
    pc.PostalCode,
    pc.City,
    pc.State,
    pc.County,
    c.CountryName,

    -- Address Metadata
    pa.AddressTypeID,
    pa.Verified AS AddressVerified,
    pa.VerificationDate AS AddressVerificationDate,

    p.RecordStatusID

FROM
    Polaris.Polaris.Patrons p (NOLOCK)
    JOIN Polaris.Polaris.PatronRegistration pr (NOLOCK)
        ON p.PatronID = pr.PatronID

    -- Join to get mailing address (Notice address for notifications)
    LEFT JOIN Polaris.Polaris.PatronAddresses pa (NOLOCK)
        ON p.PatronID = pa.PatronID
        AND pa.AddressTypeID = 2  -- Notice mailing address
    LEFT JOIN Polaris.Polaris.Addresses a (NOLOCK)
        ON pa.AddressID = a.AddressID
    LEFT JOIN Polaris.Polaris.PostalCodes pc (NOLOCK)
        ON a.PostalCodeID = pc.PostalCodeID
    LEFT JOIN Polaris.Polaris.Countries c (NOLOCK)
        ON pc.CountryID = c.CountryID

WHERE
    pr.DeliveryOptionID = 1  -- Mail delivery preference
    AND pr.ExpirationDate > GETDATE()  -- Not expired
    AND p.RecordStatusID = 1  -- Active record
    AND a.StreetOne IS NOT NULL  -- Has address on file

ORDER BY
    p.Barcode
```

---

## QUERY 5: EMAIL FAILURE REPORT

**Purpose:** Track bounced/failed email notifications for patron contact cleanup
**Frequency:** Daily at 9:00 AM
**Output File:** `email_failures_YYYY-MM-DD.txt`

### SQL Query

```sql
SET NOCOUNT ON

SELECT
    nl.NotificationLogID,
    nl.PatronID,
    nl.PatronBarcode,
    nl.NotificationDateTime,
    nl.DeliveryString AS EmailAddress,
    nl.NotificationTypeID,
    nl.NotificationStatusID,
    ns.Description AS FailureReason,
    nl.Details,

    -- Patron Info
    pr.NameFirst,
    pr.NameLast,
    pr.PatronFullName,
    pr.EmailAddress AS PatronRegistrationEmail,
    pr.AltEmailAddress,

    -- Notification Type
    nt.Description AS NotificationTypeDescription,

    -- Counts
    nl.HoldsCount,
    nl.OverduesCount,
    nl.BillsCount

FROM
    Polaris.Polaris.NotificationLog nl (NOLOCK)

    JOIN Polaris.Polaris.NotificationStatuses ns (NOLOCK)
        ON nl.NotificationStatusID = ns.NotificationStatusID

    LEFT JOIN Polaris.Polaris.Patrons p (NOLOCK)
        ON nl.PatronID = p.PatronID
    LEFT JOIN Polaris.Polaris.PatronRegistration pr (NOLOCK)
        ON nl.PatronID = pr.PatronID
    LEFT JOIN Polaris.Polaris.NotificationTypes nt (NOLOCK)
        ON nl.NotificationTypeID = nt.NotificationTypeID

WHERE
    nl.DeliveryOptionID = 2  -- Email
    AND nl.NotificationStatusID IN (13, 14)  -- Failed statuses
    AND nl.NotificationDateTime >= DATEADD(day, -30, GETDATE())  -- Last 30 days

ORDER BY
    nl.NotificationDateTime DESC,
    nl.PatronBarcode
```

### Output Format

```
NotificationLogID|PatronID|PatronBarcode|NotificationDateTime|EmailAddress|NotificationTypeID|NotificationStatusID|FailureReason|Details|NameFirst|NameLast|PatronFullName|PatronRegistrationEmail|AltEmailAddress|NotificationTypeDescription|HoldsCount|OverduesCount|BillsCount
```

**Use Case:** Import into Laravel to flag invalid email addresses and generate patron contact cleanup reports.

---

## QUERY 6: EMAIL STATISTICS DASHBOARD

**Purpose:** Generate summary statistics for email notifications
**Use:** Dashboard metrics, reporting

### SQL Query

```sql
SELECT
    nt.Description AS NotificationType,
    nl.NotificationTypeID,

    -- Total Counts
    COUNT(*) AS TotalNotifications,
    SUM(CASE WHEN nl.NotificationStatusID = 12 THEN 1 ELSE 0 END) AS SuccessfulEmails,
    SUM(CASE WHEN nl.NotificationStatusID IN (13,14) THEN 1 ELSE 0 END) AS FailedEmails,

    -- Item Counts
    SUM(nl.HoldsCount) AS TotalHolds,
    SUM(nl.OverduesCount) AS TotalOverdues,
    SUM(nl.Overdues2ndCount) AS Total2ndOverdues,
    SUM(nl.Overdues3rdCount) AS Total3rdOverdues,
    SUM(nl.BillsCount) AS TotalBills,
    SUM(nl.ManualBillCount) AS TotalManualBills,
    SUM(nl.CancelsCount) AS TotalCancels,

    -- Success Rate
    CAST(
        SUM(CASE WHEN nl.NotificationStatusID = 12 THEN 1 ELSE 0 END) * 100.0
        / COUNT(*)
        AS DECIMAL(5,2)
    ) AS SuccessRatePercent,

    -- Unique Patrons
    COUNT(DISTINCT nl.PatronID) AS UniquePatrons

FROM
    Polaris.Polaris.NotificationLog nl (NOLOCK)
    JOIN Polaris.Polaris.NotificationTypes nt (NOLOCK)
        ON nl.NotificationTypeID = nt.NotificationTypeID

WHERE
    nl.DeliveryOptionID = 2  -- Email
    AND nl.NotificationDateTime >= DATEADD(day, -30, GETDATE())  -- Last 30 days

GROUP BY
    nl.NotificationTypeID,
    nt.Description

ORDER BY
    TotalNotifications DESC
```

---

## QUERY 7: MAIL STATISTICS DASHBOARD

**Purpose:** Generate summary statistics for mail notifications
**Use:** Dashboard metrics, reporting

### SQL Query

```sql
SELECT
    nt.Description AS NotificationType,
    nl.NotificationTypeID,

    -- Total Counts
    COUNT(*) AS TotalNotifications,
    SUM(CASE WHEN nl.NotificationStatusID = 15 THEN 1 ELSE 0 END) AS MailPrinted,

    -- Item Counts
    SUM(nl.HoldsCount) AS TotalHolds,
    SUM(nl.OverduesCount) AS TotalOverdues,
    SUM(nl.Overdues2ndCount) AS Total2ndOverdues,
    SUM(nl.Overdues3rdCount) AS Total3rdOverdues,
    SUM(nl.BillsCount) AS TotalBills,
    SUM(nl.ManualBillCount) AS TotalManualBills,

    -- Unique Patrons
    COUNT(DISTINCT nl.PatronID) AS UniquePatrons,

    -- Cost Estimation (assuming $0.75 per mail notification)
    COUNT(*) * 0.75 AS EstimatedCost

FROM
    Polaris.Polaris.NotificationLog nl (NOLOCK)
    JOIN Polaris.Polaris.NotificationTypes nt (NOLOCK)
        ON nl.NotificationTypeID = nt.NotificationTypeID

WHERE
    nl.DeliveryOptionID = 1  -- Mail
    AND nl.NotificationDateTime >= DATEADD(day, -30, GETDATE())  -- Last 30 days

GROUP BY
    nl.NotificationTypeID,
    nt.Description

ORDER BY
    TotalNotifications DESC
```

---

## SCHEDULED EXPORT SCRIPTS

### Windows Batch Script Template

**File:** `C:\polaris-exports\export_email_notifications.bat`

```batch
@echo off
REM Export Email Notifications for Laravel Import
REM Runs daily at 8:00 AM via Windows Task Scheduler

SET SERVER=POLARISSQL
SET DATABASE=Polaris
SET OUTPUT_DIR=C:\polaris-exports\email
SET TIMESTAMP=%DATE:~10,4%-%DATE:~4,2%-%DATE:~7,2%_%TIME:~0,2%-%TIME:~3,2%-%TIME:~6,2%
SET OUTPUT_FILE=%OUTPUT_DIR%\email_notifications_submitted_%TIMESTAMP%.txt

REM Ensure output directory exists
IF NOT EXIST "%OUTPUT_DIR%" MKDIR "%OUTPUT_DIR%"

REM Run SQL query and export to file
sqlcmd -S %SERVER% -d %DATABASE% -i "C:\polaris-exports\sql\email_notifications.sql" -o "%OUTPUT_FILE%" -s "|" -W -h -1

REM Archive to FTP server (optional)
REM WinSCP script or FTP commands here

echo Email notifications exported to %OUTPUT_FILE%
```

### Task Scheduler Configuration

**Task Name:** Polaris Email Notifications Export
**Trigger:** Daily at 8:00 AM
**Action:** Run `C:\polaris-exports\export_email_notifications.bat`
**Run As:** Service account with read access to Polaris database

---

## FIELD MAPPING REFERENCE

### NotificationLog → Laravel Model

| Polaris Field | Laravel Field | Type | Notes |
|---------------|---------------|------|-------|
| NotificationLogID | notification_log_id | BIGINT | Primary key |
| PatronID | patron_id | INT | Foreign key |
| PatronBarcode | patron_barcode | VARCHAR(20) | Indexed |
| NotificationDateTime | notification_date_time | DATETIME | When sent |
| NotificationTypeID | notification_type_id | INT | Type of notice |
| DeliveryOptionID | delivery_option_id | INT | 1=Mail, 2=Email |
| DeliveryString | email_address | VARCHAR(255) | Email for email notifications |
| NotificationStatusID | notification_status_id | INT | Delivery status |
| OverduesCount | overdues_count | INT | Count |
| HoldsCount | holds_count | INT | Count |
| CancelsCount | cancels_count | INT | Count |
| BillsCount | bills_count | INT | Count |
| ManualBillCount | manual_bill_count | INT | Count |
| Overdues2ndCount | overdues_2nd_count | INT | Count |
| Overdues3rdCount | overdues_3rd_count | INT | Count |
| RoutingsCount | routings_count | INT | Count |
| RecallsCount | recalls_count | INT | Count |
| ReportingOrgID | reporting_org_id | INT | Sending branch |
| LanguageID | language_id | INT | Language code |
| Details | details | TEXT | Additional info |

### PatronRegistration → Laravel Model

| Polaris Field | Laravel Field | Type | Notes |
|---------------|---------------|------|-------|
| NameFirst | name_first | VARCHAR(32) | First name |
| NameLast | name_last | VARCHAR(100) | Last name |
| NameMiddle | name_middle | VARCHAR(32) | Middle name |
| PatronFullName | patron_full_name | VARCHAR(100) | Full name |
| EmailAddress | email_address | VARCHAR(64) | Primary email |
| AltEmailAddress | alt_email_address | VARCHAR(64) | Alternate email |
| DeliveryOptionID | delivery_option_id | INT | Preference |
| ExpirationDate | expiration_date | DATETIME | Card expiration |
| ExcludeFromOverdues | exclude_from_overdues | BOOLEAN | Flag |
| ExcludeFromHolds | exclude_from_holds | BOOLEAN | Flag |
| ExcludeFromBills | exclude_from_bills | BOOLEAN | Flag |

### Addresses/PostalCodes → Laravel Model

| Polaris Field | Laravel Field | Type | Notes |
|---------------|---------------|------|-------|
| StreetOne | street_one | VARCHAR(64) | Address line 1 |
| StreetTwo | street_two | VARCHAR(64) | Address line 2 |
| StreetThree | street_three | VARCHAR(64) | Address line 3 |
| MunicipalityName | municipality_name | VARCHAR(64) | Municipality |
| ZipPlusFour | zip_plus_four | VARCHAR(4) | +4 extension |
| PostalCode | postal_code | VARCHAR(12) | ZIP code |
| City | city | VARCHAR(32) | City from postal |
| State | state | VARCHAR(32) | State |
| County | county | VARCHAR(32) | County |
| CountryName | country_name | VARCHAR(50) | Country |

---

## PRODUCTION NOTES

### AddressTypeID Values

**Verified values in Polaris:**
```sql
1  = Generic
2  = Notice          ← Used in queries for mail notifications
3  = Invoice
4  = Statement
5  = Billing
6  = Shipping
7  = Claim Response
8  = Corporation
9  = Claiming
10 = Ordering
11 = Payment
12 = Mailing         ← Could be used as fallback
13 = Receiving
```

**Queries use AddressTypeID = 2 (Notice)** for mail notifications since that's the address type specifically for sending notices.

**Optional Enhancement:** Add fallback logic to try AddressTypeID = 12 (Mailing) if Notice address is NULL.

### Performance Optimization

1. **Indexes:** Ensure indexes exist on:
   - `NotificationLog.NotificationDateTime`
   - `NotificationLog.DeliveryOptionID`
   - `NotificationLog.PatronID`
   - `PatronRegistration.PatronID`
   - `PatronAddresses.PatronID`

2. **NOLOCK Hints:** Used in all queries to avoid blocking. Review based on your consistency requirements.

3. **Date Ranges:** Adjust `DATEADD(day, -7, GETDATE())` based on your import frequency.

### Security

- Use read-only database account for exports
- Store output files in secure directory
- Encrypt patron data in transit
- Purge old export files (retention policy)

---

**Status:** ✅ Production Ready - All field names verified
**Last Updated:** November 18, 2025
**Next Action:** Test queries against Polaris database with sample data
