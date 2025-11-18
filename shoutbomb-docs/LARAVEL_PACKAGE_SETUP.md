# SHOUTBOMB MONITOR LARAVEL PACKAGE SETUP

**Last Updated:** November 15, 2025
**Package:** dcplibrary/notices-shoutbomb
**Purpose:** Laravel package for monitoring and importing Shoutbomb notification data

---

## TABLE OF CONTENTS

- [Overview](#overview)
- [Installation](#installation)
- [File Download Configuration](#file-download-configuration)
- [FTP Server Setup](#ftp-server-setup)
- [SMB/CIFS Setup](#smbcifs-setup)
- [Database Configuration Override](#database-configuration-override)
- [Testing Your Setup](#testing-your-setup)
- [Troubleshooting](#troubleshooting)
- [Related Documentation](#related-documentation)

---

## OVERVIEW

This Laravel package integrates with the Shoutbomb notification system to monitor, log, and analyze notification data. It imports archived Shoutbomb export files that have been copied to a local FTP server or Windows file share after being uploaded to Shoutbomb.

### Data Flow

```
1. Polaris ILS → SQL Export → C:\shoutbomb\ftp\{type}\{type}.txt
2. WinSCP → Upload to Shoutbomb FTP server
3. On Success → Move to C:\shoutbomb\logs\{type}_submitted_{timestamp}.txt
4. Scheduled Task → Copy archived file to local FTP/SMB server
5. Laravel Package → Download from local FTP/SMB → Import & Monitor
```

### File Naming Convention

Archived files use the format: `{type}_submitted_YYYY-MM-DD_HH-MM-SS.txt` (WITH dashes)

**Examples:**
- `voice_patrons_submitted_2025-11-14_04-00-01.txt`
- `text_patrons_submitted_2025-11-14_04-00-02.txt`
- `holds_submitted_2025-11-14_08-00-01.txt`
- `overdue_submitted_2025-11-14_08-04-01.txt`
- `renew_submitted_2025-11-14_08-06-01.txt`

---

## INSTALLATION

See [INSTALLATION.md](../INSTALLATION.md) for complete installation instructions.

**Quick install:**
```bash
composer require dcplibrary/notices-shoutbomb
php artisan shoutbomb:install
```

---

## FILE DOWNLOAD CONFIGURATION

The package supports three methods for obtaining Shoutbomb export files:

### Method 1: Local Filesystem (Default)

Files are already on the local filesystem where Laravel runs.

**Configuration:**
```env
SHOUTBOMB_DOWNLOAD_METHOD=local
SHOUTBOMB_VOICE_PATRONS_PATH=/path/to/voice_patrons_submitted_20251114_040001.txt
SHOUTBOMB_TEXT_PATRONS_PATH=/path/to/text_patrons_submitted_20251114_040002.txt
SHOUTBOMB_HOLDS_PATH=/path/to/holds_submitted_20251114_080001.txt
```

**Use when:**
- Laravel runs on the same server as the Shoutbomb export process
- Files are copied to a local directory accessible by the web server

### Method 2: FTP Download

Download archived files from a dedicated FTP server.

**Configuration:**
```env
SHOUTBOMB_DOWNLOAD_METHOD=ftp
SHOUTBOMB_FTP_ENABLED=true
SHOUTBOMB_FTP_HOST=ftp.yourlibrary.org
SHOUTBOMB_FTP_PORT=21
SHOUTBOMB_FTP_USERNAME=shoutbomb_user
SHOUTBOMB_FTP_PASSWORD=your_secure_password
SHOUTBOMB_FTP_SSL=false
SHOUTBOMB_FTP_PASSIVE=true
SHOUTBOMB_FTP_TIMEOUT=90
SHOUTBOMB_FTP_BASE_PATH=/shoutbomb/logs
```

**Use when:**
- Laravel runs on a different server from Shoutbomb exports
- You have an FTP server for file distribution
- You want centralized file access

### Method 3: SMB/CIFS (Windows File Shares)

Download from Windows file shares (e.g., `\\server\shoutbomb\ftp\logs\`).

**Configuration:**
```env
SHOUTBOMB_DOWNLOAD_METHOD=smb
SHOUTBOMB_SMB_ENABLED=true
SHOUTBOMB_SMB_HOST=fileserver.yourlibrary.local
SHOUTBOMB_SMB_SHARE=shoutbomb
SHOUTBOMB_SMB_USERNAME=DOMAIN\serviceaccount
SHOUTBOMB_SMB_PASSWORD=your_secure_password
SHOUTBOMB_SMB_WORKGROUP=WORKGROUP
SHOUTBOMB_SMB_BASE_PATH=ftp/logs
```

**Use when:**
- Shoutbomb exports run on a Windows server
- You prefer Windows file sharing over FTP
- Laravel needs access to Windows network shares

---

## FTP SERVER SETUP

### Server Requirements

- FTP server (vsftpd, ProFTPD, FileZilla Server, etc.)
- PHP FTP extension enabled
- Network connectivity between Laravel and FTP server

### Recommended Setup

#### 1. Create FTP User

```bash
# Linux (vsftpd example)
sudo useradd -m -s /bin/bash shoutbomb_ftp
sudo passwd shoutbomb_ftp

# Set home directory permissions
sudo chown shoutbomb_ftp:shoutbomb_ftp /home/shoutbomb_ftp
sudo chmod 755 /home/shoutbomb_ftp
```

#### 2. Configure FTP Server

**vsftpd example** (`/etc/vsftpd.conf`):
```
anonymous_enable=NO
local_enable=YES
write_enable=NO
chroot_local_user=YES
allow_writeable_chroot=YES
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
```

#### 3. Create Directory Structure

```bash
sudo mkdir -p /home/shoutbomb_ftp/shoutbomb/logs/{voice_patrons,text_patrons,holds,overdue,renew}
sudo chown -R shoutbomb_ftp:shoutbomb_ftp /home/shoutbomb_ftp/shoutbomb
sudo chmod -R 755 /home/shoutbomb_ftp/shoutbomb
```

**Directory Structure:**
```
/home/shoutbomb_ftp/
└── shoutbomb/
    └── logs/
        ├── voice_patrons/
        │   └── voice_patrons_submitted_*.txt
        ├── text_patrons/
        │   └── text_patrons_submitted_*.txt
        ├── holds/
        │   └── holds_submitted_*.txt
        ├── overdue/
        │   └── overdue_submitted_*.txt
        └── renew/
            └── renew_submitted_*.txt
```

#### 4. Configure Automated File Copying

**Windows Task Scheduler** (on Shoutbomb export server):

```batch
@echo off
REM Upload archived files to FTP server
set FTP_HOST=ftp.yourlibrary.org
set FTP_USER=shoutbomb_ftp
set FTP_PASS=password

REM Use WinSCP or similar to upload
"C:\Program Files (x86)\WinSCP\WinSCP.com" ^
  /command ^
  "open ftp://%FTP_USER%:%FTP_PASS%@%FTP_HOST%/" ^
  "cd /shoutbomb/logs/voice_patrons" ^
  "put C:\shoutbomb\logs\voice_patrons_submitted_*.txt" ^
  "exit"
```

Schedule this task to run after the Shoutbomb export completes.

---

## SMB/CIFS SETUP

### Server Requirements

- Windows file server or Samba server
- PHP smbclient extension or `icewind/smb` package (already included)
- Network connectivity between Laravel and file server

### Recommended Setup

#### 1. Create Service Account

**Windows:**
```powershell
# Create service account
New-LocalUser -Name "ShoutbombService" -Description "Shoutbomb file access"
Set-LocalUser -Name "ShoutbombService" -Password (ConvertTo-SecureString "SecurePassword!" -AsPlainText -Force)
Set-LocalUser -Name "ShoutbombService" -PasswordNeverExpires $true
```

#### 2. Configure File Share

**Windows:**
```powershell
# Create shared folder
New-Item -Path "C:\shoutbomb" -ItemType Directory -Force
New-SmbShare -Name "shoutbomb" -Path "C:\shoutbomb" -ReadAccess "ShoutbombService"

# Set NTFS permissions
icacls "C:\shoutbomb" /grant "ShoutbombService:(OI)(CI)R"
```

#### 3. Directory Structure

```
\\fileserver\shoutbomb\
└── ftp\
    └── logs\
        ├── voice_patrons\
        ├── text_patrons\
        ├── holds\
        ├── overdue\
        └── renew\
```

#### 4. Laravel Configuration

```env
SHOUTBOMB_DOWNLOAD_METHOD=smb
SHOUTBOMB_SMB_ENABLED=true
SHOUTBOMB_SMB_HOST=fileserver.yourlibrary.local
SHOUTBOMB_SMB_SHARE=shoutbomb
SHOUTBOMB_SMB_USERNAME=DOMAIN\ShoutbombService
SHOUTBOMB_SMB_PASSWORD=SecurePassword!
SHOUTBOMB_SMB_WORKGROUP=YOURLIBRARY
SHOUTBOMB_SMB_BASE_PATH=ftp/logs
```

---

## DATABASE CONFIGURATION OVERRIDE

All configuration values can be stored in the database to override `.env` settings. This allows runtime configuration changes without editing files.

### Priority Order

1. **Database** (`shoutbomb_settings` table) - highest priority
2. **Environment** (`.env` file) - medium priority
3. **Config** (`config/shoutbomb.php`) - lowest priority (defaults)

### Storing Settings in Database

**Via API:**
```bash
POST /api/shoutbomb/config
{
  "key": "ftp.host",
  "value": "ftp.example.com",
  "type": "string",
  "is_active": true
}
```

**Via Tinker:**
```php
php artisan tinker

use Dcplibrary\NoticesShoutbomb\Models\ShoutbombSetting;

ShoutbombSetting::create([
    'key' => 'ftp.host',
    'value' => 'ftp.example.com',
    'type' => 'string',
    'is_active' => true,
]);
```

**Via Migration/Seeder:**
```php
DB::table('shoutbomb_settings')->insert([
    'key' => 'ftp.host',
    'value' => 'ftp.example.com',
    'type' => 'string',
    'is_active' => true,
    'created_at' => now(),
    'updated_at' => now(),
]);
```

### Supported Configuration Keys

All config keys from `config/shoutbomb.php` can be stored in database using dot notation:

- `ftp.host`
- `ftp.port`
- `ftp.username`
- `ftp.password`
- `ftp.base_path`
- `smb.host`
- `smb.share`
- `download_method`
- etc.

### Caching

Configuration values are cached for performance. Clear cache after updating database settings:

```bash
php artisan cache:clear
```

Or via API:
```bash
POST /api/shoutbomb/config/cache/clear
```

---

## TESTING YOUR SETUP

### Test FTP/SMB Connection

```bash
php artisan shoutbomb:test-download-connection
```

**Successful output:**
```
Testing download connection using method: ftp

✓ Download method is configured and enabled
Testing connection...

✓ Connection test successful!

Available file paths:
+---------------+--------------------------------+
| File Type     | Path                          |
+---------------+--------------------------------+
| voice_patrons | /shoutbomb/logs/voice_patrons/voice_patrons.txt |
| text_patrons  | /shoutbomb/logs/text_patrons/text_patrons.txt   |
| holds         | /shoutbomb/logs/holds/holds.txt                 |
| overdue       | /shoutbomb/logs/overdue/overdue.txt             |
| renew         | /shoutbomb/logs/renew/renew.txt                 |
+---------------+--------------------------------+
```

### Test File Import

```bash
# Test importing a single file
php artisan shoutbomb:import-voice-patrons /path/to/voice_patrons_submitted_20251114_040001.txt

# Or let the package download and import automatically
php artisan shoutbomb:import-voice-patrons
```

### Verify Database Import

```bash
php artisan tinker

use Dcplibrary\NoticesShoutbomb\Models\VoicePatron;
VoicePatron::count();  // Should show imported count
VoicePatron::latest()->first();  // Show latest record
```

---

## TROUBLESHOOTING

### FTP Connection Fails

**Problem:** "FTP connection failed: Connection refused"

**Solutions:**
1. Check firewall rules allow FTP (port 21)
2. Verify FTP server is running: `sudo systemctl status vsftpd`
3. Test with FTP client: `ftp ftp.yourlibrary.org`
4. Check passive mode ports are open (40000-40100)

**Problem:** "FTP connection failed: Login authentication failed"

**Solutions:**
1. Verify username/password in `.env`
2. Check FTP user exists: `cat /etc/passwd | grep shoutbomb_ftp`
3. Test login manually: `ftp ftp.yourlibrary.org` then enter credentials

### SMB Connection Fails

**Problem:** "SMB connection failed: Failed to connect to server"

**Solutions:**
1. Verify SMB host is reachable: `ping fileserver.yourlibrary.local`
2. Check SMB ports open (139, 445): `telnet fileserver.yourlibrary.local 445`
3. Verify share exists: `smbclient -L //fileserver.yourlibrary.local -U username`
4. Check credentials include domain: `DOMAIN\username`

**Problem:** "SMB connection failed: Access denied"

**Solutions:**
1. Verify user has read permissions on share
2. Check NTFS permissions allow user access
3. Confirm share permissions grant read access
4. Test with `smbclient`: `smbclient //server/share -U username`

### File Not Found

**Problem:** "Local file not found" or "File does not exist on FTP"

**Solutions:**
1. Verify files are being copied to FTP/SMB server
2. Check base_path configuration matches server directory structure
3. Confirm file naming matches pattern: `{type}_submitted_YYYYMMDD_HHMMSS.txt`
4. Check file permissions (must be readable)

### Database Override Not Working

**Problem:** Changes to database settings don't take effect

**Solutions:**
1. Clear cache: `php artisan cache:clear`
2. Verify setting is active: `is_active = 1`
3. Check key name matches config dot notation
4. Restart queue workers if using queues

---

## RELATED DOCUMENTATION

### Shoutbomb Integration Documentation

- [SHOUTBOMB_DOCUMENTATION_INDEX.md](shoutbomb/SHOUTBOMB_DOCUMENTATION_INDEX.md) - Central index for all Shoutbomb integration docs
- [SHOUTBOMB_VOICE_PATRONS.md](shoutbomb/SHOUTBOMB_VOICE_PATRONS.md) - Voice patron list format
- [SHOUTBOMB_TEXT_PATRONS.md](shoutbomb/SHOUTBOMB_TEXT_PATRONS.md) - Text patron list format
- [SHOUTBOMB_HOLDS_EXPORT.md](shoutbomb/SHOUTBOMB_HOLDS_EXPORT.md) - Holds export format
- [SHOUTBOMB_OVERDUE_EXPORT.md](shoutbomb/SHOUTBOMB_OVERDUE_EXPORT.md) - Overdue export format
- [SHOUTBOMB_RENEW_EXPORT.md](shoutbomb/SHOUTBOMB_RENEW_EXPORT.md) - Renew export format
- [SHOUTBOMB_REPORTS_INCOMING.md](shoutbomb/SHOUTBOMB_REPORTS_INCOMING.md) - Incoming email report formats

### Package Documentation

- [README.md](../README.md) - Package overview and quick start
- [INSTALLATION.md](../INSTALLATION.md) - Detailed installation guide
- [CONTRIBUTING.md](../CONTRIBUTING.md) - Contributing guidelines and documentation rules

### API Documentation

- Access interactive API docs at: `https://your-app.com/api/shoutbomb/documentation`
- OpenAPI spec: `https://your-app.com/api/shoutbomb/openapi.json`

---

**Last Updated:** November 15, 2025
**Package Maintainer:** Brian Lashbrook (blashbrook@dcplibrary.org)
**Organization:** DC Public Library
