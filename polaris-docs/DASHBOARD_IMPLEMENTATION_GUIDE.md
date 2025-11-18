# POLARIS NOTICES DASHBOARD - IMPLEMENTATION GUIDE

**Last Updated:** November 18, 2025
**Package:** dcplibrary/notices-dashboard
**Purpose:** Unified web dashboard for monitoring all Polaris ILS notification channels
**Framework:** Laravel 11 + FilamentPHP 3

---

## TABLE OF CONTENTS

- [Overview](#overview)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Prerequisites](#prerequisites)
- [Installation & Setup](#installation--setup)
- [Database Schema](#database-schema)
- [Backend Implementation](#backend-implementation)
- [Frontend Implementation (Filament)](#frontend-implementation-filament)
- [Authentication & Authorization](#authentication--authorization)
- [Statistics & Reports](#statistics--reports)
- [Alerting System](#alerting-system)
- [API Integration](#api-integration)
- [Testing](#testing)
- [Deployment](#deployment)
- [Troubleshooting](#troubleshooting)

---

## OVERVIEW

### What is the Notices Dashboard?

The Notices Dashboard is a **unified monitoring interface** that consolidates notification data from all four Polaris ILS delivery channels:

- 📧 **Email** (notices-polaris package)
- 📮 **Mail** (notices-polaris package)
- 📞 **Voice** (notices-shoutbomb package)
- 📱 **Text/SMS** (notices-shoutbomb package)

### Key Features

✅ **Real-time Monitoring**
- Dashboard updates every 15 minutes via scheduled imports
- Visual charts and statistics
- Color-coded success/failure indicators

✅ **Unified Search & Filtering**
- Search across all notification channels
- Filter by date, patron, type, status, organization
- Export filtered results to CSV/Excel

✅ **Failure Tracking**
- Automatic failure detection
- Email alerts for critical failures
- Detailed failure reasons and suggestions

✅ **Statistics & Reports**
- Daily, weekly, monthly volume trends
- Success rates by channel and type
- Branch/organization breakdowns
- Exportable reports

✅ **Patron Management**
- View patron notification preferences
- See notification history per patron
- Identify contact information issues

---

## ARCHITECTURE

### System Design

```
┌──────────────────────────────────────────────────────────────┐
│                     POLARIS NOTICES DASHBOARD                 │
│                      (Laravel 11 + Filament 3)                │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              FILAMENT ADMIN PANEL                    │    │
│  │  ┌─────────────┬─────────────┬──────────────────┐   │    │
│  │  │  Dashboard  │  Resources  │  Custom Pages    │   │    │
│  │  │   Widgets   │   (CRUD)    │  (Reports/Stats) │   │    │
│  │  └─────────────┴─────────────┴──────────────────┘   │    │
│  └─────────────────────────────────────────────────────┘    │
│                          ↑                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │            UNIFIED SERVICE LAYER                     │    │
│  │  • UnifiedNotificationService                        │    │
│  │  • NotificationStatisticsService                     │    │
│  │  • FailureReportService                              │    │
│  │  • ExportService                                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                          ↑                                    │
│  ┌──────────────────────┬──────────────────────────────┐    │
│  │  notices-polaris     │  notices-shoutbomb           │    │
│  │  Package             │  Package                     │    │
│  ├──────────────────────┼──────────────────────────────┤    │
│  │  • EmailNotification │  • VoiceNotification         │    │
│  │  • MailNotification  │  • TextNotification          │    │
│  │  • EmailPatron       │  • VoicePatron               │    │
│  │  • MailPatron        │  • TextPatron                │    │
│  └──────────────────────┴──────────────────────────────┘    │
│                          ↑                                    │
│  ┌──────────────────────┬──────────────────────────────┐    │
│  │  MySQL Database      │  MySQL Database              │    │
│  │  (polaris_* tables)  │  (shoutbomb_* tables)        │    │
│  └──────────────────────┴──────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
             ↑                           ↑
    ┌────────┴────────┐         ┌───────┴────────┐
    │  Polaris ILS    │         │  Shoutbomb     │
    │  SQL Database   │         │  FTP Server    │
    └─────────────────┘         └────────────────┘
```

### Package Structure Decision

**Recommended Approach:** **Standalone Laravel Application with Filament**

**Why?**
- Faster development with FilamentPHP's pre-built components
- Professional admin panel interface out-of-the-box
- Easy to customize and extend
- Complete authentication and authorization built-in
- Better for teams unfamiliar with building Laravel packages

**Alternative:** Laravel package (`dcplibrary/notices-dashboard`)
- Use if you need to distribute to multiple installations
- Requires host Laravel application
- More complex setup

---

## TECHNOLOGY STACK

### Core Framework
- **Laravel 11.x** - PHP framework
- **PHP 8.3+** - Programming language
- **MySQL 8.0+** - Database

### Admin Panel
- **FilamentPHP 3.x** - Admin panel framework
  - Built on Livewire + Alpine.js + Tailwind CSS
  - Pre-built CRUD resources
  - Charts and widgets
  - Form builder
  - Table builder

### Supporting Packages
- **Laravel Sanctum** - API authentication (optional)
- **Laravel Excel** - Export to Excel/CSV
- **Laravel Notifications** - Email alerting
- **Spatie Laravel Permission** - Role-based access control (if needed)

### Frontend (via Filament)
- **Livewire 3.x** - Reactive components
- **Alpine.js** - Minimal JavaScript framework
- **Tailwind CSS** - Utility-first CSS framework
- **Chart.js** - Charts and graphs

---

## PREREQUISITES

### System Requirements

```bash
# PHP 8.3 or higher
php -v

# Composer 2.x
composer --version

# MySQL 8.0+
mysql --version

# Node.js 18+ and NPM (for asset compilation)
node -v
npm -v
```

### Required PHP Extensions

```bash
php -m | grep -E "(pdo|pdo_mysql|mbstring|xml|openssl|json|curl|zip|gd|bcmath)"
```

### Package Dependencies

Both notices packages must be installed:
```bash
composer require dcplibrary/notices-polaris
composer require dcplibrary/notices-shoutbomb
```

---

## INSTALLATION & SETUP

### Step 1: Create Laravel Application

```bash
# Create new Laravel 11 application
composer create-project laravel/laravel notices-dashboard
cd notices-dashboard

# Or use Laravel installer
laravel new notices-dashboard
cd notices-dashboard
```

### Step 2: Install FilamentPHP

```bash
# Install Filament Panel
composer require filament/filament:"^3.0"

# Install Filament
php artisan filament:install --panels

# Create admin user
php artisan make:filament-user
```

### Step 3: Install Notices Packages

```bash
# Install both notices packages
composer require dcplibrary/notices-polaris
composer require dcplibrary/notices-shoutbomb

# Run migrations
php artisan migrate

# Publish configurations (optional)
php artisan vendor:publish --tag=notices-polaris-config
php artisan vendor:publish --tag=notices-shoutbomb-config
```

### Step 4: Install Supporting Packages

```bash
# Excel export support
composer require maatwebsite/excel

# Charts (via Filament plugin)
composer require filament/filament-widgets
```

### Step 5: Configure Environment

```env
# .env

APP_NAME="Polaris Notices Dashboard"
APP_URL=https://notices.yourlibrary.org

# Database - Local Laravel database
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=notices_dashboard
DB_USERNAME=root
DB_PASSWORD=

# Polaris Database Connection (read-only)
POLARIS_DB_HOST=polaris-sql.library.local
POLARIS_DB_PORT=1433
POLARIS_DB_NAME=Polaris
POLARIS_DB_USERNAME=polaris_readonly
POLARIS_DB_PASSWORD=

# Shoutbomb FTP Configuration
SHOUTBOMB_FTP_HOST=ftp.library.local
SHOUTBOMB_FTP_USERNAME=shoutbomb
SHOUTBOMB_FTP_PASSWORD=
SHOUTBOMB_FTP_BASE_PATH=/shoutbomb/logs

# Import Settings
POLARIS_IMPORT_DAYS=7
SHOUTBOMB_IMPORT_DAYS=7

# Mail Configuration (for alerts)
MAIL_MAILER=smtp
MAIL_HOST=smtp.library.org
MAIL_PORT=587
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_FROM_ADDRESS=notices@library.org
MAIL_FROM_NAME="${APP_NAME}"

# Alerting
ALERT_EMAIL=libraryadmin@library.org
ALERT_FAILURE_THRESHOLD=5  # Alert if >5% failure rate
```

### Step 6: Set Up Scheduler

```bash
# Add to crontab
* * * * * cd /path/to/notices-dashboard && php artisan schedule:run >> /dev/null 2>&1
```

In `app/Console/Kernel.php` or `routes/console.php`:

```php
use Illuminate\Support\Facades\Schedule;

// Import notifications every 15 minutes
Schedule::command('polaris:import-notifications --days=1')
    ->everyFifteenMinutes()
    ->withoutOverlapping();

Schedule::command('shoutbomb:import-all')
    ->everyFifteenMinutes()
    ->withoutOverlapping();

// Daily patron sync
Schedule::command('polaris:import-email-patrons')->daily();
Schedule::command('polaris:import-mail-patrons')->daily();

// Daily failure summary email
Schedule::command('notifications:send-daily-summary')
    ->dailyAt('08:00');
```

---

## DATABASE SCHEMA

### Dashboard-Specific Tables

The dashboard doesn't create new tables - it uses tables from the notices packages:

**From notices-polaris:**
- `polaris_notification_log` - Email and mail notifications
- `polaris_email_patrons` - Email patron contacts
- `polaris_mail_patrons` - Mail patron addresses
- `polaris_notification_statuses` - Status lookup
- `polaris_notification_types` - Type lookup

**From notices-shoutbomb:**
- `shoutbomb_holds` - Hold notifications (voice/text)
- `shoutbomb_overdue` - Overdue notifications (voice/text)
- `shoutbomb_renew` - Renewal reminders (voice/text)
- `shoutbomb_voice_patrons` - Voice patron contacts
- `shoutbomb_text_patrons` - Text patron contacts

**Optional Dashboard Tables:**

If you want to track dashboard users separately:

```php
// database/migrations/create_dashboard_users_table.php
Schema::create('dashboard_users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->string('password');
    $table->string('role')->default('viewer'); // admin, manager, viewer
    $table->timestamps();
});
```

---

## BACKEND IMPLEMENTATION

### Directory Structure

```
app/
├── Filament/
│   ├── Resources/
│   │   ├── NotificationResource.php
│   │   ├── PatronResource.php
│   │   └── FailureResource.php
│   ├── Pages/
│   │   ├── Dashboard.php
│   │   ├── Statistics.php
│   │   └── Reports.php
│   └── Widgets/
│       ├── NotificationStatsWidget.php
│       ├── ChannelChartWidget.php
│       └── RecentFailuresWidget.php
├── Services/
│   ├── UnifiedNotificationService.php
│   ├── NotificationStatisticsService.php
│   ├── FailureReportService.php
│   └── ExportService.php
├── Models/  (use package models directly)
└── Console/
    └── Commands/
        └── SendDailySummary.php
```

### Service Layer Implementation

#### UnifiedNotificationService.php

```php
<?php

namespace App\Services;

use Dcplibrary\NoticesPolaris\Models\NotificationLog;
use Dcplibrary\NoticesShoutbomb\Models\HoldsExport;
use Dcplibrary\NoticesShoutbomb\Models\OverdueExport;
use Illuminate\Database\Eloquent\Collection;
use Illuminate\Support\Facades\DB;

class UnifiedNotificationService
{
    /**
     * Get all notifications across all channels
     */
    public function getAllNotifications(array $filters = []): Collection
    {
        // Email notifications
        $emailNotifications = $this->getEmailNotifications($filters)
            ->map(fn($n) => $this->normalizeNotification($n, 'email'));

        // Mail notifications
        $mailNotifications = $this->getMailNotifications($filters)
            ->map(fn($n) => $this->normalizeNotification($n, 'mail'));

        // Voice notifications
        $voiceNotifications = $this->getVoiceNotifications($filters)
            ->map(fn($n) => $this->normalizeNotification($n, 'voice'));

        // Text notifications
        $textNotifications = $this->getTextNotifications($filters)
            ->map(fn($n) => $this->normalizeNotification($n, 'text'));

        // Merge all notifications
        return $emailNotifications
            ->merge($mailNotifications)
            ->merge($voiceNotifications)
            ->merge($textNotifications)
            ->sortByDesc('notification_date');
    }

    /**
     * Get email notifications from polaris package
     */
    protected function getEmailNotifications(array $filters = [])
    {
        $query = NotificationLog::where('delivery_option_id', 2);

        if (!empty($filters['date_from'])) {
            $query->where('notification_date_time', '>=', $filters['date_from']);
        }

        if (!empty($filters['date_to'])) {
            $query->where('notification_date_time', '<=', $filters['date_to']);
        }

        if (!empty($filters['patron_id'])) {
            $query->where('patron_id', $filters['patron_id']);
        }

        if (!empty($filters['notification_type_id'])) {
            $query->where('notification_type_id', $filters['notification_type_id']);
        }

        return $query->get();
    }

    /**
     * Get mail notifications from polaris package
     */
    protected function getMailNotifications(array $filters = [])
    {
        $query = NotificationLog::where('delivery_option_id', 1);

        // Apply same filters as email
        // ... (similar to getEmailNotifications)

        return $query->get();
    }

    /**
     * Get voice notifications from shoutbomb package
     */
    protected function getVoiceNotifications(array $filters = [])
    {
        $query = HoldsExport::where('delivery_channel', 'voice');

        // Apply filters
        // ... (similar to getEmailNotifications)

        return $query->get();
    }

    /**
     * Get text notifications from shoutbomb package
     */
    protected function getTextNotifications(array $filters = [])
    {
        $query = HoldsExport::where('delivery_channel', 'text');

        // Apply filters
        // ...

        return $query->get();
    }

    /**
     * Normalize notification data across channels
     */
    protected function normalizeNotification($notification, string $channel): array
    {
        return [
            'id' => "{$channel}:{$notification->id}",
            'channel' => $channel,
            'patron_id' => $notification->patron_id,
            'patron_barcode' => $notification->patron_barcode,
            'notification_type_id' => $notification->notification_type_id ?? null,
            'notification_date' => $notification->notification_date_time ?? $notification->export_date,
            'status' => $this->normalizeStatus($notification, $channel),
            'title' => $notification->title ?? $notification->browse_title ?? '',
            'delivery_destination' => $this->getDeliveryDestination($notification, $channel),
        ];
    }

    /**
     * Normalize status across different channel formats
     */
    protected function normalizeStatus($notification, string $channel): string
    {
        if ($channel === 'email' || $channel === 'mail') {
            // Polaris status codes
            return match($notification->notification_status_id) {
                12 => 'sent',      // Email Completed
                13, 14 => 'failed', // Email Failed
                15 => 'sent',      // Mail Printed
                default => 'unknown'
            };
        }

        if ($channel === 'voice' || $channel === 'text') {
            // Shoutbomb uses import_date as proxy for "sent"
            return $notification->delivery_status ?? 'exported';
        }

        return 'unknown';
    }

    /**
     * Get delivery destination (email, phone, or address)
     */
    protected function getDeliveryDestination($notification, string $channel): string
    {
        return match($channel) {
            'email' => $notification->delivery_string ?? '',
            'mail' => $notification->delivery_string ?? '',
            'voice', 'text' => $notification->phone_number ?? '',
            default => ''
        };
    }
}
```

#### NotificationStatisticsService.php

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\DB;
use Carbon\Carbon;

class NotificationStatisticsService
{
    /**
     * Get summary statistics across all channels
     */
    public function getSummaryStats(string $dateFrom = null, string $dateTo = null): array
    {
        $dateFrom = $dateFrom ?? Carbon::now()->subDays(30);
        $dateTo = $dateTo ?? Carbon::now();

        return [
            'email' => $this->getChannelStats('email', $dateFrom, $dateTo),
            'mail' => $this->getChannelStats('mail', $dateFrom, $dateTo),
            'voice' => $this->getChannelStats('voice', $dateFrom, $dateTo),
            'text' => $this->getChannelStats('text', $dateFrom, $dateTo),
            'total' => $this->getTotalStats($dateFrom, $dateTo),
        ];
    }

    /**
     * Get statistics for a specific channel
     */
    protected function getChannelStats(string $channel, $dateFrom, $dateTo): array
    {
        if ($channel === 'email' || $channel === 'mail') {
            return $this->getPolarisChannelStats($channel, $dateFrom, $dateTo);
        } else {
            return $this->getShoutbombChannelStats($channel, $dateFrom, $dateTo);
        }
    }

    /**
     * Get statistics for Polaris channels (email/mail)
     */
    protected function getPolarisChannelStats(string $channel, $dateFrom, $dateTo): array
    {
        $deliveryOptionId = $channel === 'email' ? 2 : 1;

        $stats = DB::table('polaris_notification_log')
            ->select([
                DB::raw('COUNT(*) as total'),
                DB::raw('SUM(CASE WHEN notification_status_id IN (12, 15) THEN 1 ELSE 0 END) as successful'),
                DB::raw('SUM(CASE WHEN notification_status_id IN (13, 14) THEN 1 ELSE 0 END) as failed'),
            ])
            ->where('delivery_option_id', $deliveryOptionId)
            ->whereBetween('notification_date_time', [$dateFrom, $dateTo])
            ->first();

        return [
            'total' => $stats->total ?? 0,
            'successful' => $stats->successful ?? 0,
            'failed' => $stats->failed ?? 0,
            'success_rate' => $stats->total > 0
                ? round(($stats->successful / $stats->total) * 100, 2)
                : 0,
        ];
    }

    /**
     * Get statistics for Shoutbomb channels (voice/text)
     */
    protected function getShoutbombChannelStats(string $channel, $dateFrom, $dateTo): array
    {
        // Query shoutbomb tables
        $stats = DB::table('shoutbomb_holds')
            ->select([
                DB::raw('COUNT(*) as total'),
                DB::raw('COUNT(*) as successful'), // All imports considered sent
                DB::raw('0 as failed'), // Failures tracked separately
            ])
            ->where('delivery_channel', $channel)
            ->whereBetween('export_date', [$dateFrom, $dateTo])
            ->first();

        return [
            'total' => $stats->total ?? 0,
            'successful' => $stats->successful ?? 0,
            'failed' => $stats->failed ?? 0,
            'success_rate' => 100, // Adjust if failure tracking available
        ];
    }

    /**
     * Get daily notification volumes
     */
    public function getDailyVolumes(string $channel = null, $dateFrom = null, $dateTo = null): array
    {
        $dateFrom = $dateFrom ?? Carbon::now()->subDays(30);
        $dateTo = $dateTo ?? Carbon::now();

        // Implementation for daily volume trends
        // Returns array of dates with counts
        // ...
    }
}
```

---

## FRONTEND IMPLEMENTATION (FILAMENT)

### Dashboard Widgets

#### NotificationStatsWidget.php

```php
<?php

namespace App\Filament\Widgets;

use Filament\Widgets\StatsOverviewWidget as BaseWidget;
use Filament\Widgets\StatsOverviewWidget\Stat;
use App\Services\NotificationStatisticsService;

class NotificationStatsWidget extends BaseWidget
{
    protected static ?string $pollingInterval = '30s';

    protected function getStats(): array
    {
        $service = app(NotificationStatisticsService::class);
        $stats = $service->getSummaryStats();

        return [
            Stat::make('Email Notifications', number_format($stats['email']['total']))
                ->description($stats['email']['success_rate'] . '% success rate')
                ->descriptionIcon('heroicon-m-envelope')
                ->color($stats['email']['success_rate'] >= 95 ? 'success' : 'warning'),

            Stat::make('Mail Notifications', number_format($stats['mail']['total']))
                ->description($stats['mail']['success_rate'] . '% success rate')
                ->descriptionIcon('heroicon-m-inbox')
                ->color('success'),

            Stat::make('Voice Calls', number_format($stats['voice']['total']))
                ->description($stats['voice']['success_rate'] . '% success rate')
                ->descriptionIcon('heroicon-m-phone')
                ->color($stats['voice']['success_rate'] >= 90 ? 'success' : 'warning'),

            Stat::make('Text Messages', number_format($stats['text']['total']))
                ->description($stats['text']['success_rate'] . '% success rate')
                ->descriptionIcon('heroicon-m-chat-bubble-left')
                ->color($stats['text']['success_rate'] >= 95 ? 'success' : 'warning'),
        ];
    }
}
```

#### ChannelChartWidget.php

```php
<?php

namespace App\Filament\Widgets;

use Filament\Widgets\ChartWidget;
use App\Services\NotificationStatisticsService;

class ChannelChartWidget extends ChartWidget
{
    protected static ?string $heading = 'Daily Notification Volume (Last 30 Days)';

    protected function getData(): array
    {
        $service = app(NotificationStatisticsService::class);
        $dailyData = $service->getDailyVolumes();

        return [
            'datasets' => [
                [
                    'label' => 'Email',
                    'data' => array_column($dailyData['email'], 'count'),
                    'borderColor' => 'rgb(59, 130, 246)',
                    'backgroundColor' => 'rgba(59, 130, 246, 0.1)',
                ],
                [
                    'label' => 'Mail',
                    'data' => array_column($dailyData['mail'], 'count'),
                    'borderColor' => 'rgb(16, 185, 129)',
                    'backgroundColor' => 'rgba(16, 185, 129, 0.1)',
                ],
                [
                    'label' => 'Voice',
                    'data' => array_column($dailyData['voice'], 'count'),
                    'borderColor' => 'rgb(249, 115, 22)',
                    'backgroundColor' => 'rgba(249, 115, 22, 0.1)',
                ],
                [
                    'label' => 'Text',
                    'data' => array_column($dailyData['text'], 'count'),
                    'borderColor' => 'rgb(168, 85, 247)',
                    'backgroundColor' => 'rgba(168, 85, 247, 0.1)',
                ],
            ],
            'labels' => array_column($dailyData['dates'], 'date'),
        ];
    }

    protected function getType(): string
    {
        return 'line';
    }
}
```

### Filament Resources

#### NotificationResource.php

```php
<?php

namespace App\Filament\Resources;

use App\Filament\Resources\NotificationResource\Pages;
use App\Services\UnifiedNotificationService;
use Filament\Forms;
use Filament\Resources\Resource;
use Filament\Tables;
use Filament\Tables\Table;

class NotificationResource extends Resource
{
    protected static ?string $model = null; // Virtual resource

    protected static ?string $navigationIcon = 'heroicon-o-bell';

    protected static ?string $navigationLabel = 'All Notifications';

    public static function table(Table $table): Table
    {
        return $table
            ->query(function() {
                // Use service to get unified notifications
                $service = app(UnifiedNotificationService::class);
                return collect($service->getAllNotifications());
            })
            ->columns([
                Tables\Columns\BadgeColumn::make('channel')
                    ->colors([
                        'primary' => 'email',
                        'success' => 'mail',
                        'warning' => 'voice',
                        'info' => 'text',
                    ])
                    ->icons([
                        'heroicon-m-envelope' => 'email',
                        'heroicon-m-inbox' => 'mail',
                        'heroicon-m-phone' => 'voice',
                        'heroicon-m-chat-bubble-left' => 'text',
                    ]),

                Tables\Columns\TextColumn::make('patron_barcode')
                    ->label('Patron')
                    ->searchable(),

                Tables\Columns\TextColumn::make('title')
                    ->limit(50)
                    ->searchable(),

                Tables\Columns\TextColumn::make('notification_date')
                    ->label('Date')
                    ->dateTime()
                    ->sortable(),

                Tables\Columns\BadgeColumn::make('status')
                    ->colors([
                        'success' => 'sent',
                        'danger' => 'failed',
                        'warning' => 'pending',
                    ]),

                Tables\Columns\TextColumn::make('delivery_destination')
                    ->label('Sent To')
                    ->limit(30),
            ])
            ->filters([
                Tables\Filters\SelectFilter::make('channel')
                    ->options([
                        'email' => 'Email',
                        'mail' => 'Mail',
                        'voice' => 'Voice',
                        'text' => 'Text',
                    ]),

                Tables\Filters\SelectFilter::make('status')
                    ->options([
                        'sent' => 'Sent',
                        'failed' => 'Failed',
                        'pending' => 'Pending',
                    ]),

                Tables\Filters\Filter::make('date_range')
                    ->form([
                        Forms\Components\DatePicker::make('date_from'),
                        Forms\Components\DatePicker::make('date_to'),
                    ]),
            ])
            ->actions([
                Tables\Actions\ViewAction::make(),
            ])
            ->bulkActions([
                Tables\Actions\ExportBulkAction::make(),
            ]);
    }

    public static function getPages(): array
    {
        return [
            'index' => Pages\ListNotifications::route('/'),
        ];
    }
}
```

### Custom Pages

#### Statistics Page

```php
<?php

namespace App\Filament\Pages;

use Filament\Pages\Page;
use App\Services\NotificationStatisticsService;

class Statistics extends Page
{
    protected static ?string $navigationIcon = 'heroicon-o-chart-bar';

    protected static string $view = 'filament.pages.statistics';

    protected static ?string $navigationLabel = 'Statistics';

    public $stats;

    public function mount()
    {
        $service = app(NotificationStatisticsService::class);
        $this->stats = $service->getSummaryStats();
    }
}
```

View file: `resources/views/filament/pages/statistics.blade.php`

```blade
<x-filament-panels::page>
    <div class="space-y-6">
        <!-- Summary Cards -->
        <div class="grid grid-cols-1 md:grid-cols-4 gap-4">
            <x-filament::card>
                <h3 class="text-lg font-semibold">Email</h3>
                <p class="text-3xl font-bold">{{ number_format($stats['email']['total']) }}</p>
                <p class="text-sm text-gray-500">{{ $stats['email']['success_rate'] }}% success</p>
            </x-filament::card>

            <x-filament::card>
                <h3 class="text-lg font-semibold">Mail</h3>
                <p class="text-3xl font-bold">{{ number_format($stats['mail']['total']) }}</p>
                <p class="text-sm text-gray-500">{{ $stats['mail']['success_rate'] }}% success</p>
            </x-filament::card>

            <x-filament::card>
                <h3 class="text-lg font-semibold">Voice</h3>
                <p class="text-3xl font-bold">{{ number_format($stats['voice']['total']) }}</p>
                <p class="text-sm text-gray-500">{{ $stats['voice']['success_rate'] }}% success</p>
            </x-filament::card>

            <x-filament::card>
                <h3 class="text-lg font-semibold">Text</h3>
                <p class="text-3xl font-bold">{{ number_format($stats['text']['total']) }}</p>
                <p class="text-sm text-gray-500">{{ $stats['text']['success_rate'] }}% success</p>
            </x-filament::card>
        </div>

        <!-- Charts -->
        <livewire:channel-chart-widget />
    </div>
</x-filament-panels::page>
```

---

## AUTHENTICATION & AUTHORIZATION

### Setup Filament Authentication

FilamentPHP includes authentication out-of-the-box:

```bash
# Create admin user
php artisan make:filament-user
```

### Role-Based Access Control (Optional)

If you need multiple user roles:

```bash
composer require spatie/laravel-permission
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
php artisan migrate
```

Define roles in `app/Filament/Resources/*`:

```php
public static function canViewAny(): bool
{
    return auth()->user()->can('view_notifications');
}
```

---

## STATISTICS & REPORTS

### Report Types

1. **Daily Summary Report**
   - Total notifications by channel
   - Success/failure rates
   - Top failures

2. **Weekly Trend Report**
   - 7-day rolling averages
   - Channel comparison
   - Volume trends

3. **Monthly Summary**
   - Month-over-month comparison
   - Failure analysis
   - Patron contact preference changes

### Export Functionality

```php
// app/Services/ExportService.php

public function exportToExcel(array $filters)
{
    return Excel::download(
        new NotificationsExport($filters),
        'notifications-' . now()->format('Y-m-d') . '.xlsx'
    );
}
```

---

## ALERTING SYSTEM

### Email Alerts

```php
// app/Console/Commands/SendDailySummary.php

public function handle()
{
    $service = app(NotificationStatisticsService::class);
    $stats = $service->getSummaryStats();

    Mail::to(config('app.alert_email'))
        ->send(new DailySummaryMail($stats));
}
```

### Failure Threshold Alerts

```php
// Check if failure rate exceeds threshold
if ($stats['email']['success_rate'] < 95) {
    Mail::to(config('app.alert_email'))
        ->send(new HighFailureRateAlert('email', $stats['email']));
}
```

---

## API INTEGRATION

### Optional REST API

If you implemented the OpenAPI spec:

```bash
# Install Laravel Sanctum
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

Register routes in `routes/api.php`:

```php
use App\Http\Controllers\Api\NotificationController;

Route::middleware('auth:sanctum')->group(function () {
    Route::get('/notifications', [NotificationController::class, 'index']);
    Route::get('/notifications/{id}', [NotificationController::class, 'show']);
    Route::get('/statistics/summary', [StatisticsController::class, 'summary']);
});
```

---

## TESTING

### Feature Tests

```php
// tests/Feature/NotificationDashboardTest.php

public function test_dashboard_loads()
{
    $user = User::factory()->create();

    $this->actingAs($user)
        ->get('/admin')
        ->assertOk();
}

public function test_can_view_notifications()
{
    $user = User::factory()->create();

    $this->actingAs($user)
        ->get('/admin/notifications')
        ->assertOk();
}
```

### Service Tests

```php
public function test_unified_service_returns_all_channels()
{
    $service = new UnifiedNotificationService();
    $notifications = $service->getAllNotifications();

    $this->assertGreaterThan(0, $notifications->count());
    $this->assertContains('email', $notifications->pluck('channel'));
}
```

---

## DEPLOYMENT

### Production Checklist

- [ ] Configure production database connections
- [ ] Set up scheduled tasks (cron)
- [ ] Configure email for alerts
- [ ] Enable HTTPS
- [ ] Optimize Laravel (`php artisan optimize`)
- [ ] Set up monitoring (Sentry, etc.)
- [ ] Configure backups
- [ ] Set proper file permissions
- [ ] Enable rate limiting
- [ ] Configure cache (Redis recommended)

### Server Requirements

```nginx
# Nginx configuration example
server {
    listen 80;
    server_name notices.yourlibrary.org;
    root /var/www/notices-dashboard/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

---

## TROUBLESHOOTING

### Common Issues

**Issue:** Dashboard shows no data
- **Solution:** Run import commands manually, check database connections

**Issue:** Slow dashboard performance
- **Solution:** Add database indexes, enable query caching

**Issue:** Scheduler not running
- **Solution:** Verify crontab, check Laravel logs

**Issue:** High memory usage
- **Solution:** Chunk large queries, enable pagination

---

## CONCLUSION

This implementation guide provides a complete roadmap for building the Polaris Notices Dashboard using Laravel and FilamentPHP. The recommended approach leverages FilamentPHP's powerful admin panel framework to rapidly build a professional, full-featured dashboard.

**Next Steps:**
1. Make architectural decisions (see REMAINING_TASKS_AND_DECISIONS.md)
2. Set up development environment
3. Install Laravel + Filament
4. Install notices packages
5. Implement service layer
6. Build Filament resources and widgets
7. Test and deploy

**Estimated Timeline:** 7-10 days for MVP with core features

---

**Last Updated:** November 18, 2025
**Version:** 1.0.0
**Status:** Ready for Implementation
