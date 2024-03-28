"powershell"

#The Script will install global protect when not connected to vpn , 
#add portal address, 
#register PanPlapProvider,
#remove old certificate and
#install new certificate


#Global Protect install version
$GlobalProtectRequiredVersion = "1.1.1"

# Define the MSI package and name
$MSIPackage = "package.msi"

# GlobalProtect Portal Address
$PortalAddress = 'vpn.test.com.au'

# Define Script Log CSV File path
$csvFile = "c:\" + "{0}" -f ($MSIPackage -replace '\.msi$', '-ScriptLog.csv')

# Create the install log file name
$InstallLog = "{0}" -f ($MSIPackage -replace '\.msi$', '.log')

# Define MSI arguments
$MSIArguments = "/i  `"$PSScriptRoot\$MSIPackage`" /qn /norestart /L*v  `"c:\$InstallLog`" "

$GlobalProtectInstallCheck = $false
$PortalAddressCheck = $false
$GlobalProtectPanGPSCheck = $false
$GlobalProtectCertificateCheck = $false


#log message to csv	
function Log-Message
{
  param (
    [string]$Message
  )
	
  $Info = [PSCustomObject]@{
    Timestamp = (Get-Date).ToString("dd-MM-yyyy HH:mm:ss")
    Hostname  = $env:computername
    Message   = $Message
  }
	
  try
  {
    $Info | Export-csv -Path $csvFile -NoTypeInformation -Append -ErrorAction Stop
  }
  catch
  {
    Write-Host "Error appending message to csv file: $_"
  }
}



#Get Global Protect adapter properties
$Adapter = Get-NetAdapter | Where-Object { ($_.InterfaceDescription -like "*PANGP*") -and ($_.DriverProvider -like "*PALO*") } `
| Select-Object Name, InterfaceDescription, DriverProvider, Status


#Get Global Protect Version
$GlobalProtectInstalledVersion = Get-ItemProperty -Path 'HKLM:\SOFTWARE\Palo Alto Networks\GlobalProtect' | Select-Object version -ExpandProperty version -ErrorAction SilentlyContinue



# Global Protect is either disabled or not installed, and the installed version is older than the required version.
if ((-not $adapter -or $adapter.Status -eq "Disabled") -and (-not $GlobalProtectInstalledVersion -or $GlobalProtectInstalledVersion -lt $GlobalProtectRequiredVersion))
{
  Write-Output "Global Protect is not connected or the required version. Proceed with install"
  $Message = "Global Protect is not connected or the required version. Proceed with install"
  $GlobalProtectinstallCheck = $true
  Log-Message -Message $Message
	
  try
  {
    #Install Software
    Start-Process "msiexec.exe" -ArgumentList $MSIArguments -Wait -NoNewWindow -verbose
    Write-Output "Installing: $($MSIArguments)"
    $Message = "Installing: $($MSIArguments)"
    $GlobalProtectinstallCheck = $true
    Log-Message -Message $Message
		
  }
  catch
  {
    Write-Output "Error occurred during $($MSIPackage) Install: $_"
    $Message = "Error occurred during $($MSIPackage) Install: $_"
    Log-Message -Message $Message
    exit 1
  }
	
}
else
{
  Write-Host "Global Protect is not in the correct state, or version $($GlobalProtectInstalledVersion) is installed, which is older or equal to the required $($GlobalProtectRequiredVersion) version. Terminating install"
  $Message = "Global Protect is not in the correct state, or version $($GlobalProtectInstalledVersion) is installed, which is older or equal to the required $($GlobalProtectRequiredVersion) version. Terminating install"
  $GlobalProtectinstallCheck = $True
  Log-Message -Message $Message
  exit 1
	
}

# Get Portal Address property
$PortalValue = Get-ItemProperty -Path 'HKLM:\SOFTWARE\Palo Alto Networks\GlobalProtect\PanSetup' | Select-Object portal -ExpandProperty portal -ErrorAction SilentlyContinue

# Add portal Address
if (-not $PortalValue -or $PortalValue -ne "vpn.test.com.au")
{
  New-ItemProperty -Path 'HKLM:\SOFTWARE\Palo Alto Networks\GlobalProtect\PanSetup' -Name 'Portal' -Value $PortalAddress -PropertyType String -Force | Out-Null
  Write-Output "Add Palo Address to Global Protect: HKLM:\SOFTWARE\Palo Alto Networks\GlobalProtect\PanSetup Name  $($PortalAddress)"
  $Message = "Add Palo Address to Global Protect: HKLM:\SOFTWARE\Palo Alto Networks\GlobalProtect\PanSetup Name  $($PortalAddress)"
  $PortalAddressCheck = $true
  Log-Message -Message $Message
}
else
{
  Write-Host "Portal Address $($PortalValue) is set"
  $Message = "Portal Address $($PortalValue) is set"
  $PortalAddressCheck = $true
  Log-Message -Message $Message
}


# Get PanPlapProvider property
$PanGPS = Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Authentication\PLAP Providers\{20A29589-E76A-488B-A520-63582302A285}' | Select-Object '(Default)' -ExpandProperty '(Default)' -ErrorAction SilentlyContinue

# Register PanPlapProvider
if (-not $PanGPS -or $PanGPS -ne "PanPlapProvider")
{
  Start-Process -FilePath "$env:ProgramFiles\Palo Alto Networks\GlobalProtect\PanGPS.exe" -ArgumentList "-registerplap" -Wait
  Write-Output "Register PanPlapProvider"
  $Message = "Register PanPlapProvider"
  $GlobalprotectPanGPSCheck = $true
  Log-Message -Message $Message
}
else
{
  Write-Host "$($PanGPS) is Set "
  $Message = "$($PanGPS) is Set "
  $GlobalprotectPanGPSCheck = $true
  Log-Message -Message $Message
}


# Certificate Thumbprint
$OldThumbprint = "2D2A1A5CAFCDA1"
$NewThumbprint = "B580CDDE7F095C"  
 

# Get Certificate property
$Certificate = Get-ChildItem -Path 'Cert:\LocalMachine\My' -ErrorAction SilentlyContinue | Select-Object Subject, Issuer, Thumbprint, FriendlyName, NotBefore, NotAfter  

# Check
$NewThumbprintFound = $false

if (-not $Certificate) {
   
    Import-PfxCertificate -filepath "$PSScriptRoot\certificate.pfx" -CertStoreLocation 'Cert:\LocalMachine\My' -Password (ConvertTo-SecureString -String 'password' -AsPlainText -Force)
    $GlobalProtectCertificateCheck = $true
    $NewThumbprintFound = $true
    $Message = "No Certificates found, Installing certificate"
    write-host "No Certificates found, Installing certificate"
    Log-Message -Message $Message

} else {

    # Remove expired certs and check for new certificate
    foreach ($cert in $Certificate) {
 
        # Remove old certificate  
        if ($OldThumbprint -eq $cert.Thumbprint) {

            Remove-Item -Path "Cert:\LocalMachine\My\$($cert.Thumbprint)"
            $Message = "Removing old certificate $($cert.Thumbprint)"
            write-host "Removing old certificate $($cert.Thumbprint)"
            Log-Message -Message $Message
        } 


        # Check for NEW certificate  
        if ($NewThumbprint -eq $cert.Thumbprint) {
            $NewThumbprintFound = $true
            $GlobalProtectCertificateCheck = $true
            $Message = "New certificate is installed."
            Write-Host "New certificate is installed."
            Log-Message -Message $Message
        }
    }

    # If the new thumbprint does not exist
    if (-not $NewThumbprintFound) {
        Write-Host "New thumbprint does not exist. Importing new certificate."
          Import-PfxCertificate -filepath "$PSScriptRoot\certificate.pfx" -CertStoreLocation 'Cert:\LocalMachine\My' -Password (ConvertTo-SecureString -String 'password' -AsPlainText -Force)
          $GlobalProtectCertificateCheck = $true
          $Message = "New Certificate not found. Installing new certificate"
          Write-Host "New Certificate not found. Installing new certificate"
          Log-Message -Message $Message

    }
}


# Final checks
if ($GlobalProtectinstallCheck -and $PortalAddressCheck -and $GlobalprotectPanGPSCheck -and $GlobalProtectCertificateCheck -eq $true)
{
  Write-Host "Final check Pass - Global Protect installed, Portal Address and PanPlapProvider Set"
  $Message = "Final check Pass - Global Protect installed, Portal Address and PanPlapProvider Set"
  Log-Message -Message $Message
}
else
{
  Write-Host "Final check Fail - Further action is required."
  $Message = "Final check Fail - Further action is required."
  Log-Message -Message $Message
  exit 1
}
