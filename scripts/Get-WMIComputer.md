```powershell

Function  Get-WMIComputer
{
  <#
      .SYNOPSIS
      Gets the WMI on a local or remote computer..

      .DESCRIPTION
      The Get-WMIComputer cmdlet gets WMI objects from a local or remote computer.

      .PARAMETER Computername
      Gets WMI information from a string of computers.  

      .PARAMETER Computername
      Create log in location ' '  

      .EXAMPLE 
      Get-WMIComputer -ComputerName "Server02" 
    
      This command gets the WMI on the Server02 remote computer. 

      .EXAMPLE 
      Get-WMIComputer -ComputerName "Server02" -ErrorLog 'c:\Powershell\log.txt'
    
      This command gets the WMI on the Server02 remote computer and append errors to specified location 


      .EXAMPLE 
      Get-WMIComputer | Where-object {$_.Status -eq "Success"} 
    
      This command displays only the Success WMI Computers. It uses the Get-WMIComputer cmdlet to get all of the WMI on the computer. The pipeline operator (|) passes the results to the Where-Object cmdlet, which selects
      only the Computers with a Status property that equals Success.

      .NOTES
      Windows 10 version history
      https://en.wikipedia.org/wiki/Windows_10_version_history

      .LINK
      https://github.com/DigitalNotebook/PowerShell
    

  #>


  [cmdletBinding()]
  param(

    [Parameter(Mandatory=$True,
        ValueFromPipeline=$true,
        ValueFromPipelineByPropertyName =$True)]
    [Alias('Hostname', 'cn', 'Name', 'Computer', 'PC')]  # Flexible ByPropertyName or use Hastable @{Name = 'Name'; Expression={$_.ComputerName}}
    [String[]]$Computername,
    [String]$ErrorLog =''
      )
  
  
  Begin {
  
  Write-Verbose "Error Logs location $ErrorLog"
  
  }
  Process # Process block is needed to get more than one object out of the pipeline           
  { # Process block is ignored when there is no pipleline input
     
    Foreach ($Computer in $Computername) { 

      try { # TRY

        $CimSessionOption = New-CimSessionOption -Protocol Dcom
        $CimSession = New-CimSession -ComputerName $Computer -SessionOption $CimSessionOption -ErrorAction Stop             
   
        $ComputerSystem  = Get-CimInstance -Class Win32_ComputerSystem  -Namespace root\CIMv2 -CimSession $CimSession    
        $OperatingSystem = Get-CimInstance -Class Win32_OperatingSystem -Namespace root\CIMv2 -CimSession $CimSession  
       
    
          #Splatting hash table
          $Properties           =  @{
            Status              = 'Success'
            Computername        = $Computer
            UserName            = $ComputerSystem.UserName            
            TotalPhysicalMemory = [math]::round($ComputerSystem.TotalPhysicalMemory / 1gb)
            Manufacturer        = $ComputerSystem.Manufacturer
            model               = $ComputerSystem.Model 
            SystemType          = $ComputerSystem.SystemType
               
            
            OS                  = $OperatingSystem.Version  
            ProductType         = $OperatingSystem.ProductType} # OS Type - 1 = Desktop OS, 2 = Server OS DC, 3 = Server OS Not DC
            
         

      } catch { # Catch only runs if TRY fails

          $Properties           = @{
            Status              = 'Failure'
            Computername        = $null
            UserName            = $null
            TotalPhysicalMemory = $null
            Manufacturer        = $null
            model               = $null
            SystemType          = $null
               
            
            OS                  = $null
            ProductType         = $null}
       
        if ($ErrorLog) {
          $error[0] | Out-File $ErrorLog -Append
        }
       
        Write-Verbose $error[0]                            
      
      } finally { # Finally only runs if TRY is Success
      
        $object = New-Object -TypeName psobject -Property $Properties  
        $object.psobject.typenames.insert(0, 'WMI.DigitalNotebook.objects')            
      
      
  
        if ($object.status -eq 'Connected CimSession')
        {
          Remove-CimSession -CimSession $CimSession
       
        }
        Write-Output  $object
      
      }
 
    }
  }
  
 

  
  End {}

}  

 '127.0.0.1' | Get-WMIComputer  
 
