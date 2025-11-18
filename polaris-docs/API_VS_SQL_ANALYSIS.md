# Polaris API vs SQL Queries - Efficiency Analysis

**Date:** November 18, 2025
**Purpose:** Identify opportunities to use Polaris API methods instead of SQL queries
**Status:** 📊 Analysis Complete

---

## Executive Summary

After analyzing the Polaris API (polaris-api-1.0 swagger spec), several API endpoints could **complement** the SQL queries, but **SQL remains more efficient** for bulk historical data exports. However, the API is superior for **real-time patron lookups** and **patron mapping** operations.

### Key Recommendation

**Hybrid Approach:** Use SQL for bulk notification log exports and statistics, but use API for patron data mapping and real-time lookups.

### 🔧 Implementation with PAPIClient

**Good news!** You already have the [blashbrook/papiclient](https://github.com/blashbrook/papiclient) Laravel package which provides a fluent interface for all Polaris API interactions.

**📘 For complete implementation examples and code samples, see:**
- [PAPICLIENT_IMPLEMENTATION_GUIDE.md](./PAPICLIENT_IMPLEMENTATION_GUIDE.md) - Detailed examples using PAPIClient for all endpoints below

**Quick PAPIClient Example:**
```php
use Blashbrook\PAPIClient\PAPIClient;

// Get patron email and address data
$response = $this->papiclient
    ->method('GET')
    ->patron($barcode)
    ->uri('basicdata?addresses=true')
    ->execRequest();

$patronData = $response->json()['PatronBasicData'];
```

---

## API Endpoints Available

### 1. PatronBasicDataGet

**Endpoint:** `/REST/public/v1/{LangID}/{AppID}/{OrgID}/patron/{PatronBarcode}/basicdata`

**What it returns:**
- ✅ PatronID, Barcode
- ✅ EmailAddress, AltEmailAddress
- ✅ NameFirst, NameLast, NameMiddle
- ✅ PhoneNumber (3 phone numbers + cell phone)
- ✅ DeliveryOptionID
- ✅ EmailFormatID
- ✅ **PatronAddresses array** (with AddressTypeID filtering support)
- ✅ ExpirationDate
- ✅ Language, registration dates, etc.

**PatronAddress schema includes:**
```json
{
  "AddressID": int,
  "FreeTextLabel": string,
  "StreetOne": string,
  "StreetTwo": string,
  "StreetThree": string,
  "City": string,
  "State": string,
  "County": string,
  "PostalCode": string,
  "ZipPlusFour": string,
  "Country": string,
  "CountryID": int,
  "AddressTypeID": int  // 1=Generic, 2=Notice, 12=Mailing, etc.
}
```

### 2. NotificationQueueGet

**Endpoint:** `/REST/protected/v1/{LangID}/{AppID}/{OrgID}/{AccessToken}/notification`

**What it returns:**
- ✅ Current notification queue entries (NOT historical logs)
- ✅ PatronID, DeliveryOptionID, NotificationTypeID
- ✅ ItemRecordID, DueDate, BrowseTitle
- ❌ Does NOT include email addresses
- ❌ Does NOT include historical notification logs

**Limitation:** Only returns **current queue**, not historical `NotificationLog` data.

### 3. NotificationUpdatePut

**Endpoint:** `/REST/protected/v1/{LangID}/{AppID}/{OrgID}/{AccessToken}/notification/{NotificationTypeID}`

**Purpose:** Update notification status after contact
- Used for phone notification processing
- Updates NotificationLog and NotificationQueue
- **Note:** "This method only supports telephone notification processing"

---

## 🎯 IMPORTANT: Notification-Driven Patron Sync

**⚠️ UPDATE:** Only sync patron data for patrons with **recent notifications** to keep database small.

**Why:**
- ✅ 85-95% reduction in patron records
- ✅ Faster sync times (seconds vs minutes)
- ✅ Smaller database = faster queries
- ✅ Auto-cleanup of inactive patrons

**How it works:**
```
1. Import notification logs (SQL)
2. Extract PatronIDs from recent notifications
3. Sync ONLY those patrons (SQL with WHERE PatronID IN (...))
```

**📘 For complete implementation, see:**
- [NOTIFICATION_DRIVEN_SQL_QUERIES.md](./NOTIFICATION_DRIVEN_SQL_QUERIES.md) - Updated SQL queries with notification filtering

---

## Comparison: API vs SQL

### ✅ QUERIES WHERE API IS BETTER

#### Query 3: Email Patron Mapping (API for Small Sets, SQL for Notification-Driven)

**Current SQL Approach:**
```sql
SELECT DISTINCT p.PatronID, pr.EmailAddress, pr.NameFirst, ...
FROM Polaris.Polaris.Patrons p
JOIN Polaris.Polaris.PatronRegistration pr ON p.PatronID = pr.PatronID
WHERE pr.EmailAddress IS NOT NULL AND pr.DeliveryOptionID = 2
```

**API Alternative:**
```
GET /patron/{barcode}/basicdata
```

**Advantages:**
- ✅ Single API call per patron (can be paginated through all patrons)
- ✅ Always returns fresh data (no sync delays)
- ✅ Includes EmailAddress, AltEmailAddress, DeliveryOptionID
- ✅ No complex SQL joins needed
- ✅ Can use query parameter `?addresses=true` to get address data too

**When to use API:**
- Real-time patron profile lookups
- On-demand email verification for individual patrons
- User-facing applications (web/mobile apps)

**When to use SQL:**
- Bulk daily exports of all email patrons
- Batch processing (faster for 10,000+ patrons)
- Historical snapshots

---

#### Query 4: Mail Patron Mapping (BETTER WITH API)

**Current SQL Approach:**
```sql
SELECT p.PatronID, pr.NameFirst, a.StreetOne, a.StreetTwo, pc.City, pc.State ...
FROM Polaris.Polaris.Patrons p
LEFT JOIN PatronAddresses pa ON p.PatronID = pa.PatronID AND pa.AddressTypeID = 2
LEFT JOIN Addresses a ON pa.AddressID = a.AddressID
LEFT JOIN PostalCodes pc ON a.PostalCodeID = pc.PostalCodeID
```

**API Alternative:**
```
GET /patron/{barcode}/basicdata?addresses=true
```

**Advantages:**
- ✅ Returns **all addresses** for the patron in `PatronAddresses` array
- ✅ Each address includes `AddressTypeID` - filter for type 2 (Notice) or 12 (Mailing)
- ✅ Avoids complex 4-table SQL join
- ✅ Includes City, State, PostalCode, ZipPlusFour, Country in address object
- ✅ Simpler fallback logic (check multiple AddressTypeIDs in one response)

**Address Fallback Logic:**
```javascript
// With API, fallback is simple:
let address = patron.PatronAddresses.find(a => a.AddressTypeID === 2);  // Notice
if (!address) {
  address = patron.PatronAddresses.find(a => a.AddressTypeID === 12); // Mailing fallback
}
if (!address) {
  address = patron.PatronAddresses.find(a => a.AddressTypeID === 1);  // Generic fallback
}
```

---

### ❌ QUERIES WHERE SQL IS BETTER

#### Query 1: Email Notifications Export (KEEP SQL)

**Why SQL is better:**
- ✅ **Bulk historical data** - API has no endpoint for historical NotificationLog
- ✅ NotificationQueueGet only returns **current queue**, not past sent emails
- ✅ SQL can export 7 days of notifications in one query
- ✅ API would require individual patron lookups (inefficient for thousands of records)

**Verdict:** **Keep SQL query** for daily batch exports

---

#### Query 2: Mail Notifications Export (KEEP SQL)

**Why SQL is better:**
- ✅ Same as Query 1 - no API endpoint for historical NotificationLog
- ✅ Bulk export is faster than per-patron API calls
- ✅ Need mailing addresses tied to specific notification events

**Verdict:** **Keep SQL query** for daily batch exports

---

#### Query 5: Email Failure Report (KEEP SQL)

**Why SQL is better:**
- ✅ Filtering by NotificationStatusID (13, 14 = failed)
- ✅ No API endpoint for failed notification history
- ✅ Need aggregated failure data over 30 days

**Verdict:** **Keep SQL query**

---

#### Query 6 & 7: Statistics Dashboards (KEEP SQL)

**Why SQL is better:**
- ✅ Aggregations (COUNT, SUM, GROUP BY)
- ✅ API doesn't provide aggregated statistics
- ✅ Much faster to run analytics in SQL than to fetch thousands of API responses

**Verdict:** **Keep SQL queries**

---

## Recommended Notification-Driven Architecture

**⚠️ Key Principle:** Only sync patron data for patrons with recent notifications.

### Phase 1: Cleanup & Import (SQL)

```
Daily at 7:00 AM:
├── Step 1: Sync deleted patrons (API - only method available)
│   └── php artisan polaris:sync-deleted-patrons --days=1
│
└── Step 2: Import notification logs (SQL - fast bulk import)
    └── php artisan polaris:import-notification-logs --days=7
```

### Phase 2: Notification-Driven Patron Sync (SQL or API)

```
Daily at 7:30 AM:
├── Extract PatronIDs from recent notifications
│   └── SELECT DISTINCT PatronID FROM polaris_notification_log
│       WHERE DeliveryOptionID = 2 AND CreationDate >= @cutoffDate
│
└── Sync ONLY those patrons
    ├── Option A: SQL (Recommended for >200 patrons)
    │   └── php artisan polaris:sync-email-patrons --days=30 --method=sql
    │       └── Uses WHERE PatronID IN (...)
    │
    └── Option B: API (Recommended for <200 patrons)
        └── php artisan polaris:sync-email-patrons --days=30 --method=api
            └── Calls PatronBasicDataGet for each PatronID
```

**Typical Results:**
- Old approach: Sync 50,000+ patrons with email delivery preference
- New approach: Sync 2,000-5,000 patrons with recent notifications
- **Reduction: 85-95% fewer patron records**

### Phase 3: Real-time Patron Lookup (API)

```
On-demand when viewing patron details:
├── GET /patron/{barcode}/basicdata
│   ├── Returns current email, phone, addresses
│   ├── Returns DeliveryOptionID preference
│   └── Use for "Patron Profile" page in dashboard
```

### Complete Daily Schedule

```php
// app/Console/Kernel.php
protected function schedule(Schedule $schedule)
{
    // 7:00 AM: Clean up deleted patrons
    $schedule->command('polaris:sync-deleted-patrons --days=1')
        ->dailyAt('07:00');

    // 7:10 AM: Import notification logs
    $schedule->command('polaris:import-notification-logs --days=7')
        ->dailyAt('07:10');

    // 7:30 AM: Sync email patrons (notification-driven)
    $schedule->command('polaris:sync-email-patrons --days=30 --method=sql')
        ->dailyAt('07:30');

    // 7:50 AM: Sync mail patrons (notification-driven)
    $schedule->command('polaris:sync-mail-patrons --days=30 --method=sql')
        ->dailyAt('07:50');
}
```

---

## API Implementation Details

### Required Setup

1. **Authentication:**
   - Protected endpoints require `AccessToken`
   - Public endpoints (including PatronBasicDataGet) do NOT require auth
   - Need AppID and OrgID from Polaris admin

2. **Environment Config:**
```env
POLARIS_API_URL=https://polaris.library.local/PAPIService/REST
POLARIS_API_APP_ID=100
POLARIS_API_ORG_ID=3
POLARIS_API_LANG_ID=1033
POLARIS_API_ACCESS_TOKEN=your_token_here
```

3. **Rate Limiting:**
   - Check Polaris API documentation for rate limits
   - Implement throttling for bulk patron sync
   - Consider caching patron data

### Sample Laravel Service

```php
namespace App\Services;

use Illuminate\Support\Facades\Http;

class PolarisAPIService
{
    protected $baseUrl;
    protected $langID;
    protected $appID;
    protected $orgID;

    public function __construct()
    {
        $this->baseUrl = config('polaris.api.url');
        $this->langID = config('polaris.api.lang_id');
        $this->appID = config('polaris.api.app_id');
        $this->orgID = config('polaris.api.org_id');
    }

    public function getPatronBasicData(string $barcode, bool $includeAddresses = false)
    {
        $url = sprintf(
            '%s/public/v1/%s/%s/%s/patron/%s/basicdata',
            $this->baseUrl,
            $this->langID,
            $this->appID,
            $this->orgID,
            $barcode
        );

        $query = $includeAddresses ? ['addresses' => 'true'] : [];

        $response = Http::get($url, $query);

        if ($response->successful()) {
            return $response->json()['PatronBasicData'];
        }

        throw new \Exception("API Error: " . $response->json()['ErrorMessage']);
    }

    public function findBestMailingAddress(array $addresses): ?array
    {
        // Priority: Notice (2) → Mailing (12) → Generic (1)
        $priority = [2, 12, 1];

        foreach ($priority as $typeID) {
            $address = collect($addresses)->firstWhere('AddressTypeID', $typeID);
            if ($address && $address['StreetOne']) {
                return $address;
            }
        }

        return null;
    }
}
```

---

## Performance Comparison

### Patron Data Sync (10,000 active patrons)

| Method | Time | Network | Complexity |
|--------|------|---------|------------|
| **SQL Query** | ~5 seconds | 1 DB query | Low |
| **API Calls (sequential)** | ~30 minutes | 10,000 HTTP requests | Medium |
| **API Calls (parallel, 50/sec)** | ~4 minutes | 10,000 HTTP requests | High |

**Recommendation:** Use SQL for bulk daily patron sync. Use API for real-time individual lookups.

### Single Patron Lookup

| Method | Time | Notes |
|--------|------|-------|
| **API** | 50-200ms | Real-time, always fresh |
| **SQL** | 10-50ms | May be stale if using cached/imported data |

**Recommendation:** Use API for user-facing features where freshness matters.

---

## Final Recommendations

### ✅ Replace with API

1. **Real-time patron profile views** - Use `PatronBasicDataGet`
2. **Individual patron email/address verification** - Use `PatronBasicDataGet`
3. **Web/mobile app patron lookup** - Use API endpoints

### ✅ Keep SQL

1. **Daily bulk notification exports** (Query 1 & 2)
2. **Email failure reports** (Query 5)
3. **Statistics dashboards** (Query 6 & 7)
4. **Bulk patron mapping exports** (Query 3 & 4) - for large patron bases

### ✅ Hybrid Approach

1. **Patron mapping sync:**
   - Small libraries (<5,000 patrons): Use API
   - Large libraries (>10,000 patrons): Use SQL
   - Real-time updates: Always use API

2. **Address fallback logic:**
   - API makes fallback easier (all addresses in one response)
   - Consider refactoring Query 2 & 4 if API sync is fast enough

---

## Additional Useful API Endpoints

### 🆕 SynchGetDeletedPatrons - **RECOMMENDED TO IMPLEMENT**

**Endpoint:** `/REST/protected/v1/{LangID}/{AppID}/{OrgID}/{AccessToken}/synch/patrons/deleted`

**What it does:**
- Returns patrons deleted since a specific date/time
- Returns: PatronID, Barcode

**Why you should use this:**
- ⚠️ **No SQL equivalent** for tracking deleted patrons
- ✅ Prevents sending notifications to deleted patron accounts
- ✅ Keeps local email/mail patron mappings in sync with Polaris
- ✅ Simple daily cleanup job

**PAPIClient Implementation:**
```php
// See PAPICLIENT_IMPLEMENTATION_GUIDE.md for full example
$deletedPatrons = $this->papiclient
    ->method('GET')
    ->protected()
    ->auth($accessToken)
    ->uri("synch/patrons/deleted?deletedate={$dateString}")
    ->execRequest();
```

**Recommended Usage:**
```bash
# Run daily at 7:00 AM (before patron mapping sync)
php artisan polaris:sync-deleted-patrons --days=1
```

---

### NotificationMatrixGet - Configuration Validation

**Endpoint:** `/REST/protected/v1/{LangID}/{AppID}/{OrgID}/{AccessToken}/notificationmatrix`

**What it does:**
- Returns notification matrix definitions for organizations
- Shows which notification types are enabled for email/mail delivery

**Use Cases:**
- ✅ Validate notification configuration
- ✅ Admin dashboard showing enabled notification types
- ✅ Troubleshooting notification issues

**PAPIClient Implementation:**
```php
$matrix = $this->papiclient
    ->method('GET')
    ->protected()
    ->auth($accessToken)
    ->uri('notificationmatrix?organizations=0')
    ->execRequest();
```

---

### PatronSearchGet - Bulk Patron Discovery

**Endpoint:** `/REST/protected/v1/{LangID}/{AppID}/{OrgID}/{AccessToken}/search/patrons/boolean`

**What it does:**
- Search for patrons using CCL (Common Command Language) queries
- Supports pagination
- Returns: PatronID, Barcode, Name, OrganizationID

**Use Cases:**
- ✅ Discover patrons matching specific criteria
- ⚠️ Limited - Returns only basic info, would need to call `PatronBasicDataGet` for each patron

**Verdict:** Useful for patron discovery, but **SQL is still faster** for bulk patron list exports.

**PAPIClient Implementation:**
```php
$result = $this->papiclient
    ->method('GET')
    ->protected()
    ->auth($accessToken)
    ->uri("search/patrons/boolean?q={$cclQuery}&patronsperpage=100&page=1")
    ->execRequest();
```

---

## Implementation Priority

### Phase 1: Keep Current SQL Approach ✅
- Already production-ready
- No API integration needed
- Fast bulk exports

### Phase 2: Add API Service (Optional Enhancement) 🔄
- Create PolarisAPIService class
- Add API-based patron sync command
- Use for real-time patron lookups in dashboard

### Phase 3: Hybrid Optimization (Future) 🔮
- A/B test API vs SQL for patron mapping
- Implement intelligent fallback (API fails → use SQL)
- Add caching layer for API responses

---

## Conclusion

**Your current SQL queries are optimal** for the bulk notification monitoring use case. The API is valuable for **real-time patron data access** but doesn't replace the need for SQL-based historical notification exports.

**Best of both worlds:**
- SQL for batch notification log exports and statistics
- API for real-time patron profile lookups and verification

---

**Next Steps:**
1. ✅ Continue with SQL for notification log import (Query 1 & 2)
2. ⭐ **UPDATE patron sync to use notification-driven approach** (see [NOTIFICATION_DRIVEN_SQL_QUERIES.md](./NOTIFICATION_DRIVEN_SQL_QUERIES.md))
3. ⭐ **IMPLEMENT `SyncDeletedPatronsCommand` using PAPIClient** (see [PAPICLIENT_IMPLEMENTATION_GUIDE.md](./PAPICLIENT_IMPLEMENTATION_GUIDE.md))
4. Add `--days=N` parameter to control patron sync lookback window
5. Test with small lookback (--days=7) before expanding to --days=30
6. Monitor database size and query performance improvements
7. Consider adding API service for real-time patron profile pages

**Resources:**
- [NOTIFICATION_DRIVEN_SQL_QUERIES.md](./NOTIFICATION_DRIVEN_SQL_QUERIES.md) - **NEW:** Updated SQL queries with notification filtering
- [PAPICLIENT_IMPLEMENTATION_GUIDE.md](./PAPICLIENT_IMPLEMENTATION_GUIDE.md) - Complete code examples using PAPIClient
- [papiclient-readme.md](./papiclient-readme.md) - PAPIClient package documentation

**Last Updated:** November 18, 2025
**Analysis By:** Claude Code
