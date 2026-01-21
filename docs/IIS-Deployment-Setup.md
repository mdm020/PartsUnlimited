# IIS Deployment Setup Guide

## Prerequisites for Windows IIS Server

### 1. Install IIS with Required Features
Run this PowerShell script as Administrator:

```powershell
# Install IIS and required features
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
Install-WindowsFeature -Name Web-Asp-Net45
Install-WindowsFeature -Name Web-ISAPI-Ext
Install-WindowsFeature -Name Web-ISAPI-Filter
Install-WindowsFeature -Name Web-Net-Ext45
Install-WindowsFeature -Name Web-AppInit
Install-WindowsFeature -Name Web-WebSockets

# Install .NET Framework 4.8 Hosting Bundle
# Download from: https://dotnet.microsoft.com/download/dotnet-framework/net48
```

### 2. Install Web Deploy (MS Deploy)
- Download Web Deploy 3.6: https://www.iis.net/downloads/microsoft/web-deploy
- Or run: `choco install webdeploy -y` (if using Chocolatey)

### 3. Configure IIS Application Pool

```powershell
# Create Application Pool for PartsUnlimited
Import-Module WebAdministration

$appPoolName = "PartsUnlimitedAppPool"

# Create App Pool
New-WebAppPool -Name $appPoolName

# Configure App Pool
Set-ItemProperty IIS:\AppPools\$appPoolName -Name "managedRuntimeVersion" -Value "v4.0"
Set-ItemProperty IIS:\AppPools\$appPoolName -Name "enable32BitAppOnWin64" -Value $false
Set-ItemProperty IIS:\AppPools\$appPoolName -Name "processModel.identityType" -Value "ApplicationPoolIdentity"
Set-ItemProperty IIS:\AppPools\$appPoolName -Name "recycling.periodicRestart.time" -Value "00:00:00"
```

### 4. Create IIS Website

```powershell
# Create website directory
$sitePath = "C:\inetpub\wwwroot\PartsUnlimited"
New-Item -ItemType Directory -Path $sitePath -Force

# Create IIS Website
$siteName = "PartsUnlimited"
$port = 80  # Change as needed
$hostName = ""  # Optional: Set hostname binding

New-Website -Name $siteName `
            -Port $port `
            -PhysicalPath $sitePath `
            -ApplicationPool $appPoolName `
            -Force

# Grant permissions to App Pool identity
$acl = Get-Acl $sitePath
$permission = "IIS AppPool\$appPoolName", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow"
$accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule $permission
$acl.SetAccessRule($accessRule)
Set-Acl $sitePath $acl
```

### 5. Configure SQL Server Database
If using SQL Server, update connection string in Web.config:

```xml
<connectionStrings>
  <add name="DefaultConnectionString" 
       connectionString="Server=YOUR_SERVER;Database=PartsUnlimited;Integrated Security=true;" 
       providerName="System.Data.SqlClient" />
</connectionStrings>
```

Or create the database:
```powershell
# Install SQL Server Express if needed
# Download from: https://www.microsoft.com/en-us/sql-server/sql-server-downloads
```

### 6. Setup GitHub Self-Hosted Runner

On your Windows IIS server:

1. Go to your GitHub repository: Settings → Actions → Runners → New self-hosted runner
2. Follow the instructions to download and configure the runner:

```powershell
# Download runner (example for Windows x64)
mkdir actions-runner; cd actions-runner
Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-win-x64-2.311.0.zip -OutFile actions-runner-win-x64-2.311.0.zip
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::ExtractToDirectory("$PWD/actions-runner-win-x64-2.311.0.zip", "$PWD")

# Configure runner
./config.cmd --url https://github.com/YOUR_USERNAME/PartsUnlimited --token YOUR_TOKEN

# Install and start as a Windows service
./svc.sh install
./svc.sh start
```

### 7. Configure GitHub Secrets

In your GitHub repository, go to Settings → Secrets and variables → Actions, and add:

| Secret Name | Description | Example Value |
|------------|-------------|---------------|
| `IIS_APP_POOL_NAME` | IIS Application Pool name | `PartsUnlimitedAppPool` |
| `IIS_SITE_PATH` | Physical path to IIS website | `C:\inetpub\wwwroot\PartsUnlimited` |
| `IIS_SITE_URL` | Website URL for verification | `http://localhost` or `http://yourserver.com` |

### 8. Configure Firewall (if needed)

```powershell
# Allow HTTP traffic
New-NetFirewallRule -DisplayName "Allow HTTP" -Direction Inbound -LocalPort 80 -Protocol TCP -Action Allow

# Allow HTTPS traffic
New-NetFirewallRule -DisplayName "Allow HTTPS" -Direction Inbound -LocalPort 443 -Protocol TCP -Action Allow
```

### 9. Install Required Dependencies on Server

```powershell
# Install .NET Framework 4.8 (if not already installed)
# Download: https://dotnet.microsoft.com/download/dotnet-framework/net48

# Install Visual C++ Redistributables
# Download: https://support.microsoft.com/en-us/help/2977003/the-latest-supported-visual-c-downloads

# Install URL Rewrite Module (optional but recommended)
# Download: https://www.iis.net/downloads/microsoft/url-rewrite
```

### 10. Test IIS Configuration

```powershell
# Verify IIS is running
Get-Service W3SVC | Select-Object Status, Name

# Check website status
Import-Module WebAdministration
Get-Website | Select-Object Name, State, PhysicalPath, Bindings

# Check App Pool status
Get-WebAppPoolState -Name "PartsUnlimitedAppPool"

# Browse to website
Start-Process "http://localhost"
```

## Deployment Workflow

Once setup is complete:

1. Push code to `master` or `main` branch
2. CI workflow builds and tests the application
3. CD workflow automatically deploys to IIS
4. Application pool is stopped, files copied, then restarted
5. Deployment verification runs to confirm site is up

## Troubleshooting

### Check IIS Logs
```powershell
# View IIS logs
Get-Content "C:\inetpub\logs\LogFiles\W3SVC1\*.log" -Tail 50
```

### Check Application Event Log
```powershell
Get-EventLog -LogName Application -Source "ASP.NET*" -Newest 20
```

### Reset IIS
```powershell
iisreset /restart
```
