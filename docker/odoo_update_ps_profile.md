# Odoo PowerShell Helper — Setup Guide

A PowerShell function to install and update Odoo addons running in Docker with a single command.

---

## Prerequisites

- Windows with PowerShell 5.1 or later
- Docker Desktop installed and running
- `docker-compose.yml` configured for your Odoo project

---

## Installation

### Step 1: Check if a PowerShell profile exists

```powershell
Test-Path $PROFILE
```

- **True** → Profile exists, skip to Step 3
- **False** → Create it in Step 2

---

### Step 2: Create the profile (if it doesn't exist)

```powershell
New-Item -Path $PROFILE -ItemType File -Force
```

This creates the profile at:
```
C:\Users\<YourName>\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
```

---

### Step 3: Open the profile in an editor

```powershell
# VS Code
code $PROFILE

# Notepad
notepad $PROFILE
```

---

### Step 4: Paste the function

Copy and paste the following function into your profile file and save it:

```powershell
function Odoo {
    param(
        [Parameter(Mandatory=$false)]
        [Alias("i")]
        [string]$Install,

        [Parameter(Mandatory=$false)]
        [Alias("u")]
        [string]$Update,

        [Parameter(Mandatory=$true)]
        [Alias("d")]
        [string]$Database,

        [Parameter(Mandatory=$false)]
        [Alias("dcp")]
        [string]$ComposePath = "."
    )

    if (-not $Install -and -not $Update) {
        Write-Host "ERROR: You must specify either -i <module> to install or -u <module> to update." -ForegroundColor Red
        return
    }

    if ($Install -and $Update) {
        Write-Host "ERROR: Cannot use -i and -u at the same time. Run separately." -ForegroundColor Red
        return
    }

    if ($Install) {
        $flag = "-i"
        $Module = $Install
        $actionLabel = "INSTALL"
    } else {
        $flag = "-u"
        $Module = $Update
        $actionLabel = "UPDATE"
    }

    $originalLocation = Get-Location
    Set-Location $ComposePath

    Write-Host ""
    Write-Host "========================================" -ForegroundColor DarkCyan
    Write-Host "         ODOO ADDONS $actionLabel           " -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor DarkCyan
    Write-Host "  Action    : $actionLabel"               -ForegroundColor Magenta
    Write-Host "  Module(s) : $Module"                   -ForegroundColor Yellow
    Write-Host "  Database  : $Database"                 -ForegroundColor Yellow
    Write-Host "  Path      : $(Resolve-Path $ComposePath)" -ForegroundColor Yellow
    Write-Host "========================================" -ForegroundColor DarkCyan
    Write-Host ""

    Write-Host ">> Step 1/3 : Running $actionLabel on module(s)..." -ForegroundColor Cyan
    docker compose run --rm odoo odoo $flag $Module -d $Database --stop-after-init

    if ($LASTEXITCODE -ne 0) {
        Write-Host ""
        Write-Host "FAILED: $actionLabel failed! Aborting restart." -ForegroundColor Red
        Set-Location $originalLocation
        return
    }

    Write-Host "OK: Module $actionLabel complete." -ForegroundColor Green
    Write-Host ""

    Write-Host ">> Step 2/3 : Stopping Odoo container..." -ForegroundColor Cyan
    docker compose stop odoo

    if ($LASTEXITCODE -ne 0) {
        Write-Host "FAILED: Failed to stop Odoo container." -ForegroundColor Red
        Set-Location $originalLocation
        return
    }

    Write-Host "OK: Odoo stopped." -ForegroundColor Green
    Write-Host ""

    Write-Host ">> Step 3/3 : Starting Odoo container..." -ForegroundColor Cyan
    docker compose up -d odoo

    if ($LASTEXITCODE -ne 0) {
        Write-Host "FAILED: Failed to start Odoo container." -ForegroundColor Red
        Set-Location $originalLocation
        return
    }

    Write-Host "OK: Odoo is back up!" -ForegroundColor Green
    Write-Host ""
    Write-Host "========================================" -ForegroundColor DarkCyan
    Write-Host "  $actionLabel Successful!" -ForegroundColor Green
    Write-Host "  Module(s) : $Module" -ForegroundColor Green
    Write-Host "  Database  : $Database" -ForegroundColor Green
    Write-Host "========================================" -ForegroundColor DarkCyan
    Write-Host ""

    Set-Location $originalLocation
}
```

---

### Step 5: Reload the profile

```powershell
. $PROFILE
```

> New PowerShell windows will load the function automatically from this point on.

---

### Step 6: Verify the function is loaded

```powershell
Get-Command Odoo
```

Expected output:
```
CommandType     Name     Version    Source
-----------     ----     -------    ------
Function        Odoo
```

---

## Usage

### Parameters

| Parameter | Alias | Mandatory | Description                              |
|-----------|-------|-----------|------------------------------------------|
| `-Install`  | `-i`  | No*       | Module(s) to install                     |
| `-Update`   | `-u`  | No*       | Module(s) to update                      |
| `-Database` | `-d`  | Yes       | Odoo database name                       |
| `-ComposePath` | `-dcp` | No   | Path to `docker-compose.yml` (default: current directory) |

> *Either `-i` or `-u` must be provided, but not both.

---

### Examples

```powershell
# Install a single module
Odoo -i sd_library_mgmt -d training

# Update a single module
Odoo -u sd_library_mgmt -d training

# Install multiple modules
Odoo -i module1,module2,module3 -d my_db

# Update multiple modules
Odoo -u module1,module2,module3 -d my_db

# With a custom path to docker-compose.yml
Odoo -i sd_library_mgmt -d training -dcp "C:\Projects\odoo"
Odoo -u sd_library_mgmt -d training -dcp "C:\Projects\odoo"
```

---

## Error Cases

| Scenario | Behaviour |
|----------|-----------|
| Neither `-i` nor `-u` provided | Prints error and exits |
| Both `-i` and `-u` provided | Prints error and exits |
| `-d` not provided | PowerShell prompts for it |
| Docker update/install step fails | Prints error and aborts restart |
| Docker stop step fails | Prints error and exits |
| Docker start step fails | Prints error and exits |

---

## Troubleshooting

**"Running scripts is disabled on this system"**

Run this once as Administrator:
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

**Function not found after opening a new terminal**

Verify the profile path and content:
```powershell
$PROFILE
cat $PROFILE
```

**Wrong docker-compose directory**

Always pass `-dcp` explicitly if you are running the command from a different directory:
```powershell
Odoo -u my_module -d my_db -dcp "C:\Projects\odoo"
```

---

## How It Works

The function wraps three Docker commands into one:

```
1. docker compose run --rm odoo odoo -i/-u <module> -d <db> --stop-after-init
2. docker compose stop odoo
3. docker compose up -d odoo
```

Each step is validated — if any step fails, the process halts and reports the error.