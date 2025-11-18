# Polaris API: Confirming Notice Delivery

## Technical Implementation Guide with Laravel HTTP Client

---

## Overview

The Polaris API provides methods to confirm that patron notifications have been successfully delivered. This is essential for maintaining accurate notification logs and ensuring patrons are properly contacted about holds, overdues, and other library notifications.

This guide covers the NotificationUpdatePut endpoint and provides complete Laravel HTTP client implementation examples for PHP applications.

---

## API Endpoint

### NotificationUpdatePut

**Endpoint:** `PUT /PAPIservice/REST/protected/v1/{LangID}/{AppID}/{OrgID}/{AccessToken}/notification/{NotificationTypeID}`

**Purpose:** Updates the Notification Log and removes or updates the Notification Queue entry. This method should be called after a patron is contacted to confirm delivery.

### URL Parameters

| Parameter | Description |
|-----------|-------------|
| **LangID** | Language identifier (integer) |
| **AppID** | Calling application (product) ID |
| **OrgID** | Organization identifier (system-level or branch-level) |
| **AccessToken** | Authentication token obtained from staff authentication |
| **NotificationTypeID** | Notification type identifier (0-20) |

### Request Body Parameters

The request body should be JSON formatted with the following fields:

| Field | Type | Description |
|-------|------|-------------|
| **NotificationStatusID** | integer | Status of the notification |
| **NotificationDeliveryDate** | date-time | Timestamp when notice was delivered |
| **DeliveryOptionID** | integer | Delivery method identifier |
| **DeliveryString** | string | Delivery destination (phone, email, etc.) |
| **Details** | string | Additional delivery details or notes |
| **PatronID** | integer | Patron identifier |
| **PatronBarcode** | string | Patron barcode |
| **ItemRecordID** | integer | Item record identifier (if applicable) |
| **ItemBarcode** | string | Item barcode (if applicable) |
| **LogonBranchID** | integer | Branch ID for logging (optional) |
| **LogonUserID** | integer | User ID for logging (optional) |
| **LogonWorkstationID** | integer | Workstation ID for logging (optional) |
| **ReportingOrgID** | integer | Organization ID for reporting (optional) |
| **PatronLanguageID** | integer | Patron's language preference (optional) |

---

## Laravel HTTP Client Implementation

### Basic Example: Confirming a Notice

```php
<?php

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;
use Carbon\Carbon;

// Configuration
$baseUrl = 'https://your-polaris-server.com/PAPIservice';
$langId = 1033;  // English
$appId = 100;    // Your application ID
$orgId = 1;      // Organization ID
$accessToken = 'your-access-token';  // From staff authentication
$notificationTypeId = 1;  // Notification type

// Prepare notification data
$notificationData = [
    'NotificationStatusID' => 2,  // 2 = Successfully delivered
    'NotificationDeliveryDate' => Carbon::now()->toIso8601String(),
    'DeliveryOptionID' => 1,  // 1 = Phone, 2 = Email, 3 = Text
    'DeliveryString' => '555-0123',  // Phone number or email
    'Details' => 'Successfully delivered via automated system',
    'PatronID' => 12345,
    'PatronBarcode' => '21234567890123',
    'ItemRecordID' => 98765,
    'ItemBarcode' => '31234567890123'
];

// Build the endpoint URL
$endpoint = sprintf(
    '%s/REST/protected/v1/%d/%d/%d/%s/notification/%d',
    $baseUrl,
    $langId,
    $appId,
    $orgId,
    $accessToken,
    $notificationTypeId
);

// Make the PUT request
try {
    $response = Http::timeout(30)
        ->acceptJson()
        ->put($endpoint, $notificationData);

    if ($response->successful()) {
        $result = $response->json();
        
        if ($result['PAPIErrorCode'] === 0) {
            Log::info('Notice marked as sent successfully', [
                'patron_id' => $notificationData['PatronID'],
                'notification_type' => $notificationTypeId
            ]);
            echo "Notice marked as sent successfully!\n";
        } else {
            Log::error('Polaris API error', [
                'error_code' => $result['PAPIErrorCode'],
                'error_message' => $result['ErrorMessage']
            ]);
            echo "Error: " . $result['ErrorMessage'] . "\n";
        }
    } else {
        Log::error('HTTP request failed', [
            'status' => $response->status(),
            'body' => $response->body()
        ]);
        echo "Request failed with status: " . $response->status() . "\n";
    }
} catch (\Exception $e) {
    Log::error('Exception during notification confirmation', [
        'exception' => $e->getMessage(),
        'trace' => $e->getTraceAsString()
    ]);
    echo "Request failed: " . $e->getMessage() . "\n";
}
```

---

## Service Class Implementation

For production use, create a reusable service class:

```php
<?php

namespace App\Services\Polaris;

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;
use Illuminate\Http\Client\Response;
use Carbon\Carbon;

class NotificationService
{
    private string $baseUrl;
    private int $langId;
    private int $appId;
    private int $orgId;
    private string $accessToken;

    public function __construct(
        string $baseUrl,
        int $langId,
        int $appId,
        int $orgId,
        string $accessToken
    ) {
        $this->baseUrl = $baseUrl;
        $this->langId = $langId;
        $this->appId = $appId;
        $this->orgId = $orgId;
        $this->accessToken = $accessToken;
    }

    /**
     * Mark a notification as sent
     *
     * @param int $notificationTypeId Notification type ID
     * @param array $data Notification data
     * @return array Response from API
     * @throws \RuntimeException
     */
    public function markAsSent(int $notificationTypeId, array $data): array
    {
        $endpoint = sprintf(
            '%s/REST/protected/v1/%d/%d/%d/%s/notification/%d',
            $this->baseUrl,
            $this->langId,
            $this->appId,
            $this->orgId,
            $this->accessToken,
            $notificationTypeId
        );

        try {
            $response = Http::timeout(30)
                ->acceptJson()
                ->put($endpoint, $data);

            if (!$response->successful()) {
                throw new \RuntimeException(
                    "HTTP request failed with status {$response->status()}: {$response->body()}"
                );
            }

            $result = $response->json();

            if ($result['PAPIErrorCode'] !== 0) {
                throw new \RuntimeException(
                    "Polaris API error: {$result['ErrorMessage']} (Code: {$result['PAPIErrorCode']})"
                );
            }

            return $result;

        } catch (\Exception $e) {
            Log::error('Failed to mark notification as sent', [
                'notification_type_id' => $notificationTypeId,
                'patron_id' => $data['PatronID'] ?? null,
                'exception' => $e->getMessage()
            ]);

            throw new \RuntimeException(
                'Failed to mark notification as sent: ' . $e->getMessage(),
                0,
                $e
            );
        }
    }

    /**
     * Helper method to build notification data array
     *
     * @param int $patronId
     * @param string $patronBarcode
     * @param int $deliveryOptionId (1=Phone, 2=Email, 3=Text)
     * @param string $deliveryString
     * @param int|null $itemRecordId
     * @param string|null $itemBarcode
     * @param string|null $details
     * @return array
     */
    public function buildNotificationData(
        int $patronId,
        string $patronBarcode,
        int $deliveryOptionId,
        string $deliveryString,
        ?int $itemRecordId = null,
        ?string $itemBarcode = null,
        ?string $details = null
    ): array {
        $data = [
            'NotificationStatusID' => 2, // Successfully delivered
            'NotificationDeliveryDate' => Carbon::now()->toIso8601String(),
            'DeliveryOptionID' => $deliveryOptionId,
            'DeliveryString' => $deliveryString,
            'PatronID' => $patronId,
            'PatronBarcode' => $patronBarcode,
        ];

        if ($itemRecordId !== null) {
            $data['ItemRecordID'] = $itemRecordId;
        }

        if ($itemBarcode !== null) {
            $data['ItemBarcode'] = $itemBarcode;
        }

        if ($details !== null) {
            $data['Details'] = $details;
        }

        return $data;
    }
}
```

---

## Usage in Laravel Controller

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Services\Polaris\NotificationService;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class NotificationController extends Controller
{
    private NotificationService $notificationService;

    public function __construct(NotificationService $notificationService)
    {
        $this->notificationService = $notificationService;
    }

    /**
     * Confirm that a notification was sent
     *
     * @param Request $request
     * @return JsonResponse
     */
    public function confirmSent(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'notification_type_id' => 'required|integer|min:0|max:20',
            'patron_id' => 'required|integer',
            'patron_barcode' => 'required|string',
            'delivery_option_id' => 'required|integer|in:1,2,3,4',
            'delivery_string' => 'required|string',
            'item_record_id' => 'nullable|integer',
            'item_barcode' => 'nullable|string',
            'details' => 'nullable|string|max:500'
        ]);

        try {
            $notificationData = $this->notificationService->buildNotificationData(
                patronId: $validated['patron_id'],
                patronBarcode: $validated['patron_barcode'],
                deliveryOptionId: $validated['delivery_option_id'],
                deliveryString: $validated['delivery_string'],
                itemRecordId: $validated['item_record_id'] ?? null,
                itemBarcode: $validated['item_barcode'] ?? null,
                details: $validated['details'] ?? null
            );

            $result = $this->notificationService->markAsSent(
                $validated['notification_type_id'],
                $notificationData
            );

            return response()->json([
                'success' => true,
                'message' => 'Notification marked as sent',
                'data' => $result
            ]);

        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => 'Failed to confirm notification',
                'error' => $e->getMessage()
            ], 500);
        }
    }
}
```

---

## Service Provider Configuration

Register the service in `AppServiceProvider`:

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Services\Polaris\NotificationService;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(NotificationService::class, function ($app) {
            return new NotificationService(
                baseUrl: config('polaris.base_url'),
                langId: config('polaris.lang_id', 1033),
                appId: config('polaris.app_id'),
                orgId: config('polaris.org_id'),
                accessToken: config('polaris.access_token')
            );
        });
    }
}
```

---

## Configuration File

Create `config/polaris.php`:

```php
<?php

return [
    /*
    |--------------------------------------------------------------------------
    | Polaris API Configuration
    |--------------------------------------------------------------------------
    */

    'base_url' => env('POLARIS_BASE_URL', 'https://your-polaris-server.com/PAPIservice'),
    
    'lang_id' => env('POLARIS_LANG_ID', 1033), // 1033 = English
    
    'app_id' => env('POLARIS_APP_ID'),
    
    'org_id' => env('POLARIS_ORG_ID', 1),
    
    'access_token' => env('POLARIS_ACCESS_TOKEN'),
    
    'timeout' => env('POLARIS_TIMEOUT', 30),
];
```

Add to `.env`:

```
POLARIS_BASE_URL=https://your-polaris-server.com/PAPIservice
POLARIS_APP_ID=100
POLARIS_ORG_ID=1
POLARIS_ACCESS_TOKEN=your-access-token-here
POLARIS_LANG_ID=1033
POLARIS_TIMEOUT=30
```

---

## Integration with Shoutbomb

### Example: Confirming Text Message Delivery

```php
<?php

namespace App\Services\Shoutbomb;

use App\Services\Polaris\NotificationService;
use Illuminate\Support\Facades\Log;

class ShoutbombCallbackHandler
{
    private NotificationService $notificationService;

    public function __construct(NotificationService $notificationService)
    {
        $this->notificationService = $notificationService;
    }

    /**
     * Handle Shoutbomb delivery confirmation callback
     *
     * @param array $callbackData Data from Shoutbomb webhook
     * @return bool
     */
    public function handleDeliveryConfirmation(array $callbackData): bool
    {
        try {
            // Extract data from Shoutbomb callback
            $patronId = $callbackData['patron_id'];
            $patronBarcode = $callbackData['patron_barcode'];
            $phoneNumber = $callbackData['phone_number'];
            $itemRecordId = $callbackData['item_record_id'] ?? null;
            $itemBarcode = $callbackData['item_barcode'] ?? null;
            $notificationTypeId = $callbackData['notification_type_id'];
            $deliveryStatus = $callbackData['delivery_status']; // 'sent', 'delivered', 'failed'

            // Only confirm if message was successfully delivered
            if ($deliveryStatus !== 'delivered' && $deliveryStatus !== 'sent') {
                Log::warning('Skipping confirmation for non-delivered message', [
                    'patron_id' => $patronId,
                    'status' => $deliveryStatus
                ]);
                return false;
            }

            // Build notification data
            $notificationData = $this->notificationService->buildNotificationData(
                patronId: $patronId,
                patronBarcode: $patronBarcode,
                deliveryOptionId: 3, // 3 = Text Message
                deliveryString: $phoneNumber,
                itemRecordId: $itemRecordId,
                itemBarcode: $itemBarcode,
                details: "Shoutbomb SMS delivery confirmed: {$deliveryStatus}"
            );

            // Confirm in Polaris
            $result = $this->notificationService->markAsSent(
                $notificationTypeId,
                $notificationData
            );

            Log::info('Successfully confirmed notification in Polaris', [
                'patron_id' => $patronId,
                'notification_type_id' => $notificationTypeId,
                'polaris_response' => $result
            ]);

            return true;

        } catch (\Exception $e) {
            Log::error('Failed to confirm Shoutbomb delivery in Polaris', [
                'callback_data' => $callbackData,
                'exception' => $e->getMessage()
            ]);
            return false;
        }
    }
}
```

---

## Best Practices

1. **Always use ISO 8601 format for dates**: Use Carbon's `toIso8601String()` method to ensure proper formatting.

2. **Handle errors gracefully**: Check the PAPIErrorCode in the response and implement proper error logging.

3. **Set appropriate timeouts**: The default 30-second timeout is recommended for API calls.

4. **Log all notification confirmations**: Maintain your own audit trail of confirmed notifications for reporting purposes.

5. **Validate data before sending**: Ensure PatronID, ItemRecordID, and other identifiers exist before attempting to confirm delivery.

6. **Use environment variables**: Store API credentials and configuration in `.env`, not in code.

7. **Implement retry logic**: For temporary network failures, consider using Laravel's HTTP client retry feature:

```php
$response = Http::timeout(30)
    ->retry(3, 100) // Retry 3 times with 100ms delay
    ->acceptJson()
    ->put($endpoint, $data);
```

8. **Use dependency injection**: Inject the NotificationService into controllers and other services for testability.

9. **Monitor API usage**: Log all API calls and monitor for errors or rate limiting issues.

10. **Keep access tokens fresh**: Access tokens typically expire after 24 hours - implement token refresh logic.

---

## Common Issues & Troubleshooting

### Authentication Errors

**Problem:** 401 Unauthorized response

**Solutions:**
- Verify that your access token is valid and not expired
- Ensure you're using the correct authentication method for protected endpoints
- Access tokens typically expire after 24 hours - implement token refresh logic
- Check that your API credentials are correctly set in `.env`

### Date Format Errors

**Problem:** Invalid date format errors

**Solutions:**
- Always use ISO 8601 format: `Carbon::now()->toIso8601String()`
- Example: `2024-11-13T14:30:00-05:00`
- Avoid using `date('Y-m-d H:i:s')` - this is not ISO 8601 compliant

### Missing Required Fields

**Problem:** API returns error about missing fields

**Solutions:**
- Ensure PatronID or PatronBarcode is provided
- NotificationStatusID and DeliveryOptionID are required
- NotificationDeliveryDate must be provided to confirm delivery
- Use the `buildNotificationData` helper method to ensure all required fields are included

### HTTP Timeout Errors

**Problem:** Requests timing out

**Solutions:**
- Increase timeout value: `Http::timeout(60)`
- Implement retry logic with exponential backoff
- Check network connectivity to Polaris server
- Consider queueing notification confirmations for asynchronous processing

### Rate Limiting

**Problem:** Too many requests causing failures

**Solutions:**
- Implement request throttling using Laravel's rate limiter
- Queue notification confirmations and process them in batches
- Add delays between bulk confirmation operations
- Monitor API usage and adjust accordingly

---

## Appendix: Reference Tables

### Common Notification Status IDs

| Status ID | Description |
|-----------|-------------|
| 1 | Pending |
| 2 | Successfully Delivered |
| 3 | Failed Delivery |

### Common Delivery Option IDs

| Option ID | Delivery Method |
|-----------|----------------|
| 1 | Phone |
| 2 | Email |
| 3 | Text Message (SMS) |
| 4 | Print |

### Common Notification Type IDs

| Type ID | Notification Type |
|---------|------------------|
| 1 | Hold Ready for Pickup |
| 2 | Overdue Notice |
| 3 | Courtesy Notice |
| 4 | Bill Notice |
| 5 | Pre-Overdue Notice |

**Note:** Actual status IDs, delivery option IDs, and notification type IDs may vary by Polaris installation. Consult your Polaris administrator for the exact IDs used in your system.

---

## Testing

### Unit Test Example

```php
<?php

namespace Tests\Unit\Services\Polaris;

use Tests\TestCase;
use App\Services\Polaris\NotificationService;
use Illuminate\Support\Facades\Http;
use Carbon\Carbon;

class NotificationServiceTest extends TestCase
{
    public function test_mark_as_sent_success()
    {
        // Mock the HTTP response
        Http::fake([
            '*' => Http::response([
                'PAPIErrorCode' => 0,
                'ErrorMessage' => null
            ], 200)
        ]);

        $service = new NotificationService(
            'https://test-polaris.com/PAPIservice',
            1033,
            100,
            1,
            'test-token'
        );

        $data = $service->buildNotificationData(
            patronId: 12345,
            patronBarcode: '21234567890123',
            deliveryOptionId: 3,
            deliveryString: '555-0123'
        );

        $result = $service->markAsSent(1, $data);

        $this->assertEquals(0, $result['PAPIErrorCode']);
    }

    public function test_build_notification_data()
    {
        $service = new NotificationService(
            'https://test-polaris.com/PAPIservice',
            1033,
            100,
            1,
            'test-token'
        );

        $data = $service->buildNotificationData(
            patronId: 12345,
            patronBarcode: '21234567890123',
            deliveryOptionId: 3,
            deliveryString: '555-0123',
            itemRecordId: 98765,
            itemBarcode: '31234567890123',
            details: 'Test details'
        );

        $this->assertEquals(12345, $data['PatronID']);
        $this->assertEquals('21234567890123', $data['PatronBarcode']);
        $this->assertEquals(3, $data['DeliveryOptionID']);
        $this->assertEquals('555-0123', $data['DeliveryString']);
        $this->assertEquals(98765, $data['ItemRecordID']);
        $this->assertEquals('31234567890123', $data['ItemBarcode']);
        $this->assertEquals('Test details', $data['Details']);
        $this->assertEquals(2, $data['NotificationStatusID']);
        $this->assertNotNull($data['NotificationDeliveryDate']);
    }
}
```

---

## Additional Resources

- [Polaris API Documentation](https://your-polaris-server.com/PAPIservice) (contact your Polaris administrator)
- [Laravel HTTP Client Documentation](https://laravel.com/docs/http-client)
- [Carbon Date/Time Library](https://carbon.nesbot.com/docs/)

---

*Document Version: 1.0*  
*Last Updated: November 13, 2024*
