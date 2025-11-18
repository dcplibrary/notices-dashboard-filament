# SHOUTBOMB & POLARIS INTEGRATION DOCUMENTATION INDEX

**Last Updated:** November 14, 2025  
**System Owner:** Brian Lashbrook, Daviess County Public Library  
**Purpose:** Central index for all ShoutBomb automated patron notification system documentation

---

## TABLE OF CONTENTS

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Documentation Structure](#documentation-structure)
- [Quick Reference Guide](#quick-reference-guide)
- [Data Flow Diagram](#data-flow-diagram)
- [Troubleshooting Index](#troubleshooting-index)
- [API Integration Resources](#api-integration-resources)

---

## OVERVIEW

### What is ShoutBomb?

ShoutBomb is a third-party automated notification service that delivers hold, overdue, and renewal reminders to library patrons via voice calls and text messages. The system integrates with the Polaris ILS through daily FTP file exchanges and provides delivery confirmation reports.

### System Components

1. **Polaris ILS** - Library management system (source of truth)
2. **ShoutBomb Service** - External notification delivery platform
3. **FTP Exchange** - File transfer mechanism
4. **Windows Scheduled Tasks** - Automation
5. **Email Reports** - Delivery confirmations and failure reports
6. **Polaris API** (future) - Two-way integration for delivery confirmations

### Key Workflows

- **Overnight Processing** (1:30am - 4:00am): Conflict resolution and patron list exports
- **Morning Exports** (8:00am - 9:00am): Hold and overdue notifications
- **Afternoon Exports** (1:00pm - 5:00pm): Additional hold notifications
- **Incoming Reports** (Daily): Failure reports and delivery statistics from ShoutBomb

---

## SYSTEM ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────┐
│                        POLARIS ILS                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Patrons     │  │ Circulation  │  │ Notification │          │
│  │  Database    │  │   Records    │  │    Queue     │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                   │
│         └──────────────────┴──────────────────┘                   │
│                            │                                       │
│                            ▼                                       │
│              ┌─────────────────────────┐                          │
│              │   SQL Export Scripts    │                          │
│              │  (Custom & Standard)    │                          │
│              └────────────┬────────────┘                          │
└───────────────────────────┼────────────────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │   Local File System     │
              │  C:\shoutbomb\ftp\*     │
              └────────────┬────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │   WinSCP FTP Upload     │
              │  (Automated Transfer)   │
              └────────────┬────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SHOUTBOMB SERVICE                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Patron     │  │   Voice      │  │    Text      │          │
│  │  Processing  │  │  Dialing     │  │   Messaging  │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                   │
│         └──────────────────┴──────────────────┘                   │
│                            │                                       │
│                            ▼                                       │
│              ┌─────────────────────────┐                          │
│              │   Delivery Attempts     │                          │
│              └────────────┬────────────┘                          │
└───────────────────────────┼────────────────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │   Email Reports to      │
              │   Library Staff         │
              └─────────────────────────┘
```

---

## DOCUMENTATION STRUCTURE

### Core Data Source Documentation

#### Exports TO ShoutBomb (Outgoing)

| Document | Description | Frequency | File Location |
|----------|-------------|-----------|---------------|
| [SHOUTBOMB_VOICE_PATRONS.md](SHOUTBOMB_VOICE_PATRONS.md) | Patrons opted for voice notifications | Daily 4:00am | `/voice_patrons/voice_patrons.txt` |
| [SHOUTBOMB_TEXT_PATRONS.md](SHOUTBOMB_TEXT_PATRONS.md) | Patrons opted for text notifications | Daily 4:00am | `/text_patrons/text_patrons.txt` |
| [SHOUTBOMB_HOLDS_EXPORT.md](SHOUTBOMB_HOLDS_EXPORT.md) | Hold ready for pickup notifications | 4x daily | `/holds/holds.txt` |
| [SHOUTBOMB_OVERDUE_EXPORT.md](SHOUTBOMB_OVERDUE_EXPORT.md) | Overdue item notifications | Daily 8:04am | `/overdue/overdue.txt` |
| [SHOUTBOMB_RENEW_EXPORT.md](SHOUTBOMB_RENEW_EXPORT.md) | Renewal reminder notifications | Daily 8:06am | `/renew/renew.txt` |

#### Reports FROM ShoutBomb (Incoming)

| Document | Description | Frequency | Delivery Method |
|----------|-------------|-----------|-----------------|
| [SHOUTBOMB_REPORTS_INCOMING.md](SHOUTBOMB_REPORTS_INCOMING.md) | Failure reports and statistics | Daily + Monthly | Email |

#### Polaris Standard Exports

| Document | Description | Frequency | File Location |
|----------|-------------|-----------|---------------|
| [POLARIS_PHONE_NOTICES.md](POLARIS_PHONE_NOTICES.md) | Native Polaris phone notification export (25 fields) | Daily | Standard Polaris export |

### Laravel Package Documentation

| Document | Description | Purpose |
|----------|-------------|---------|
| [LARAVEL_PACKAGE_SETUP.md](../LARAVEL_PACKAGE_SETUP.md) | Shoutbomb Monitor Laravel package setup guide | FTP/SMB configuration, database overrides, installation |

### API Integration Documentation

| Document | Description | Purpose |
|----------|-------------|---------|
| [Polaris_Notification_Guide_PAPIClient.md](Polaris_Notification_Guide_PAPIClient.md) | Laravel package for Polaris API integration | Future: Delivery confirmations |
| [Polaris-API-swagger.json](Polaris-API-swagger.json) | Complete Polaris API specification | API endpoint reference |

### Templates & Reference

| Document | Description | Purpose |
|----------|-------------|---------|
| [DATA_SOURCE_TEMPLATE.md](DATA_SOURCE_TEMPLATE.md) | Template for documenting new data sources | Standardization |

---

## QUICK REFERENCE GUIDE

### File Naming Patterns

| Export Type | File Pattern | Example |
|-------------|-------------|----------|
| Voice Patrons | `voice_patrons.txt` | `voice_patrons_submitted_20251114_040001.txt` |
| Text Patrons | `text_patrons.txt` | `text_patrons_submitted_20251114_040002.txt` |
| Holds | `holds.txt` | `holds_submitted_20251114_080001.txt` |
| Overdue | `overdue.txt` | `overdue_submitted_20251114_080401.txt` |
| Renew | `renew.txt` | `renew_submitted_20251114_080601.txt` |

### Field Delimiters

| Export Type | Delimiter | Header Row? | Example |
|-------------|-----------|-------------|---------|
| Voice Patrons | Pipe (`\|`) | No | `5551234567\|23307000001234` |
| Text Patrons | Pipe (`\|`) | No | `5551234567\|23307000001234` |
| Holds | Pipe (`\|`) | No | `23307000001234\|...` |
| Overdue | Pipe (`\|`) | No | `100001\|33307000012345\|...` |
| Renew | Pipe (`\|`) | No | `23307000001234\|...` |
| Phone Notices | Comma (`,`) | Yes | `"V","eng","2",...` |

### Critical Phone Number Rules

**MUST NEVER OVERLAP:**
- A phone number can appear multiple times on voice_patrons.txt (same phone, multiple patrons)
- A phone number can appear multiple times on text_patrons.txt (same phone, multiple patrons)
- **A phone number CANNOT appear on BOTH lists** (conflict resolution prevents this)

### Notification Type Mappings

| Polaris NotificationTypeID | Description | Used by ShoutBomb? | Export File |
|---------------------------|-------------|-------------------|-------------|
| 2 | Hold ready for pickup | Yes | holds.txt |
| 1 | 1st Overdue | Yes | overdue.txt |
| 7 | Almost overdue/Renewal reminder | Yes | renew.txt |
| 8 | Fine | Yes | overdue.txt |
| 11 | Bill | Yes | overdue.txt |
| 12 | 2nd Overdue | Yes | overdue.txt |
| 13 | 3rd Overdue | Yes | overdue.txt |

### Delivery Option IDs

| DeliveryOptionID | Description | Used by ShoutBomb? |
|------------------|-------------|-------------------|
| 1 | Mailing Address | No |
| 2 | Email Address | No |
| 3 | Phone 1 (Voice) | Yes |
| 4 | Phone 2 (Voice) | No |
| 5 | Phone 3 (Voice) | No |
| 6 | FAX | No |
| 7 | EDI | No |
| 8 | TXT Messaging | Yes |

---

## DATA FLOW DIAGRAM

### Daily Processing Timeline

```
1:30am - Conflict Resolution Script #1
         └─> Resolves phone number conflicts between voice/text preferences

1:45am - Conflict Resolution Script #2
         └─> Final conflict check and patron preference sync

4:00am - Patron List Exports
         ├─> voice_patrons.txt (DeliveryOptionID = 3)
         └─> text_patrons.txt (DeliveryOptionID = 8)

8:00am - Hold Notification Export #1
         └─> holds.txt (First morning batch)

8:04am - Overdue Notification Export
         └─> overdue.txt (Types 1, 7, 8, 11, 12, 13)

8:06am - Renewal Reminder Export
         └─> renew.txt (Items due in 3 days)

9:00am - Hold Notification Export #2
         └─> holds.txt (Second morning batch)

1:00pm - Hold Notification Export #3
         └─> holds.txt (Afternoon batch)

5:00pm - Hold Notification Export #4
         └─> holds.txt (Evening batch)

6:01am (Next Day) - Invalid Phone Report Email
                    └─> Lists opt-outs and invalid numbers

4:10pm (Same Day) - Voice Failure Report Email
                    └─> Lists failed voice deliveries
```

### File Upload Process

```
1. SQL Export → Local file system (C:\shoutbomb\ftp\*)
2. WinSCP Upload → ShoutBomb FTP server
3. On Success → Move to logs (C:\shoutbomb\logs\*_submitted_*.txt)
4. Archive Copy → Local FTP server (backup)
5. WinSCP Log → Activity log (C:\shoutbomb\logs\*.log)
```

---

## TROUBLESHOOTING INDEX

### Common Issues & Solutions

| Issue | Possible Cause | Solution Document | Section |
|-------|----------------|-------------------|---------|
| Phone appears on both voice and text lists | Conflict resolution failed | SHOUTBOMB_VOICE_PATRONS.md | Conflict Resolution System |
| 3rd overdue not triggering billing | No delivery confirmation | SHOUTBOMB_OVERDUE_EXPORT.md | Third Overdue Confirmation Gap |
| Missing holds in export | Hold status changed | SHOUTBOMB_HOLDS_EXPORT.md | Known Quirks |
| Duplicate notifications | Multiple patron accounts with same phone | SHOUTBOMB_VOICE_PATRONS.md | Duplicate Phone Numbers |
| Invalid phone numbers | Patron data entry errors | SHOUTBOMB_REPORTS_INCOMING.md | Daily Invalid Phone Report |
| Voice delivery failures | Disconnected/blocked numbers | SHOUTBOMB_REPORTS_INCOMING.md | Daily Voice Failure Report |

### Validation Queries

All documentation includes SQL validation queries in their respective "VALIDATION QUERIES" sections.

### Contact Information

**System Owner:** Brian Lashbrook (blashbrook@dcplibrary.org)  
**Library:** Daviess County Public Library  
**Vendor:** ShoutBomb support  
**ILS:** Innovative Interfaces - Polaris

---

## API INTEGRATION RESOURCES

### Current State: File-Based Integration

All notification delivery is currently handled through FTP file exchanges. ShoutBomb processes the files and delivers notifications, but **there is no automated feedback loop** to confirm deliveries back to Polaris.

### Future Enhancement: Polaris API Integration

**Goal:** Implement two-way integration using Polaris API to:
1. Read notifications from NotificationQueue (already exported via files)
2. Confirm successful deliveries back to Polaris using NotificationUpdatePut endpoint
3. Solve the "3rd overdue confirmation gap" issue

**Key Resources:**
- Laravel Package: `blashbrook/papiclient` - Custom package for Polaris API integration
- API Documentation: See Polaris_Notification_Guide_PAPIClient.md
- API Specification: Polaris-API-swagger.json

**Critical Endpoint:**
```
PUT /REST/protected/v1/{LangID}/{AppID}/{OrgID}/{AccessToken}/notification/{NotificationTypeID}
```

**Use Case Example:**
```php
// After ShoutBomb confirms 3rd overdue delivery
$notificationService->markAsSent(
    notificationTypeId: 13, // 3rd Overdue
    data: [
        'NotificationStatusID' => 16, // Sent
        'PatronID' => $patronId,
        'PatronBarcode' => $patronBarcode,
        'DeliveryOptionID' => 8, // Text
        'DeliveryString' => $phoneNumber,
        'ItemRecordID' => $itemRecordId
    ]
);
```

### API Reference Tables

Complete lookup tables available in:
- **Polaris_Notification_Guide_PAPIClient.md** - Quick reference tables
- **POLARIS_PHONE_NOTICES.md** - Comprehensive reference tables with 44 languages, 22 notification types, 16 status codes

---

## DOCUMENT MAINTENANCE

### Version History

| Date | Document | Change Description |
|------|----------|-------------------|
| 2025-11-14 | POLARIS_PHONE_NOTICES.md | Complete field mapping correction (25 fields) |
| 2025-11-14 | All documentation | Added comprehensive lookup tables |
| 2025-11-13 | SHOUTBOMB_HOLDS_EXPORT.md | Fixed pipe character handling in titles |
| 2025-11-13 | SHOUTBOMB_RENEW_EXPORT.md | Documented Thursday edge case |

### Documentation Standards

When creating or updating documentation, follow the DATA_SOURCE_TEMPLATE.md structure:
1. Source Metadata
2. Field Definitions (exact order, with data types)
3. Sample Data (anonymized)
4. Cross-Reference Keys
5. Known Quirks
6. Source SQL Query (if applicable)
7. Validation Rules
8. Processing Notes
9. Change Log

### Review Schedule

- **Monthly:** Verify all documentation reflects current system state
- **After Changes:** Update documentation immediately when SQL queries or file formats change
- **Quarterly:** Review validation queries and known quirks for accuracy

---

## APPENDIX: FILE LOCATIONS

### Production System Paths

```
C:\shoutbomb\
├── shoutbomb.bat              # Master batch script
├── sql\
│   ├── voice_patrons.sql      # Voice patron list export
│   ├── text_patrons.sql       # Text patron list export
│   ├── holds.sql              # Hold notifications export
│   ├── overdue.sql            # Overdue notifications export
│   └── renew.sql              # Renewal reminder export
├── ftp\
│   ├── voice_patrons\         # Voice patron staging
│   ├── text_patrons\          # Text patron staging
│   ├── holds\                 # Hold notification staging
│   ├── overdue\               # Overdue notification staging
│   └── renew\                 # Renewal reminder staging
└── logs\
    ├── *_submitted_*.txt      # Archive of uploaded files
    └── *.log                  # WinSCP activity logs
```

### ShoutBomb FTP Structure

```
/voice_patrons/voice_patrons.txt
/text_patrons/text_patrons.txt
/holds/holds.txt
/overdue/overdue.txt
/renew/renew.txt
```

---

**END OF DOCUMENTATION INDEX**

*For detailed information on any specific component, refer to the individual documentation files listed above.*
