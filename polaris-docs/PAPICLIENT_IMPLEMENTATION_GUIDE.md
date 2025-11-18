# PAPIClient Implementation Guide for Email/Mail Notices

**Date:** November 18, 2025
**Purpose:** Practical examples using PAPIClient for Polaris notification management
**Package:** [blashbrook/papiclient](https://github.com/blashbrook/papiclient)

---

## Table of Contents

1. [Overview](#overview)
2. [Installation & Setup](#installation--setup)
3. [Patron Data Retrieval](#patron-data-retrieval)
4. [Deleted Patron Sync](#deleted-patron-sync)
5. [Notification Configuration](#notification-configuration)
6. [Patron Search & Discovery](#patron-search--discovery)
7. [Complete Implementation Examples](#complete-implementation-examples)
8. [Performance Considerations](#performance-considerations)

---

## Overview

This guide demonstrates how to use the PAPIClient Laravel package to interact with Polaris API endpoints for email and mail notification management. All examples use the fluent API interface provided by PAPIClient.

### PAPIClient Fluent API Syntax

```php
$response = $this->papiclient
    ->method('GET')              // HTTP method: GET, POST, PUT, DELETE
    ->protected()                // Use protected endpoint (requires auth)
    ->patron('BARCODE')          // Insert patron barcode in URI
    ->uri('endpoint')            // API endpoint path
    ->params(['key' => 'value']) // Request parameters
    ->auth('AccessToken')        // Authentication token
    ->execRequest();             // Execute the request
```

---

## Installation & Setup

### 1. Install PAPIClient

```bash
composer require blashbrook/papiclient
```

### 2. Configure Environment Variables

Add to your `.env` file:

```bash
# PAPI Credentials
PAPI_ACCESS_ID=your_access_id
PAPI_ACCESS_KEY=your_access_key
PAPI_BASE_URL=https://catalog.yourlibrary.org/PAPIService/REST

# PAPI Configuration
PAPI_LANGID=1033
PAPI_APPID=100
PAPI_ORGID=3
PAPI_VERSION=v1

# Staff Authentication (for protected endpoints)
PAPI_DOMAIN=YOURDOMAIN
PAPI_STAFF=staff_username
PAPI_PASSWORD="staff_password"
PAPI_LOGONBRANCHID=1
PAPI_LOGONUSERID=1
PAPI_LOGONWORKSTATIONID=1
```

### 3. Inject PAPIClient in Your Classes

```php
use Blashbrook\PAPIClient\PAPIClient;

class NotificationService
{
    protected PAPIClient $papiclient;

    public function __construct(PAPIClient $papiclient)
    {
        $this->papiclient = $papiclient;
    }
}
```

---

## Patron Data Retrieval

### PatronBasicDataGet - Get Patron Email & Address

**Endpoint:** `/REST/public/v1/{LangID}/{AppID}/{OrgID}/patron/{PatronBarcode}/basicdata`

**Use Case:** Retrieve patron email addresses and mailing addresses for notification mapping.

#### Basic Example

```php
/**
 * Get patron basic data including email and addresses
 *
 * @param string $barcode Patron barcode
 * @return object|null Patron data or null on error
 */
public function getPatronBasicData(string $barcode): ?object
{
    try {
        $response = $this->papiclient
            ->method('GET')
            ->patron($barcode)
            ->uri('basicdata?addresses=true')  // Include address data
            ->execRequest();

        if ($response->successful()) {
            return $response->json()['PatronBasicData'] ?? null;
        }

        Log::error('PatronBasicDataGet failed', [
            'barcode' => $barcode,
            'error' => $response->json()['ErrorMessage'] ?? 'Unknown error'
        ]);

        return null;

    } catch (\Exception $e) {
        Log::error('PatronBasicDataGet exception', [
            'barcode' => $barcode,
            'exception' => $e->getMessage()
        ]);
        return null;
    }
}
```

#### Extract Email Information

```php
/**
 * Get patron email addresses and delivery preference
 *
 * @param string $barcode Patron barcode
 * @return array Email data
 */
public function getPatronEmailData(string $barcode): array
{
    $patronData = $this->getPatronBasicData($barcode);

    if (!$patronData) {
        return [
            'PatronID' => null,
            'EmailAddress' => null,
            'AltEmailAddress' => null,
            'DeliveryOptionID' => null,
            'EmailFormatID' => null,
        ];
    }

    return [
        'PatronID' => $patronData->PatronID,
        'Barcode' => $patronData->Barcode,
        'EmailAddress' => $patronData->EmailAddress,
        'AltEmailAddress' => $patronData->AltEmailAddress,
        'DeliveryOptionID' => $patronData->DeliveryOptionID,
        'EmailFormatID' => $patronData->EmailFormatID,
        'NameFirst' => $patronData->NameFirst,
        'NameLast' => $patronData->NameLast,
    ];
}
```

#### Extract Mailing Address with Fallback

```php
/**
 * Get patron mailing address with fallback logic
 * Priority: Notice Address (2) → Mailing Address (12) → Generic Address (1)
 *
 * @param string $barcode Patron barcode
 * @return array|null Address data or null if no valid address
 */
public function getPatronMailingAddress(string $barcode): ?array
{
    $patronData = $this->getPatronBasicData($barcode);

    if (!$patronData || empty($patronData->PatronAddresses)) {
        return null;
    }

    // Priority order for address types
    $addressTypePriority = [2, 12, 1]; // Notice, Mailing, Generic

    foreach ($addressTypePriority as $typeID) {
        foreach ($patronData->PatronAddresses as $address) {
            if ($address->AddressTypeID === $typeID && !empty($address->StreetOne)) {
                return [
                    'PatronID' => $patronData->PatronID,
                    'Barcode' => $patronData->Barcode,
                    'NameFirst' => $patronData->NameFirst,
                    'NameLast' => $patronData->NameLast,
                    'AddressID' => $address->AddressID,
                    'AddressTypeID' => $address->AddressTypeID,
                    'StreetOne' => $address->StreetOne,
                    'StreetTwo' => $address->StreetTwo,
                    'StreetThree' => $address->StreetThree,
                    'City' => $address->City,
                    'State' => $address->State,
                    'PostalCode' => $address->PostalCode,
                    'ZipPlusFour' => $address->ZipPlusFour,
                    'County' => $address->County,
                    'Country' => $address->Country,
                ];
            }
        }
    }

    return null; // No valid address found
}
```

---

## Deleted Patron Sync

### SynchGetDeletedPatrons - Track Deleted Patrons

**Endpoint:** `/REST/protected/v1/{LangID}/{AppID}/{OrgID}/{AccessToken}/synch/patrons/deleted`

**Use Case:** Clean up local patron mappings when patrons are deleted from Polaris.

**⚠️ Important:** This is a **protected** endpoint requiring staff authentication.

#### Get Deleted Patrons Since Date

```php
/**
 * Get patrons deleted since a specific date
 *
 * @param Carbon|string $sinceDate Date to check from
 * @param string $accessToken Staff access token
 * @return array Array of deleted patron objects
 */
public function getDeletedPatrons($sinceDate, string $accessToken): array
{
    try {
        // Convert to ISO 8601 format
        $dateString = $sinceDate instanceof \Carbon\Carbon
            ? $sinceDate->toIso8601String()
            : \Carbon\Carbon::parse($sinceDate)->toIso8601String();

        $response = $this->papiclient
            ->method('GET')
            ->protected()
            ->auth($accessToken)
            ->uri("synch/patrons/deleted?deletedate={$dateString}")
            ->execRequest();

        if ($response->successful()) {
            $data = $response->json();
            return $data['PatronBarcodesAndPatronIDsRows'] ?? [];
        }

        Log::error('SynchGetDeletedPatrons failed', [
            'since_date' => $dateString,
            'error' => $response->json()['ErrorMessage'] ?? 'Unknown error'
        ]);

        return [];

    } catch (\Exception $e) {
        Log::error('SynchGetDeletedPatrons exception', [
            'since_date' => $sinceDate,
            'exception' => $e->getMessage()
        ]);
        return [];
    }
}
```

#### Clean Up Deleted Patrons (Laravel Command)

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Services\PolarisAPIService;
use App\Models\EmailPatron;
use App\Models\MailPatron;
use Carbon\Carbon;

class SyncDeletedPatronsCommand extends Command
{
    protected $signature = 'polaris:sync-deleted-patrons
                            {--days=1 : Number of days to look back}';

    protected $description = 'Remove deleted patrons from local patron mappings';

    protected PolarisAPIService $polarisAPI;

    public function __construct(PolarisAPIService $polarisAPI)
    {
        parent::__construct();
        $this->polarisAPI = $polarisAPI;
    }

    public function handle()
    {
        $days = (int) $this->option('days');
        $sinceDate = Carbon::now()->subDays($days);

        $this->info("Checking for deleted patrons since {$sinceDate->toDateString()}...");

        // Get access token (you'll need to implement this based on your auth flow)
        $accessToken = $this->polarisAPI->getStaffAccessToken();

        if (!$accessToken) {
            $this->error('Failed to authenticate with Polaris API');
            return 1;
        }

        $deletedPatrons = $this->polarisAPI->getDeletedPatrons($sinceDate, $accessToken);

        if (empty($deletedPatrons)) {
            $this->info('No deleted patrons found.');
            return 0;
        }

        $this->info("Found " . count($deletedPatrons) . " deleted patrons.");

        $emailDeleted = 0;
        $mailDeleted = 0;

        foreach ($deletedPatrons as $patron) {
            $patronID = $patron['PatronID'];
            $barcode = $patron['Barcode'] ?? 'Unknown';

            // Delete from email patron mapping
            $emailCount = EmailPatron::where('PatronID', $patronID)->delete();
            $emailDeleted += $emailCount;

            // Delete from mail patron mapping
            $mailCount = MailPatron::where('PatronID', $patronID)->delete();
            $mailDeleted += $mailCount;

            if ($emailCount > 0 || $mailCount > 0) {
                $this->line("✓ Removed patron {$barcode} (ID: {$patronID})");
            }
        }

        $this->info("Cleanup complete:");
        $this->line("  - Email patron records removed: {$emailDeleted}");
        $this->line("  - Mail patron records removed: {$mailDeleted}");

        return 0;
    }
}
```

#### Schedule Daily Cleanup

Add to `app/Console/Kernel.php`:

```php
protected function schedule(Schedule $schedule)
{
    // Run deleted patron sync every day at 7:00 AM (before patron mapping sync)
    $schedule->command('polaris:sync-deleted-patrons --days=1')
        ->dailyAt('07:00')
        ->emailOutputOnFailure(config('polaris.admin_email'));
}
```

---

## Notification Configuration

### NotificationMatrixGet - Get Notification Settings

**Endpoint:** `/REST/protected/v1/{LangID}/{AppID}/{OrgID}/{AccessToken}/notificationmatrix`

**Use Case:** Validate notification configuration and display settings in admin dashboard.

**⚠️ Important:** This is a **protected** endpoint requiring staff authentication.

```php
/**
 * Get notification matrix configuration for all or specific organizations
 *
 * @param string $accessToken Staff access token
 * @param string|null $organizations Comma-separated org IDs (null = all)
 * @return array Notification matrix data
 */
public function getNotificationMatrix(string $accessToken, ?string $organizations = null): array
{
    try {
        $uri = 'notificationmatrix';

        if ($organizations !== null) {
            $uri .= "?organizations={$organizations}";
        } else {
            $uri .= "?organizations=0"; // 0 = all organizations
        }

        $response = $this->papiclient
            ->method('GET')
            ->protected()
            ->auth($accessToken)
            ->uri($uri)
            ->execRequest();

        if ($response->successful()) {
            return $response->json()['NotificationMatrixGetRows'] ?? [];
        }

        Log::error('NotificationMatrixGet failed', [
            'organizations' => $organizations,
            'error' => $response->json()['ErrorMessage'] ?? 'Unknown error'
        ]);

        return [];

    } catch (\Exception $e) {
        Log::error('NotificationMatrixGet exception', [
            'organizations' => $organizations,
            'exception' => $e->getMessage()
        ]);
        return [];
    }
}
```

#### Check if Email/Mail Notifications are Enabled

```php
/**
 * Check if email and mail notifications are properly configured
 *
 * @param string $accessToken Staff access token
 * @return array Configuration status
 */
public function checkNotificationConfiguration(string $accessToken): array
{
    $matrix = $this->getNotificationMatrix($accessToken);

    $config = [
        'email_enabled' => false,
        'mail_enabled' => false,
        'notification_types' => [],
    ];

    foreach ($matrix as $row) {
        $notificationType = $row['NotificationTypeID'] ?? null;
        $deliveryOption = $row['DeliveryOptionID'] ?? null;

        if ($deliveryOption === 2) { // Email
            $config['email_enabled'] = true;
            $config['notification_types'][] = $notificationType;
        }

        if ($deliveryOption === 1) { // Mail
            $config['mail_enabled'] = true;
            $config['notification_types'][] = $notificationType;
        }
    }

    $config['notification_types'] = array_unique($config['notification_types']);

    return $config;
}
```

---

## Patron Search & Discovery

### PatronSearchGet - Search Patrons by Criteria

**Endpoint:** `/REST/protected/v1/{LangID}/{AppID}/{OrgID}/{AccessToken}/search/patrons/boolean`

**Use Case:** Discover patrons matching specific criteria before fetching detailed data.

**⚠️ Important:** This is a **protected** endpoint requiring staff authentication.

```php
/**
 * Search for patrons using CCL query
 *
 * @param string $accessToken Staff access token
 * @param string $query CCL query string
 * @param int $patronsPerPage Number of results per page
 * @param int $page Page number
 * @return array Search results
 */
public function searchPatrons(
    string $accessToken,
    string $query,
    int $patronsPerPage = 100,
    int $page = 1
): array {
    try {
        $queryParams = http_build_query([
            'q' => $query,
            'patronsperpage' => $patronsPerPage,
            'page' => $page,
        ]);

        $response = $this->papiclient
            ->method('GET')
            ->protected()
            ->auth($accessToken)
            ->uri("search/patrons/boolean?{$queryParams}")
            ->execRequest();

        if ($response->successful()) {
            $data = $response->json();
            return [
                'total' => $data['TotalRecordsFound'] ?? 0,
                'patrons' => $data['PatronSearchRows'] ?? [],
            ];
        }

        Log::error('PatronSearchGet failed', [
            'query' => $query,
            'error' => $response->json()['ErrorMessage'] ?? 'Unknown error'
        ]);

        return ['total' => 0, 'patrons' => []];

    } catch (\Exception $e) {
        Log::error('PatronSearchGet exception', [
            'query' => $query,
            'exception' => $e->getMessage()
        ]);
        return ['total' => 0, 'patrons' => []];
    }
}
```

#### Example: Find All Active Email Patrons

```php
/**
 * Find all active patrons with email delivery preference
 * Note: CCL query syntax depends on Polaris ILS configuration
 * This is a conceptual example - test with your ILS
 *
 * @param string $accessToken Staff access token
 * @return array Patron barcodes
 */
public function findEmailPatrons(string $accessToken): array
{
    $allPatrons = [];
    $page = 1;
    $patronsPerPage = 100;

    do {
        // Note: This CCL query may need adjustment based on your Polaris setup
        $result = $this->searchPatrons(
            $accessToken,
            'DeliveryOptionID:2', // Assuming this field is searchable
            $patronsPerPage,
            $page
        );

        foreach ($result['patrons'] as $patron) {
            $allPatrons[] = [
                'PatronID' => $patron['PatronID'],
                'Barcode' => $patron['Barcode'],
                'Name' => $patron['PatronFirstLastName'],
            ];
        }

        $page++;
        $hasMore = count($result['patrons']) === $patronsPerPage;

    } while ($hasMore && $page <= 100); // Safety limit

    return $allPatrons;
}
```

---

## Complete Implementation Examples

### Example 1: Notification-Driven Patron Sync (RECOMMENDED APPROACH)

**⚠️ Important:** Only sync patron data for patrons who have **recent notifications**. This keeps your database smaller by tracking only active patrons.

#### Workflow:
1. Import notification logs first (SQL)
2. Extract unique PatronIDs from recent notifications
3. Sync patron data ONLY for those PatronIDs (API or SQL)

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Services\PolarisAPIService;
use App\Models\EmailPatron;
use Illuminate\Support\Facades\DB;
use Carbon\Carbon;

class SyncEmailPatronsCommand extends Command
{
    protected $signature = 'polaris:sync-email-patrons
                            {--days=30 : Only sync patrons with notifications in last N days}
                            {--method=api : Sync method: api or sql}';

    protected $description = 'Sync email patron data for patrons with recent notifications';

    protected PolarisAPIService $polarisAPI;

    public function __construct(PolarisAPIService $polarisAPI)
    {
        parent::__construct();
        $this->polarisAPI = $polarisAPI;
    }

    public function handle()
    {
        $days = (int) $this->option('days');
        $cutoffDate = Carbon::now()->subDays($days);

        $this->info("Syncing email patrons with notifications since {$cutoffDate->toDateString()}...");

        // STEP 1: Get PatronIDs from recent EMAIL notifications (DeliveryOptionID = 2)
        $patronIDs = DB::table('polaris_notification_log')
            ->where('DeliveryOptionID', 2) // Email notifications only
            ->where('CreationDate', '>=', $cutoffDate)
            ->distinct()
            ->pluck('PatronID')
            ->toArray();

        if (empty($patronIDs)) {
            $this->info('No email patrons with recent notifications found.');
            return 0;
        }

        $this->info("Found " . count($patronIDs) . " patrons with recent email notifications.");

        if ($this->option('method') === 'api') {
            return $this->syncViaAPI($patronIDs);
        } else {
            return $this->syncViaSQL($patronIDs);
        }
    }

    /**
     * Sync patron data via Polaris API (better for small patron sets)
     */
    protected function syncViaAPI(array $patronIDs): int
    {
        $this->info('Using API method...');

        // Get staff access token
        $accessToken = $this->polarisAPI->getStaffAccessToken();

        if (!$accessToken) {
            $this->error('Failed to authenticate with Polaris API');
            return 1;
        }

        // Get barcodes for these PatronIDs
        $patrons = DB::table('polaris_notification_log')
            ->whereIn('PatronID', $patronIDs)
            ->select('PatronID', 'Barcode')
            ->distinct()
            ->get();

        $bar = $this->output->createProgressBar(count($patrons));
        $synced = 0;
        $skipped = 0;
        $errors = 0;

        foreach ($patrons as $patron) {
            try {
                $emailData = $this->polarisAPI->getPatronEmailData($patron->Barcode);

                // Only store if patron still has email delivery preference
                if ($emailData['DeliveryOptionID'] === 2 && !empty($emailData['EmailAddress'])) {
                    EmailPatron::updateOrCreate(
                        ['PatronID' => $emailData['PatronID']],
                        [
                            'Barcode' => $emailData['Barcode'],
                            'EmailAddress' => $emailData['EmailAddress'],
                            'AltEmailAddress' => $emailData['AltEmailAddress'],
                            'NameFirst' => $emailData['NameFirst'],
                            'NameLast' => $emailData['NameLast'],
                            'EmailFormatID' => $emailData['EmailFormatID'],
                            'last_synced_at' => Carbon::now(),
                        ]
                    );
                    $synced++;
                } else {
                    $skipped++;
                }

            } catch (\Exception $e) {
                $this->error("Error processing PatronID {$patron->PatronID}: " . $e->getMessage());
                $errors++;
            }

            $bar->advance();
            usleep(50000); // Rate limiting: ~20 requests/second
        }

        $bar->finish();
        $this->newLine(2);

        $this->info("API sync complete:");
        $this->line("  - Synced: {$synced}");
        $this->line("  - Skipped: {$skipped}");
        $this->line("  - Errors: {$errors}");

        return 0;
    }

    /**
     * Sync patron data via SQL (better for large patron sets)
     */
    protected function syncViaSQL(array $patronIDs): int
    {
        $this->info('Using SQL method...');

        // Query Polaris database directly - MUCH faster for bulk operations
        $query = "
            SELECT DISTINCT
                p.PatronID,
                pr.Barcode,
                pr.EmailAddress,
                pr.AltEmailAddress,
                pr.NameFirst,
                pr.NameLast,
                pr.DeliveryOptionID,
                pr.EmailFormatID
            FROM Polaris.Polaris.Patrons p
            INNER JOIN Polaris.Polaris.PatronRegistration pr ON p.PatronID = pr.PatronID
            WHERE p.PatronID IN (" . implode(',', $patronIDs) . ")
                AND pr.DeliveryOptionID = 2
                AND pr.EmailAddress IS NOT NULL
                AND pr.EmailAddress != ''
        ";

        $results = DB::connection('polaris')->select($query);

        $synced = 0;

        foreach ($results as $patron) {
            EmailPatron::updateOrCreate(
                ['PatronID' => $patron->PatronID],
                [
                    'Barcode' => $patron->Barcode,
                    'EmailAddress' => $patron->EmailAddress,
                    'AltEmailAddress' => $patron->AltEmailAddress,
                    'NameFirst' => $patron->NameFirst,
                    'NameLast' => $patron->NameLast,
                    'EmailFormatID' => $patron->EmailFormatID,
                    'last_synced_at' => Carbon::now(),
                ]
            );
            $synced++;
        }

        $this->info("SQL sync complete: {$synced} patrons synced.");

        return 0;
    }
}
```

#### Same Approach for Mail Patrons

```php
class SyncMailPatronsCommand extends Command
{
    protected $signature = 'polaris:sync-mail-patrons
                            {--days=30 : Only sync patrons with notifications in last N days}';

    public function handle()
    {
        $days = (int) $this->option('days');
        $cutoffDate = Carbon::now()->subDays($days);

        // Get PatronIDs from recent MAIL notifications (DeliveryOptionID = 1)
        $patronIDs = DB::table('polaris_notification_log')
            ->where('DeliveryOptionID', 1) // Mail notifications only
            ->where('CreationDate', '>=', $cutoffDate)
            ->distinct()
            ->pluck('PatronID')
            ->toArray();

        // Then sync mail patron data using SQL or API...
        // (similar to email sync above)
    }
}
```

### Example 2: Real-time Patron Profile Lookup (Web Dashboard)

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Services\PolarisAPIService;

class PatronProfileController extends Controller
{
    protected PolarisAPIService $polarisAPI;

    public function __construct(PolarisAPIService $polarisAPI)
    {
        $this->polarisAPI = $polarisAPI;
    }

    /**
     * Display patron profile with real-time API data
     */
    public function show(Request $request, string $barcode)
    {
        // Get fresh patron data from Polaris API
        $patronData = $this->polarisAPI->getPatronBasicData($barcode);

        if (!$patronData) {
            return view('patrons.not-found', ['barcode' => $barcode]);
        }

        // Get notification history from local database (SQL-based)
        $notificationHistory = \DB::table('polaris_notification_log')
            ->where('PatronID', $patronData->PatronID)
            ->orderBy('CreationDate', 'desc')
            ->limit(50)
            ->get();

        // Get mailing address with fallback
        $mailingAddress = $this->polarisAPI->getPatronMailingAddress($barcode);

        return view('patrons.profile', [
            'patron' => $patronData,
            'mailingAddress' => $mailingAddress,
            'notifications' => $notificationHistory,
        ]);
    }
}
```

### Example 3: Hybrid Approach - SQL for Bulk, API for Real-time

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\DB;
use Carbon\Carbon;

class NotificationService
{
    protected PolarisAPIService $polarisAPI;

    public function __construct(PolarisAPIService $polarisAPI)
    {
        $this->polarisAPI = $polarisAPI;
    }

    /**
     * Get patron contact info using hybrid approach
     * - Check local cache first (SQL-based)
     * - Fall back to API if cache is stale or missing
     */
    public function getPatronContactInfo(string $barcode, bool $forceRefresh = false): ?array
    {
        if (!$forceRefresh) {
            // Try local cache first (fast)
            $cached = DB::table('email_patrons')
                ->where('Barcode', $barcode)
                ->where('last_synced_at', '>', Carbon::now()->subHours(24))
                ->first();

            if ($cached) {
                return (array) $cached;
            }
        }

        // Cache miss or stale - fetch from API (fresh but slower)
        $apiData = $this->polarisAPI->getPatronEmailData($barcode);

        if ($apiData['PatronID']) {
            // Update local cache
            DB::table('email_patrons')->updateOrInsert(
                ['PatronID' => $apiData['PatronID']],
                array_merge($apiData, ['last_synced_at' => Carbon::now()])
            );
        }

        return $apiData;
    }

    /**
     * Export notification logs (always use SQL for bulk operations)
     */
    public function exportNotificationLogs(Carbon $startDate, Carbon $endDate): array
    {
        // SQL is much faster for bulk historical data
        return DB::table('polaris_notification_log')
            ->whereBetween('CreationDate', [$startDate, $endDate])
            ->get()
            ->toArray();
    }
}
```

---

## Performance Considerations

### API vs SQL Performance

| Operation | Recommended Method | Rationale |
|-----------|-------------------|-----------|
| **Single patron lookup** | API (PAPIClient) | Real-time, always fresh |
| **Bulk patron sync (10,000+)** | SQL | 100x faster than API |
| **Notification log exports** | SQL | No API endpoint available |
| **Real-time profile pages** | API (PAPIClient) | User-facing needs fresh data |
| **Daily patron mapping** | SQL for bulk, API for small libraries | SQL: <10 sec, API: ~4 min with parallelization |
| **Deleted patron tracking** | API (SynchGetDeletedPatrons) | Only available via API |

### Rate Limiting Best Practices

```php
// For bulk API calls, implement rate limiting
public function syncPatronsWithRateLimit(array $barcodes, int $requestsPerSecond = 20)
{
    $delayMicroseconds = (int) (1000000 / $requestsPerSecond);

    foreach ($barcodes as $barcode) {
        $this->polarisAPI->getPatronBasicData($barcode);
        usleep($delayMicroseconds); // Throttle requests
    }
}
```

### Caching Strategy

```php
// Cache API responses for performance
use Illuminate\Support\Facades\Cache;

public function getPatronDataCached(string $barcode, int $ttlMinutes = 60): ?array
{
    $cacheKey = "patron_data_{$barcode}";

    return Cache::remember($cacheKey, $ttlMinutes * 60, function () use ($barcode) {
        return $this->polarisAPI->getPatronBasicData($barcode);
    });
}
```

### Parallel Processing (for larger implementations)

```php
use Illuminate\Support\Facades\Process;

// Process patron sync in parallel chunks
public function syncPatronsParallel(array $barcodes, int $chunkSize = 100)
{
    $chunks = array_chunk($barcodes, $chunkSize);

    foreach ($chunks as $index => $chunk) {
        Process::run(
            "php artisan polaris:sync-chunk --chunk={$index}",
            timeout: 600
        );
    }
}
```

---

## Summary

### When to Use PAPIClient

✅ **Use PAPIClient for:**
- Real-time patron profile lookups
- Individual patron verification
- Deleted patron synchronization (only available via API)
- Notification configuration checks
- User-facing web applications
- Small patron sets from notifications (<1,000 active patrons)

❌ **Don't Use PAPIClient for:**
- Bulk notification log exports (use SQL)
- Large patron base sync (>1,000 patrons - use SQL with WHERE IN)
- Statistical dashboards (use SQL aggregations)
- Historical data analysis (use SQL)

### Recommended Notification-Driven Architecture

**⚠️ Key Principle:** Only sync patron data for patrons with **recent notifications** to keep database small.

```
Daily Scheduled Jobs (7:00 AM - 8:00 AM):
├── 7:00 AM: polaris:sync-deleted-patrons --days=1
│   └── API via PAPIClient (only method available)
│
├── 7:10 AM: polaris:import-notification-logs --days=7
│   └── SQL import of NotificationLog table (fast bulk import)
│
├── 7:30 AM: polaris:sync-email-patrons --days=30 --method=sql
│   ├── STEP 1: Extract PatronIDs from recent email notifications (DeliveryOptionID=2)
│   └── STEP 2: Sync ONLY those patrons via SQL (WHERE PatronID IN (...))
│       └── OR use --method=api for very small sets (<200 patrons)
│
└── 7:50 AM: polaris:sync-mail-patrons --days=30 --method=sql
    ├── STEP 1: Extract PatronIDs from recent mail notifications (DeliveryOptionID=1)
    └── STEP 2: Sync ONLY those patrons via SQL (WHERE PatronID IN (...))
        └── OR use --method=api for very small sets (<200 patrons)

Real-time Web Features:
├── Patron profile pages → API via PAPIClient (always fresh data)
├── Dashboard statistics → SQL (fast aggregations)
├── Notification history → SQL (bulk queries)
└── Configuration validation → API via PAPIClient
```

### Workflow Order Matters

**✅ Correct Order:**
1. Delete removed patrons (cleanup first)
2. Import notification logs (get recent activity)
3. Sync patron data for active patrons only (from notification log PatronIDs)

**❌ Wrong Order:**
~~1. Sync all patrons with email/mail preferences~~ (wastes resources, bloats database)

### Benefits of Notification-Driven Sync

- ✅ **Smaller database**: Only tracks patrons who actually receive notifications
- ✅ **Faster sync**: Only syncs hundreds/thousands instead of tens of thousands
- ✅ **Better performance**: Queries are faster with smaller patron tables
- ✅ **Auto-cleanup**: Inactive patrons naturally age out (no notifications = no sync)

---

**Next Steps:**
1. ⭐ Implement `SyncDeletedPatronsCommand` using PAPIClient (see example above)
2. ⭐ Update existing patron sync to use notification-driven approach
3. Add `--days=N` parameter to control lookback window
4. Configure scheduled jobs in `app/Console/Kernel.php`
5. Test with small `--days` value first (e.g., --days=7)

**Last Updated:** November 18, 2025
**Created By:** Claude Code
