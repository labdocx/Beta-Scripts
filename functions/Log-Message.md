```powershell

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
