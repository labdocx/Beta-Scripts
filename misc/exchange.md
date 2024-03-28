```powershell
$UserCredential = Get-Credential
$Exchange = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri http://server/powershell -Authentication Kerberos -Credential $UserCredential
Import-PSSession $Exchange -allowclobber  
#remove-pssession $Exchange
