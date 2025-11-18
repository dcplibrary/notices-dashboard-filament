# Polaris API: Confirming Notice Delivery

## Implementation Guide Using blashbrook/papiclient

---

## Overview

This guide demonstrates how to use the `blashbrook/papiclient` Laravel package to confirm that patron notifications have been successfully delivered through the Polaris API. The PAPIClient provides a fluent interface specifically designed for Polaris API integration.

### Key Benefits of Using PAPIClient

- **Fluent Interface**: Chain commands together for cleaner, more readable code
- **Pre-configured**: Built-in handling of Polaris API authentication and headers
- **Dependency Injection**: Easy integration with Laravel's service container
- **Environment-based Configuration**: Secure credential management through `.env` files
- **Session Management**: Built-in support for patron authentication tokens

---

## Installation

If you haven't already installed the package:

```bash
composer require blashbrook/papiclient
```

### Environment Configuration

Add these variables to your `.env` file:

```env
# Polaris API Credentials
PAPI_ACCESS_ID=your_access_id
PAPI_ACCESS_KEY=your_access_key
PAPI_BASE_URL=https://catalog.yourlibrary.org/PAPIService/REST

# API Configuration (defaults shown)
PAPI_PROTECTED_SCOPE=protected
PAPI_PUBLIC_SCOPE=public
PAPI_VERSION=v1
PAPI_LANGID=1033
PAPI_APPID=100
PAPI_ORGID=3

# Staff Authentication (for protected endpoints)
PAPI_LOGONBRANCHID=your_branch_id
PAPI_LOGONUSERID=your_user_id
PAPI_LOGONWORKSTATIONID=your_workstation_id
PAPI_DOMAIN=your_domain
PAPI_STAFF=your_staff_username
PAPI_PASSWORD="your_password"

# Notifications
PAPI_ADMIN_EMAIL="notifications@yourlibrary.org"
PAPI_ADMIN_NAME="Library Admin"
```

---

## API Endpoint Overview

### NotificationUpdatePut

**Endpoint:** `PUT /REST/protected/v1/{LangID}/{AppID}/{OrgID}/{AccessToken}/notification/{NotificationTypeID}`

**Purpose:** Updates the Notification Log and removes or updates the Notification Queue entry. This method should be called after a patron is contacted to confirm delivery.

### URL Parameters

| Parameter | Description |
|-----------|-------------|
| **NotificationTypeID** | Notification type identifier (0-20) in the URL path |

### Request Body Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| **NotificationStatusID** | integer | ✓ | Status of the notification (2 = Successfully delivered) |
| **NotificationDeliveryDate** | date-time | ✓ | Timestamp when notice was delivered (ISO 8601) |
| **DeliveryOptionID** | integer | ✓ | Delivery method (1=Phone, 2=Email, 3=Text, 4=Print) |
| **DeliveryString** | string | ✓ | Delivery destination (phone number, email, etc.) |
| **PatronID** | integer | ✓ | Patron identifier |
| **PatronBarcode** | string | ✓ | Patron barcode |
| **ItemRecordID** | integer | ○ | Item record identifier (if applicable) |
| **ItemBarcode** | string | ○ | Item barcode (if applicable) |
| **Details** | string | ○ | Additional delivery details or notes |
| **LogonBranchID** | integer | ○ | Branch ID for logging |
| **LogonUserID** | integer | ○ | User ID for logging |
| **LogonWorkstationID** | integer | ○ | Workstation ID for logging |
| **ReportingOrgID** | integer | ○ | Organization ID for reporting |
| **PatronLanguageID** | integer | ○ | Patron's language preference |

---

## Basic Implementation

### Simple Example: Confirming a Text Message Delivery

```php
<?php

namespace App\Services;

use Blashbrook\PAPIClient\PAPIClient;
use Illuminate\Support\Facades\Log;
use Carbon\Carbon;

class NotificationConfirmationService
{
    protected PAPIClient $papiclient;

    public function __construct(PAPIClient $papiclient)
    {
        $this->papiclient = $papiclient;
    }

    /**
     * Confirm a notification has been sent
     */
    public function confirmNotificationSent(array $data): array
    {
        $notificationTypeId = $data['notification_type_id'];
        
        // Build the request data
        $requestData = [
            'NotificationStatusID' => 16, // Sent - Use specific status based on delivery method
            'NotificationDeliveryDate' => Carbon::now()->toIso8601String(),
            'DeliveryOptionID' => $data['delivery_option_id'],
            'DeliveryString' => $data['delivery_string'],
            'PatronID' => $data['patron_id'],
            'PatronBarcode' => $data['patron_barcode'],
            'ItemRecordID' => $data['item_record_id'] ?? null,
            'ItemBarcode' => $data['item_barcode'] ?? null,
            'Details' => $data['details'] ?? 'Notification delivered via automated system'
        ];

        // Remove null values
        $requestData = array_filter($requestData, fn($value) => $value !== null);

        try {
            // Make the API call using PAPIClient's fluent interface
            $response = $this->papiclient
                ->method('PUT')
                ->protected() // Use protected endpoint
                ->uri("notification/{$notificationTypeId}")
                ->params($requestData)
                ->execRequest();

            // Check response
            if ($response['PAPIErrorCode'] === 0) {
                Log::info('Notification marked as sent successfully', [
                    'patron_id' => $data['patron_id'],
                    'notification_type_id' => $notificationTypeId
                ]);

                return [
                    'success' => true,
                    'message' => 'Notification confirmed',
                    'response' => $response
                ];
            } else {
                Log::error('Polaris API error confirming notification', [
                    'error_code' => $response['PAPIErrorCode'],
                    'error_message' => $response['ErrorMessage'],
                    'patron_id' => $data['patron_id']
                ]);

                return [
                    'success' => false,
                    'message' => $response['ErrorMessage'],
                    'error_code' => $response['PAPIErrorCode']
                ];
            }
        } catch (\Exception $e) {
            Log::error('Exception confirming notification', [
                'exception' => $e->getMessage(),
                'patron_id' => $data['patron_id'] ?? null
            ]);

            return [
                'success' => false,
                'message' => 'Failed to confirm notification: ' . $e->getMessage()
            ];
        }
    }
}
```

---

## Choosing the Correct Notification Status ID

When confirming a notification delivery, use the appropriate status ID based on the delivery method and result:

### For Voice Calls (Phone 1, Phone 2, Phone 3)
- **Status 1** (Call completed - Voice): Person answered and received the message
- **Status 2** (Call completed - Answering machine): Message left on voicemail
- **Status 3-11**: Various call failure statuses (use based on specific failure reason)

### For Text Messages (TXT Messaging)
- **Status 16** (Sent): Generic successful delivery
- **Status 12** (Email Completed): Can also be used for SMS confirmations

### For Email (Email Address)
- **Status 12** (Email Completed): Email successfully sent
- **Status 13** (Email Failed - Invalid address): Invalid email
- **Status 14** (Email Failed): Other email failures

### For Physical Mail (Mailing Address)
- **Status 15** (Mail Printed): Notice has been printed and ready for mailing
- **Status 16** (Sent): Generic status for completed mailing

**General Rule:** Use **Status 16 (Sent)** when you need a generic "successfully delivered" status that works across all delivery methods.

---

## Service Class Implementation

### Complete Notification Service with Helper Methods

```php
<?php

namespace App\Services\Polaris;

use Blashbrook\PAPIClient\PAPIClient;
use Illuminate\Support\Facades\Log;
use Carbon\Carbon;

class NotificationService
{
    protected PAPIClient $papiclient;

    public function __construct(PAPIClient $papiclient)
    {
        $this->papiclient = $papiclient;
    }

    /**
     * Mark a notification as sent
     *
     * @param int $notificationTypeId
     * @param array $data Notification data
     * @return array API response
     */
    public function markAsSent(int $notificationTypeId, array $data): array
    {
        // Ensure required fields are present
        $this->validateNotificationData($data);

        // Add delivery timestamp if not provided
        if (!isset($data['NotificationDeliveryDate'])) {
            $data['NotificationDeliveryDate'] = Carbon::now()->toIso8601String();
        }

        // Set status to delivered if not specified
        if (!isset($data['NotificationStatusID'])) {
            $data['NotificationStatusID'] = 2; // Successfully delivered
        }

        try {
            $response = $this->papiclient
                ->method('PUT')
                ->protected()
                ->uri("notification/{$notificationTypeId}")
                ->params($data)
                ->execRequest();

            if ($response['PAPIErrorCode'] === 0) {
                Log::info('Notification confirmed in Polaris', [
                    'notification_type_id' => $notificationTypeId,
                    'patron_id' => $data['PatronID'],
                    'delivery_option' => $data['DeliveryOptionID']
                ]);
            } else {
                Log::warning('Polaris notification confirmation returned error', [
                    'error_code' => $response['PAPIErrorCode'],
                    'error_message' => $response['ErrorMessage']
                ]);
            }

            return $response;

        } catch (\Exception $e) {
            Log::error('Failed to confirm notification in Polaris', [
                'exception' => $e->getMessage(),
                'notification_type_id' => $notificationTypeId,
                'data' => $data
            ]);

            throw new \RuntimeException(
                'Failed to mark notification as sent: ' . $e->getMessage(),
                0,
                $e
            );
        }
    }

    /**
     * Build notification data array with sensible defaults
     *
     * @param int $patronId
     * @param string $patronBarcode
     * @param int $deliveryOptionId (1=Mail, 2=Email, 3=Phone1, 4=Phone2, 5=Phone3, 6=FAX, 7=EDI, 8=TXT)
     * @param string $deliveryString (phone number, email address, etc.)
     * @param int|null $itemRecordId
     * @param string|null $itemBarcode
     * @param string|null $details
     * @param int $statusId (default 16=Sent, or specify 1=Voice, 2=Voicemail, 12=Email, etc.)
     * @return array
     */
    public function buildNotificationData(
        int $patronId,
        string $patronBarcode,
        int $deliveryOptionId,
        string $deliveryString,
        ?int $itemRecordId = null,
        ?string $itemBarcode = null,
        ?string $details = null,
        int $statusId = 16
    ): array {
        $data = [
            'NotificationStatusID' => $statusId, // Default 16 (Sent) or custom status
            'NotificationDeliveryDate' => Carbon::now()->toIso8601String(),
            'DeliveryOptionID' => $deliveryOptionId,
            'DeliveryString' => $deliveryString,
            'PatronID' => $patronId,
            'PatronBarcode' => $patronBarcode,
        ];

        // Add optional fields if provided
        if ($itemRecordId !== null) {
            $data['ItemRecordID'] = $itemRecordId;
        }

        if ($itemBarcode !== null) {
            $data['ItemBarcode'] = $itemBarcode;
        }

        if ($details !== null) {
            $data['Details'] = $details;
        }

        // Add logon information from environment if available
        if ($logonBranchId = config('papiclient.logonbranchid')) {
            $data['LogonBranchID'] = (int) $logonBranchId;
        }

        if ($logonUserId = config('papiclient.logonuserid')) {
            $data['LogonUserID'] = (int) $logonUserId;
        }

        if ($logonWorkstationId = config('papiclient.logonworkstationid')) {
            $data['LogonWorkstationID'] = (int) $logonWorkstationId;
        }

        return $data;
    }

    /**
     * Validate notification data has required fields
     *
     * @param array $data
     * @throws \InvalidArgumentException
     */
    protected function validateNotificationData(array $data): void
    {
        $required = ['DeliveryOptionID', 'DeliveryString', 'PatronID', 'PatronBarcode'];

        foreach ($required as $field) {
            if (!isset($data[$field]) || empty($data[$field])) {
                throw new \InvalidArgumentException("Required field '{$field}' is missing or empty");
            }
        }

        // Validate DeliveryOptionID is in valid range
        if (!in_array($data['DeliveryOptionID'], [1, 2, 3, 4, 5, 6, 7, 8])) {
            throw new \InvalidArgumentException(
                'DeliveryOptionID must be 1-8 (1=Mail, 2=Email, 3=Phone1, 4=Phone2, 5=Phone3, 6=FAX, 7=EDI, 8=TXT)'
            );
        }
    }

    /**
     * Confirm text message delivery
     * Uses Status 16 (Sent) for successful text message delivery
     */
    public function confirmTextMessage(
        int $notificationTypeId,
        int $patronId,
        string $patronBarcode,
        string $phoneNumber,
        ?int $itemRecordId = null,
        ?string $itemBarcode = null
    ): array {
        $data = $this->buildNotificationData(
            patronId: $patronId,
            patronBarcode: $patronBarcode,
            deliveryOptionId: 8, // TXT Messaging
            deliveryString: $phoneNumber,
            itemRecordId: $itemRecordId,
            itemBarcode: $itemBarcode,
            details: 'Text message delivered via Shoutbomb'
        );

        return $this->markAsSent($notificationTypeId, $data);
    }

    /**
     * Confirm voice call delivery
     * Uses Status 16 (Sent) - consider using Status 1 or 2 based on actual result
     */
    public function confirmVoiceCall(
        int $notificationTypeId,
        int $patronId,
        string $patronBarcode,
        string $phoneNumber,
        ?int $itemRecordId = null,
        ?string $itemBarcode = null
    ): array {
        $data = $this->buildNotificationData(
            patronId: $patronId,
            patronBarcode: $patronBarcode,
            deliveryOptionId: 3, // Phone 1
            deliveryString: $phoneNumber,
            itemRecordId: $itemRecordId,
            itemBarcode: $itemBarcode,
            details: 'Voice call delivered via Shoutbomb'
        );

        return $this->markAsSent($notificationTypeId, $data);
    }

    /**
     * Confirm email delivery
     * Uses Status 16 (Sent) - consider using Status 12 (Email Completed) for emails
     */
    public function confirmEmail(
        int $notificationTypeId,
        int $patronId,
        string $patronBarcode,
        string $emailAddress,
        ?int $itemRecordId = null,
        ?string $itemBarcode = null
    ): array {
        $data = $this->buildNotificationData(
            patronId: $patronId,
            patronBarcode: $patronBarcode,
            deliveryOptionId: 2, // Email
            deliveryString: $emailAddress,
            itemRecordId: $itemRecordId,
            itemBarcode: $itemBarcode,
            details: 'Email delivered'
        );

        return $this->markAsSent($notificationTypeId, $data);
    }
}
```

---

## Integration with Shoutbomb

### Webhook Handler for Delivery Confirmations

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Services\Polaris\NotificationService;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Log;

class ShoutbombWebhookController extends Controller
{
    protected NotificationService $notificationService;

    public function __construct(NotificationService $notificationService)
    {
        $this->notificationService = $notificationService;
    }

    /**
     * Handle Shoutbomb delivery confirmation webhook
     *
     * @param Request $request
     * @return JsonResponse
     */
    public function handleDeliveryConfirmation(Request $request): JsonResponse
    {
        // Log incoming webhook
        Log::info('Shoutbomb webhook received', [
            'payload' => $request->all()
        ]);

        // Validate webhook data
        $validated = $request->validate([
            'notification_type_id' => 'required|integer|min:0|max:20',
            'patron_id' => 'required|integer',
            'patron_barcode' => 'required|string',
            'delivery_type' => 'required|string|in:text,voice,email',
            'delivery_destination' => 'required|string',
            'delivery_status' => 'required|string|in:sent,delivered,failed',
            'item_record_id' => 'nullable|integer',
            'item_barcode' => 'nullable|string',
            'timestamp' => 'nullable|date',
        ]);

        // Only confirm successful deliveries
        if (!in_array($validated['delivery_status'], ['sent', 'delivered'])) {
            Log::warning('Skipping non-delivered notification', [
                'status' => $validated['delivery_status'],
                'patron_id' => $validated['patron_id']
            ]);

            return response()->json([
                'success' => true,
                'message' => 'Delivery not confirmed - status: ' . $validated['delivery_status']
            ]);
        }

        try {
            // Determine delivery method
            $deliveryMethod = match($validated['delivery_type']) {
                'text' => 'confirmTextMessage',
                'voice' => 'confirmVoiceCall',
                'email' => 'confirmEmail',
                default => throw new \InvalidArgumentException('Unknown delivery type')
            };

            // Confirm delivery in Polaris
            $result = $this->notificationService->$deliveryMethod(
                notificationTypeId: $validated['notification_type_id'],
                patronId: $validated['patron_id'],
                patronBarcode: $validated['patron_barcode'],
                phoneNumber: $validated['delivery_destination'], // Works for all types
                itemRecordId: $validated['item_record_id'] ?? null,
                itemBarcode: $validated['item_barcode'] ?? null
            );

            if ($result['PAPIErrorCode'] === 0) {
                Log::info('Successfully confirmed Shoutbomb delivery in Polaris', [
                    'patron_id' => $validated['patron_id'],
                    'notification_type_id' => $validated['notification_type_id']
                ]);

                return response()->json([
                    'success' => true,
                    'message' => 'Notification confirmed in Polaris',
                    'polaris_response' => $result
                ]);
            } else {
                Log::error('Polaris API returned error', [
                    'error_code' => $result['PAPIErrorCode'],
                    'error_message' => $result['ErrorMessage']
                ]);

                return response()->json([
                    'success' => false,
                    'message' => 'Polaris API error: ' . $result['ErrorMessage']
                ], 500);
            }

        } catch (\Exception $e) {
            Log::error('Failed to process Shoutbomb webhook', [
                'exception' => $e->getMessage(),
                'trace' => $e->getTraceAsString(),
                'payload' => $validated
            ]);

            return response()->json([
                'success' => false,
                'message' => 'Failed to process webhook: ' . $e->getMessage()
            ], 500);
        }
    }
}
```

### Route Registration

```php
// routes/api.php

use App\Http\Controllers\Api\ShoutbombWebhookController;

Route::post('/webhooks/shoutbomb/delivery', [ShoutbombWebhookController::class, 'handleDeliveryConfirmation'])
    ->middleware(['api']) // Add appropriate authentication middleware
    ->name('shoutbomb.webhook.delivery');
```

---

## Batch Processing

### Processing Multiple Notifications from Reports

```php
<?php

namespace App\Console\Commands;

use App\Services\Polaris\NotificationService;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\Storage;
use Carbon\Carbon;

class ConfirmBatchNotifications extends Command
{
    protected $signature = 'notifications:confirm-batch {file}';
    protected $description = 'Confirm multiple notifications from a CSV file';

    protected NotificationService $notificationService;

    public function __construct(NotificationService $notificationService)
    {
        parent::__construct();
        $this->notificationService = $notificationService;
    }

    public function handle(): int
    {
        $filePath = $this->argument('file');

        if (!Storage::exists($filePath)) {
            $this->error("File not found: {$filePath}");
            return 1;
        }

        $csv = array_map('str_getcsv', file(Storage::path($filePath)));
        $header = array_shift($csv); // Remove header row

        $successCount = 0;
        $failureCount = 0;

        $this->info("Processing " . count($csv) . " notifications...");
        $progressBar = $this->output->createProgressBar(count($csv));

        foreach ($csv as $row) {
            $data = array_combine($header, $row);

            try {
                // Build notification data from CSV row
                $notificationData = $this->notificationService->buildNotificationData(
                    patronId: (int) $data['patron_id'],
                    patronBarcode: $data['patron_barcode'],
                    deliveryOptionId: (int) $data['delivery_option_id'],
                    deliveryString: $data['delivery_string'],
                    itemRecordId: !empty($data['item_record_id']) ? (int) $data['item_record_id'] : null,
                    itemBarcode: $data['item_barcode'] ?? null,
                    details: $data['details'] ?? 'Batch confirmation'
                );

                // Confirm in Polaris
                $result = $this->notificationService->markAsSent(
                    (int) $data['notification_type_id'],
                    $notificationData
                );

                if ($result['PAPIErrorCode'] === 0) {
                    $successCount++;
                } else {
                    $failureCount++;
                    $this->warn("\nFailed for patron {$data['patron_id']}: {$result['ErrorMessage']}");
                }

            } catch (\Exception $e) {
                $failureCount++;
                $this->error("\nException for patron {$data['patron_id']}: {$e->getMessage()}");
            }

            $progressBar->advance();

            // Small delay to avoid overwhelming the API
            usleep(100000); // 100ms delay
        }

        $progressBar->finish();

        $this->newLine(2);
        $this->info("Batch processing complete!");
        $this->info("Successful: {$successCount}");
        $this->info("Failed: {$failureCount}");

        return 0;
    }
}
```

---

## Usage in Livewire Components

### Real-time Notification Confirmation

```php
<?php

namespace App\Livewire;

use Blashbrook\PAPIClient\PAPIClient;
use App\Services\Polaris\NotificationService;
use Livewire\Component;
use Livewire\Attributes\On;

class NotificationDashboard extends Component
{
    protected PAPIClient $papiclient;
    protected NotificationService $notificationService;

    public $pendingNotifications = [];
    public $confirmedCount = 0;

    public function boot(
        PAPIClient $papiclient,
        NotificationService $notificationService
    ) {
        $this->papiclient = $papiclient;
        $this->notificationService = $notificationService;
    }

    public function mount()
    {
        $this->loadPendingNotifications();
    }

    protected function loadPendingNotifications()
    {
        // Get pending notifications from Polaris
        try {
            $response = $this->papiclient
                ->method('GET')
                ->protected()
                ->uri('notification')
                ->execRequest();

            if ($response['PAPIErrorCode'] === 0) {
                $this->pendingNotifications = $response['NotificationQueueGetRows'] ?? [];
            }
        } catch (\Exception $e) {
            session()->flash('error', 'Failed to load notifications: ' . $e->getMessage());
        }
    }

    #[On('notification-sent')]
    public function confirmNotification($data)
    {
        try {
            $notificationData = $this->notificationService->buildNotificationData(
                patronId: $data['patron_id'],
                patronBarcode: $data['patron_barcode'],
                deliveryOptionId: $data['delivery_option_id'],
                deliveryString: $data['delivery_string'],
                itemRecordId: $data['item_record_id'] ?? null,
                itemBarcode: $data['item_barcode'] ?? null,
                details: $data['details'] ?? null
            );

            $result = $this->notificationService->markAsSent(
                $data['notification_type_id'],
                $notificationData
            );

            if ($result['PAPIErrorCode'] === 0) {
                $this->confirmedCount++;
                $this->loadPendingNotifications(); // Refresh list
                
                session()->flash('success', 'Notification confirmed successfully');
            } else {
                session()->flash('error', 'Polaris error: ' . $result['ErrorMessage']);
            }

        } catch (\Exception $e) {
            session()->flash('error', 'Failed to confirm: ' . $e->getMessage());
        }
    }

    public function render()
    {
        return view('livewire.notification-dashboard');
    }
}
```

---

## Testing

### Unit Test Example

```php
<?php

namespace Tests\Unit\Services;

use Tests\TestCase;
use App\Services\Polaris\NotificationService;
use Blashbrook\PAPIClient\PAPIClient;
use Mockery;

class NotificationServiceTest extends TestCase
{
    protected function tearDown(): void
    {
        Mockery::close();
        parent::tearDown();
    }

    public function test_build_notification_data_includes_required_fields()
    {
        $papiclient = Mockery::mock(PAPIClient::class);
        $service = new NotificationService($papiclient);

        $data = $service->buildNotificationData(
            patronId: 12345,
            patronBarcode: '21234567890123',
            deliveryOptionId: 3,
            deliveryString: '555-0123'
        );

        $this->assertEquals(12345, $data['PatronID']);
        $this->assertEquals('21234567890123', $data['PatronBarcode']);
        $this->assertEquals(3, $data['DeliveryOptionID']);
        $this->assertEquals('555-0123', $data['DeliveryString']);
        $this->assertEquals(2, $data['NotificationStatusID']);
        $this->assertArrayHasKey('NotificationDeliveryDate', $data);
    }

    public function test_build_notification_data_includes_optional_item_fields()
    {
        $papiclient = Mockery::mock(PAPIClient::class);
        $service = new NotificationService($papiclient);

        $data = $service->buildNotificationData(
            patronId: 12345,
            patronBarcode: '21234567890123',
            deliveryOptionId: 3,
            deliveryString: '555-0123',
            itemRecordId: 98765,
            itemBarcode: '31234567890123',
            details: 'Test details'
        );

        $this->assertEquals(98765, $data['ItemRecordID']);
        $this->assertEquals('31234567890123', $data['ItemBarcode']);
        $this->assertEquals('Test details', $data['Details']);
    }

    public function test_mark_as_sent_success()
    {
        $papiclient = Mockery::mock(PAPIClient::class);
        
        // Mock the fluent chain
        $papiclient->shouldReceive('method')
            ->with('PUT')
            ->andReturnSelf();
        
        $papiclient->shouldReceive('protected')
            ->andReturnSelf();
        
        $papiclient->shouldReceive('uri')
            ->with('notification/1')
            ->andReturnSelf();
        
        $papiclient->shouldReceive('params')
            ->andReturnSelf();
        
        $papiclient->shouldReceive('execRequest')
            ->andReturn([
                'PAPIErrorCode' => 0,
                'ErrorMessage' => null
            ]);

        $service = new NotificationService($papiclient);

        $data = $service->buildNotificationData(
            patronId: 12345,
            patronBarcode: '21234567890123',
            deliveryOptionId: 3,
            deliveryString: '555-0123'
        );

        $result = $service->markAsSent(1, $data);

        $this->assertEquals(0, $result['PAPIErrorCode']);
    }

    public function test_validation_throws_exception_for_missing_fields()
    {
        $papiclient = Mockery::mock(PAPIClient::class);
        $service = new NotificationService($papiclient);

        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage('Required field');

        // Use reflection to call protected method
        $reflection = new \ReflectionClass($service);
        $method = $reflection->getMethod('validateNotificationData');
        $method->setAccessible(true);

        $method->invoke($service, ['PatronID' => 12345]); // Missing required fields
    }
}
```

---

## Best Practices

### 1. Always Use ISO 8601 Date Format

```php
// ✓ Correct
use Carbon\Carbon;
$date = Carbon::now()->toIso8601String();

// ✗ Incorrect
$date = date('Y-m-d H:i:s');
```

### 2. Validate Data Before Sending

```php
protected function validateNotificationData(array $data): void
{
    $required = ['DeliveryOptionID', 'DeliveryString', 'PatronID', 'PatronBarcode'];

    foreach ($required as $field) {
        if (!isset($data[$field]) || empty($data[$field])) {
            throw new \InvalidArgumentException("Required field '{$field}' is missing");
        }
    }
}
```

### 3. Log All Confirmation Attempts

```php
Log::info('Confirming notification', [
    'patron_id' => $patronId,
    'notification_type' => $notificationTypeId,
    'delivery_method' => $deliveryOptionId
]);
```

### 4. Handle Errors Gracefully

```php
try {
    $result = $this->notificationService->markAsSent($typeId, $data);
} catch (\Exception $e) {
    Log::error('Failed to confirm notification', [
        'exception' => $e->getMessage(),
        'data' => $data
    ]);
    
    // Don't throw exception - allow process to continue
    return false;
}
```

### 5. Use Dependency Injection

```php
// ✓ Correct - Injectable
class MyService
{
    public function __construct(
        protected PAPIClient $papiclient,
        protected NotificationService $notificationService
    ) {}
}

// ✗ Incorrect - Hard-coded dependency
class MyService
{
    public function __construct()
    {
        $this->papiclient = new PAPIClient();
    }
}
```

### 6. Implement Retry Logic for Failures

```php
public function confirmWithRetry(int $notificationTypeId, array $data, int $maxAttempts = 3): array
{
    $attempt = 0;
    $lastException = null;

    while ($attempt < $maxAttempts) {
        try {
            return $this->markAsSent($notificationTypeId, $data);
        } catch (\Exception $e) {
            $lastException = $e;
            $attempt++;
            
            if ($attempt < $maxAttempts) {
                // Exponential backoff
                sleep(pow(2, $attempt));
            }
        }
    }

    throw new \RuntimeException(
        "Failed after {$maxAttempts} attempts: " . $lastException->getMessage(),
        0,
        $lastException
    );
}
```

### 7. Queue Long-Running Operations

```php
<?php

namespace App\Jobs;

use App\Services\Polaris\NotificationService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;

class ConfirmNotificationJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable;

    public function __construct(
        public int $notificationTypeId,
        public array $notificationData
    ) {}

    public function handle(NotificationService $notificationService): void
    {
        $notificationService->markAsSent(
            $this->notificationTypeId,
            $this->notificationData
        );
    }
}

// Usage
ConfirmNotificationJob::dispatch($typeId, $data);
```

---

## Common Issues & Troubleshooting

### Authentication Errors

**Problem:** 401 Unauthorized or authentication failures

**Solutions:**
- Verify your `.env` credentials match Polaris admin settings
- Ensure `PAPI_ACCESS_ID` and `PAPI_ACCESS_KEY` are correct
- Check that staff credentials (`PAPI_STAFF`, `PAPI_PASSWORD`) are valid
- Verify the domain is correct if using Active Directory authentication

### PAPIClient Configuration

**Problem:** Base URL or endpoint configuration issues

**Solutions:**
- Verify `PAPI_BASE_URL` includes `/PAPIService/REST`
- Check that `PAPI_PROTECTED_URI` is properly constructed
- Ensure `PAPI_APPID`, `PAPI_ORGID`, and `PAPI_LANGID` match your Polaris setup
- Confirm the endpoint URI doesn't have extra slashes

### Date Format Errors

**Problem:** Invalid date format errors from Polaris

**Solutions:**
- Always use `Carbon::now()->toIso8601String()`
- Never use `date('Y-m-d H:i:s')` format
- Example correct format: `2024-11-13T14:30:00-05:00`

### Missing Required Fields

**Problem:** API returns error about missing required fields

**Solutions:**
- Ensure all required fields are present: `PatronID`, `PatronBarcode`, `DeliveryOptionID`, `DeliveryString`
- Use the `buildNotificationData()` helper method
- Validate data before sending with `validateNotificationData()`

### Fluent Interface Chain Errors

**Problem:** Method chaining doesn't work as expected

**Solutions:**
- Ensure methods are called in correct order: `method()` → `protected()` → `uri()` → `params()` → `execRequest()`
- Don't break the chain with variable assignments mid-way
- Each method should return `$this` except `execRequest()`

---

## Reference Tables

### Delivery Option IDs

| ID | Delivery Option | Description |
|----|-----------------|-------------|
| 1 | Mailing Address | Physical mail notification |
| 2 | Email Address | Email notification |
| 3 | Phone 1 | Primary phone number |
| 4 | Phone 2 | Secondary phone number |
| 5 | Phone 3 | Tertiary phone number |
| 6 | FAX | Fax notification |
| 7 | EDI | Electronic Data Interchange |
| 8 | TXT Messaging | SMS/Text message notification |

### Notification Status IDs

| ID | Status | Description |
|----|--------|-------------|
| 1 | Call completed - Voice | Successfully spoke with person |
| 2 | Call completed - Answering machine | Left message on answering machine |
| 3 | Call not completed - Hang up | Call was hung up |
| 4 | Call not completed - Busy | Line was busy |
| 5 | Call not completed - No answer | No one answered the call |
| 6 | Call not completed - No ring | Phone did not ring |
| 7 | Call failed - No dial tone | No dial tone detected |
| 8 | Call failed - Intercept tones heard | Intercept message detected |
| 9 | Call failed - Probable bad phone number | Invalid phone number |
| 10 | Call failed - Maximum retries exceeded | Too many retry attempts |
| 11 | Call failed - Undetermined error | Unknown error occurred |
| 12 | Email Completed | Email successfully sent |
| 13 | Email Failed - Invalid address | Email address is invalid |
| 14 | Email Failed | Email failed to send |
| 15 | Mail Printed | Physical mail notice printed |
| 16 | Sent | Generic sent status |

**For Shoutbomb Integration:** Use status ID **1** (Call completed - Voice) for successful voice calls, **2** (Call completed - Answering machine) for voicemail deliveries, **12** (Email Completed) for emails, and **16** (Sent) for text messages.

### Notification Type IDs

| ID | Type | Description |
|----|------|-------------|
| 0 | Combined | Combined notification types |
| 1 | 1st Overdue | First overdue notice |
| 2 | Hold | Hold ready for pickup |
| 3 | Cancel | Cancellation notice |
| 4 | Recall | Item recall notice |
| 5 | All | All notification types |
| 6 | Route | Routing notice |
| 7 | Almost overdue/Auto-renew reminder | Courtesy/pre-overdue notice |
| 8 | Fine | Fine/fee notice |
| 9 | Inactive Reminder | Inactive account reminder |
| 10 | Expiration Reminder | Card expiration reminder |
| 11 | Bill | Billing notice |
| 12 | 2nd Overdue | Second overdue notice |
| 13 | 3rd Overdue | Third overdue notice |
| 14 | Serial Claim | Serial publication claim |
| 15 | Polaris Fusion Access Request | Fusion access request response |
| 16 | Course Reserves | Course reserves notice |
| 17 | Borrow-By-Mail Failure | Borrow-by-mail failure notice |
| 18 | 2nd Hold | Second hold notice |
| 19 | Missing Part | Missing part notice |
| 20 | Manual Bill | Manual billing notice |
| 21 | 2nd Fine Notice | Second fine notice |

**Most Common for Shoutbomb:** 
- **Type 2** (Hold) - Item ready for pickup
- **Type 1** (1st Overdue) - Item overdue
- **Type 7** (Almost overdue) - Courtesy reminder
- **Type 11** (Bill) - Outstanding charges

---

## Additional Resources

- [PAPIClient GitHub Repository](https://github.com/blashbrook/papiclient)
- [PAPIClient on Packagist](https://packagist.org/packages/blashbrook/papiclient)
- [Laravel Documentation](https://laravel.com/docs)
- [Polaris API Documentation](https://your-polaris-server.com/PAPIservice) (contact your administrator)

---

*Document Version: 2.1 - Updated for blashbrook/papiclient with actual Polaris reference data*  
*Last Updated: November 13, 2024*

**Data Source:** Reference tables generated from actual Polaris ILS database exports including:
- Polaris_NotificationStatuses.csv
- Polaris_NotificationTypes.csv  
- Polaris_DeliveryOptions.csv
