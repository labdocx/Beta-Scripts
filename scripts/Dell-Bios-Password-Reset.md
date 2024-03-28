```powershell

<#
This script is designed to reset the Dell BIOS password.                                                                                                            
                                                                                                                                                                   
You need to add all your previously used passwords to the Array provided in the script.                                                                            
                                                                                                                                                                   
The script will then loop through each old password and set the new password.                                                                                     
                                                                                                                                                                   
To run the script, first download the DellBIOSProvider PowerShell module and save it to a folder using the command "save-module".                                   
The script should then be run from that location. If the PowerShell module is not found, the script will copy the module to the correct location and then load it.  
#>

# Define CSV File path
$csvFile  = "C:\BiosPasswordResetLog.csv"

$DisplayVersion = "1.0.10"
$date = (Get-Date).ToString("dd-MM-yyyy HH:mm:ss")


# Define passwords
$OldAdminPasswords = @("password1", "password2", "etc")
$NewAdminPassword = "password1"
$success = $false


#log message to csv
function Log-Message {
  param (
    [string]$Message
  )

  $Info       = [PSCustomObject]@{
    Timestamp = (Get-Date).ToString("dd-MM-yyyy HH:mm:ss")
    Hostname  =  $env:computername
    Message   = $Message
  }
  
  try{
 
    $Info | Export-csv -Path $csvFile -NoTypeInformation -Append  -ErrorAction Stop
  
  }

  catch {
  
    Write-Host "Error appending message to csv file: $_"
    
  
  }
   }
  
  

# Powershell Module
if (-not (Test-Path -Path "$env:ProgramFiles\WindowsPowerShell\Modules\DellBIOSProvider")) {
    Copy-Item -Path "$PSScriptRoot\DellBIOSProvider\" -Destination "$env:ProgramFiles\WindowsPowerShell\Modules\DellBIOSProvider" -Recurse -Force
    Write-Host "Copying Powershell Module to $env:ProgramFiles\WindowsPowerShell\Modules\DellBIOSProvider"
    
}

 
try {

      if((get-module -Name DellBIOSProvider | Select-Object name -ExpandProperty name) -eq "DellBIOSProvider")
  
      {
  
      Write-Host "The Dell module DellBIOSProvider is loaded."
  
      }
  
       else {

        Import-Module "DellBIOSProvider" -Force -Verbose -ErrorAction Stop  

      }
    }  

    
catch {
        
  Write-Host "Error importing module: $_"
  $message = "Error importing module: $_"

  Write-Host "Create regkey HKEY_LOCAL_MACHINE\SOFTWARE\labdocx\DellBIOSProvider\BiosPasswordSet and set value Fail"
  Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName BiosPasswordSet -registryValue "Fail" -registryType String  
  Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName InstallDate -registryValue "$date" -registryType String
  Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName DisplayName -registryValue "Bios Password Reset" -registryType String
  Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName DisplayVersion  -registryValue $DisplayVersion -registryType String

  Log-Message $Message
  exit 1
     
}
    
  
# Set registry key
function Set-RegistryKey {

  param(
    [string]$RegistryKey,
    [string]$SubRegistryKey,
    [string]$registryPath,
    [string]$registryName,
    [string]$registryValue,
    [string]$registryType
  )
  # Create the registry key if it does not exist
  if (-not (Test-Path $RegistryKey)) {
    New-Item -Path $RegistryKey -Force | Out-Null
    Write-Host "Create registry key path $RegistryKey"
  }
  
  # Create the registry key if it does not exist
  if (-not (Test-Path $SubRegistryKey)) {
    New-Item -Path $SubRegistryKey -Force | Out-Null
    Write-Host "Create Sub registry key path $SubRegistryKey"
  }
  
  
  
  # Set the registry value
  New-ItemProperty -Path $registryPath -Name $registryName -Value $registryValue -PropertyType $registryType -Force | Out-Null  
}

   
# Dell Bios Password
$currentPassword = Get-Item -Path DellSmbios:\Security\IsAdminPasswordSet | Select-Object -Property CurrentValue -ExpandProperty CurrentValue

 
        
    if ($currentPassword -eq "false") { # if no password
      Write-Host "Blank password set to $NewAdminPassword"
      Set-Item DellSmbios:\Security\AdminPassword -value $NewAdminPassword  
      $message = "New Password set"
      Log-Message $Message
      
      Write-Host "Create regkey HKEY_LOCAL_MACHINE\SOFTWARE\labdocx\DellBIOSProvider\BiosPasswordSet and set value Success"
      Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName BiosPasswordSet -registryValue "Success" -registryType String  
      Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName InstallDate -registryValue "$date" -registryType String
      Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName DisplayName -registryValue "Bios Password Reset" -registryType String
      Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName DisplayVersion  -registryValue $DisplayVersion -registryType String

      break  # Exit the loop if successful
      
    } else {

        foreach ($OldAdminPassword in $OldAdminPasswords) {
        try{
          
          Set-Item -Path DellSmbios:\Security\AdminPassword -Value $NewAdminPassword -Password $OldAdminPassword -ErrorAction Stop
          Write-Host "New Password set to $NewAdminPassword using old password $OldAdminPassword"
          $success = $true
          $message = "New Password set"
          Log-Message $Message
          
          Write-Host "Create regkey HKEY_LOCAL_MACHINE\SOFTWARE\labdocx\DellBIOSProvider\BiosPasswordSet and set value Success"
          Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName BiosPasswordSet -registryValue "Success" -registryType String  
          Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName InstallDate -registryValue "$date" -registryType String
          Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName DisplayName -registryValue "Bios Password Reset" -registryType String
          Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName DisplayVersion  -registryValue $DisplayVersion -registryType String

          break  # Exit the loop if successful
                      
         } catch {
        
          Write-Host "Failed to change password with password $OldAdminPassword"
          # Failed to change password 

            
        }
      }
    }
    

# All password change attempts failed    
if (-not $success) {
  Write-Host "All password change attempts failed."
  $message = "All password change attempts failed."
  
  Write-Host "Create regkey HKEY_LOCAL_MACHINE\SOFTWARE\labdocx\DellBIOSProvider\BiosPasswordSet and set value Fail"
          Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName BiosPasswordSet -registryValue "Fail" -registryType String  
          Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName InstallDate -registryValue "$date" -registryType String
          Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName DisplayName -registryValue "Bios Password Reset" -registryType String
          Set-RegistryKey -RegistryKey HKLM:\SOFTWARE\labdocx -SubRegistryKey HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryPath HKLM:\SOFTWARE\labdocx\DellBIOSProvider -registryName DisplayVersion  -registryValue $DisplayVersion -registryType String
  
  Log-Message $Message
  exit 1
}
