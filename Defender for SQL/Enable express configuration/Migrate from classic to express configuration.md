# Migration from classic to express configuration

## Microsoft script using user credentials for authentication
[Sample script - MigratingToExpressConfiguration.ps1](https://learn.microsoft.com/en-us/azure/defender-for-cloud/powershell-sample-vulnerability-assessment-azure-sql#sample-script---migratingtoexpressconfigurationps1)


## Script that uses application registered in Entra ID for authentication

### Permission required for application
* `Reader` role on subscription level
* `Contributor` role on the resource group level where the sql server locates
* `Storage Blob Data Reader` role on the storage account which used to save scan results and baselines

![image](https://github.com/user-attachments/assets/986a0603-2420-4dad-98f7-5fbb6ebaf6a2)


### Scirpt(including migration)
```powershell
# Requires -Modules @{ ModuleName="Az.Sql"; ModuleVersion="3.11.0" }
# Requires -Modules @{ ModuleName="Az.Accounts"; ModuleVersion="2.9.1" }
# Requires -Version 5.1

######SQL Vulnerability Assessment PowerShell Commands ######
#############################################################


#Requires -Modules @{ ModuleName="Az.Sql"; ModuleVersion="3.11.0" }
#Requires -Modules @{ ModuleName="Az.Storage"; ModuleVersion="4.8.0" }
#Requires -Modules @{ ModuleName="Az.Accounts"; ModuleVersion="2.9.1" }
#Requires -Version 5.1

<#
.SYNOPSIS
    This script configured an Azure SQL Server to the express configuration Vulnerability Assessment feature, scans all databases that belong to the selected server and set baselines (if defined in classic configuration) from the storage that is configures in the policy.

.DESCRIPTION
This script migrates Azure SQL Server to express configuration Vulnerability Assessment feature by executing the following steps:
- It deletes the current Vulnerability Assessment settings (if exists).
  This step will reset all the Vulnerability Assessment scans and baselines for all databases.
- It copies the current baselines (if exist) from the customer's storage.

In order to revert this script follow the instructions as mentioned here: https://learn.microsoft.com/azure/defender-for-cloud/sql-azure-vulnerability-assessment-manage?tabs=express#revert-back-to-the-classic-configuration.

.PARAMETER ServerSubscriptionId
    The Subscription id that the server belongs to

.PARAMETER ServerResourceGroupName
    The Resource Group that the server belongs to

.PARAMETER ServerName
    The SQL server name that we want to apply the new SQL Vulnerability Assessment policy to.

.PARAMETER Force
    Will remove the old Vulnerability Assessment setting without asking for confirmation.

.EXAMPLE
    .\MigratingToExpressConfiguration.ps1 -SubscriptionId "25b642fc-05c3-11ed-b939-0242ac120002" -ResourceGroupName "ResourceGroup01" -ServerName "Server01" -Force
    .\MigratingToExpressConfiguration.ps1 -SubscriptionId "25b642fc-05c3-11ed-b939-0242ac120002" -ResourceGroupName "ResourceGroup01" -ServerName "Server01"

#>

param
(
    [Parameter(Mandatory = $True)]
    [string]$SubscriptionId,

    [Parameter(Mandatory = $True)]
    [string]$ResourceGroupName,

    [Parameter(Mandatory = $True)]
    [string]$ServerName,

    [Parameter(Mandatory = $False)]
    [switch]$Force,

    [Parameter(Mandatory = $True)]
    [string]$TenantId,

    [Parameter(Mandatory = $True)]
    [string]$ClientId,

    [Parameter(Mandatory = $True)]
    [string]$ClientSecret
)

###### New SQL Vulnerability Assessment Commands ######
#######################################################
function ClearSqlVulnerabilityAssessmentMasterSetting($SubscriptionId, $ResourceGroupName, $ServerName) {
    $Uri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Sql/servers/$ServerName/databases/master/VulnerabilityAssessments/default?api-version=2021-11-01"
    return SendRestRequest -Method "Delete" -Uri $Uri
}

function GetSqlVulnerabilityAssessmentMasterSetting($SubscriptionId, $ResourceGroupName, $ServerName) {
    $Uri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Sql/servers/$ServerName/databases/master/VulnerabilityAssessments/default?api-version=2021-11-01"
    return SendRestRequest -Method "Get" -Uri $Uri
}

function GetSqlVulnerabilityAssessmentServerSetting($SubscriptionId, $ResourceGroupName, $ServerName) {
    $Uri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Sql/servers/$ServerName/sqlVulnerabilityAssessments/default?api-version=2022-02-01-preview"
    return SendRestRequest -Method "Get" -Uri $Uri
}

function SetSqlVulnerabilityAssessmentServerSetting($SubscriptionId, $ResourceGroupName, $ServerName) {
    $Uri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Sql/servers/$ServerName/sqlVulnerabilityAssessments/default?api-version=2022-02-01-preview"
    $Body = @{
        properties = @{
            state = "Enabled"
        }
    }

    $Body = $Body | ConvertTo-Json
    return SendRestRequest -Method "Put" -Uri $Uri -Body $Body
}

function SetSqlVulnerabilityAssessmentBaselineOnUserDatabase($SubscriptionId, $ResourceGroupName, $ServerName, $DatabaseName, $Baseline) {
    $Uri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Sql/servers/$ServerName/databases/$DatabaseName/sqlVulnerabilityAssessments/default/baselines/default?api-version=2022-02-01-preview"
    $convertedBaseline = $Baseline | ConvertFrom-Json
    $properties = @{
        properties = @{
            latestScan = $false
            results    = @{}
        }
    }

    if ($convertedBaseline.RuleBaselines.Count -eq 0) {
        # baseline is null/empty. No need to send the API call.
        return
    }

    foreach ($rule in $convertedBaseline.RuleBaselines) {
        $ruleId = $rule.RuleId
        $expectedResults = $rule.Properties.ExpectedResults

        if ($ruleId -in $VABinaryRules) {
            if ($expectedResults[0][0] -eq "1") {
                $expectedResults[0][0] = "True"
            }
            else {
                $expectedResults[0][0] = "False"
            }
        }

        $properties.properties.results[$ruleId] = $expectedResults
    }

    $Body = $properties | ConvertTo-Json -Depth 5
    return SendRestRequest -Method "Put" -Uri $Uri -Body $Body
}

function SetSqlVulnerabilityAssessmentBaselineOnSystemDatabase($SubscriptionId, $ResourceGroupName, $ServerName, $DatabaseName, $Baseline) {
    $Uri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Sql/servers/$ServerName/sqlVulnerabilityAssessments/default/baselines/default?api-version=2022-02-01-preview&systemDatabaseName=master"
    $convertedBaseline = $Baseline | ConvertFrom-Json
    $properties = @{
        properties = @{
            latestScan = $false
            results    = @{}
        }
    }

    if ($convertedBaseline.RuleBaselines.Count -eq 0) {
        # baseline is null/empty. No need to send the API call.
        return
    }

    foreach ($rule in $convertedBaseline.RuleBaselines) {
        $ruleId = $rule.RuleId
        $expectedResults = $rule.Properties.ExpectedResults

        if ($ruleId -in $VABinaryRules) {
            if ($expectedResults[0][0] -eq "1") {
                $expectedResults[0][0] = "True"
            }
            else {
                $expectedResults[0][0] = "False"
            }
        }

        $properties.properties.results[$ruleId] = $expectedResults
    }

    $Body = $properties | ConvertTo-Json -Depth 5
    return SendRestRequest -Method "Put" -Uri $Uri -Body $Body
}

function RunSqlVulnerabilityAssessmentScanOnUserDatabase($SubscriptionId, $ResourceGroupName, $ServerName, $DatabaseName) {
    $Uri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Sql/servers/$ServerName/databases/$DatabaseName/sqlVulnerabilityAssessments/default/initiateScan?api-version=2022-02-01-preview"
    SendRestRequest -Method "Post" -Uri $Uri
}

function RunSqlVulnerabilityAssessmentScanOnSystemDatabase($SubscriptionId, $ResourceGroupName, $ServerName, $DatabaseName) {
    $Uri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Sql/servers/$ServerName/sqlVulnerabilityAssessments/default/initiateScan?api-version=2022-02-01-preview&systemDatabaseName=$DatabaseName"
    SendRestRequest -Method "Post" -Uri $Uri
}

function GetSqlVulnerabilityAssessmentScanOnUserDatabase($SubscriptionId, $ResourceGroupName, $ServerName, $DatabaseName) {
    $Uri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Sql/servers/$ServerName/databases/$DatabaseName/sqlVulnerabilityAssessments/default/scans/latest?api-version=2022-02-01-preview"
    return SendRestRequest -Method "Get" -Uri $Uri
}

function GetSqlVulnerabilityAssessmentScanOnSystemDatabase($SubscriptionId, $ResourceGroupName, $ServerName, $DatabaseName) {
    $Uri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Sql/servers/$ServerName/sqlVulnerabilityAssessments/default/scans/latest?api-version=2022-02-01-preview&systemDatabaseName=$DatabaseName"
    return SendRestRequest -Method "Get" -Uri $Uri
}

function SendRestRequest(
    [Parameter(Mandatory = $True)]
    [string] $Method,
    [Parameter(Mandatory = $True)]
    [string] $Uri,
    [parameter( Mandatory = $false )]
    [string] $Body = "DEFAULT") {
        $Params = @{
            Method       = $Method
            Path         = $Uri
        }
    
        if (!($Body -eq "DEFAULT")) {
            $Params = @{
                Method       = $Method
                Path         = $Uri
                Payload      = $Body
            }
        }
    
        Invoke-AzRestMethod @Params
    }
    
    #######################################################
    
    function LogMessage {
        [CmdletBinding()]
        Param
        (
            [Parameter(Mandatory=$true, Position=0)]
            [string]$LogMessage
        )
    
        Write-Host ("{0} - {1}" -f (Get-Date), $LogMessage)
    }
    
    function LogError {
        [CmdletBinding()]
        Param
        (
            [Parameter(Mandatory=$true, Position=0)]
            [string]$LogMessage
        )
    
        Write-Error ("{0} - {1}" -f (Get-Date), $LogMessage)
    }
    
    #######################################################
    
    function Retry() {
        param(
            [Parameter(Mandatory = $true)][Action]$action,
            [Parameter(Mandatory = $false)][int]$maxAttempts = 3
        )
    
        $attempts = 1
        do {
            try {
                $result = $action.Invoke();
                return $result
            }
            catch [Exception] {
                LogMessage -LogMessage $_.Exception.Message
            }
    
            # exponential backoff delay
            $attempts++
            if ($attempts -le $maxAttempts) {
                $retryDelaySeconds = [math]::Pow(2, $attempts)
                $retryDelaySeconds = $retryDelaySeconds - 1  # Exponential Backoff Max == (2^n)-1
                LogMessage -LogMessage ("Action failed. Waiting " + $retryDelaySeconds + " seconds before attempt " + $attempts + " of " + $maxAttempts + ".")
                Start-Sleep $retryDelaySeconds
            }
            else {
                LogError $_.Exception.Message
                $ex = New-Object System.Exception($_.Exception.Message)
                throw $ex
            }
        } while ($attempts -le $maxAttempts)
    }
    
    function HaveVulnerabilityAssessmentSetting($ResourceGroupName, $ServerName, $Databases) {
        # Check if we have a server setting.
        LogMessage -LogMessage "Check Vulnerability Assessment setting for '$($ServerName)' server"
        $vaServerSetting = Get-AzSqlServerVulnerabilityAssessmentSetting -ResourceGroupName $ResourceGroupName -ServerName $ServerName
        if (![string]::IsNullOrEmpty($vaServerSetting.StorageAccountName)) {
            return $true
        }
    
        # Check if we have a database setting for server
        foreach ($database in $Databases) {
            LogMessage -LogMessage "Check VA settings for '$($database.DatabaseName)' database"
            $vaDatabaseSetting = Get-AzSqlDatabaseVulnerabilityAssessmentSetting -ResourceGroupName $ResourceGroupName -ServerName $ServerName -DatabaseName $database.DatabaseName
            if (![string]::IsNullOrEmpty($vaDatabaseSetting.StorageAccountName)) {
                return $true
            }
        }
    
        # Check if we have a master setting for server
        $vaSettingMaster =  GetSqlVulnerabilityAssessmentMasterSetting -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName
        $object = ConvertFrom-Json $vaSettingMaster.Content
        if (![string]::IsNullOrEmpty($object.properties.storageContainerPath)) {
            return $true
        }
    
        return $false
    }
    
    function HaveExpressConfigurationVulnerabilityAssessmentSetting($SubscriptionId, $ResourceGroupName, $ServerName) {
        # Check if we have a server setting.
        LogMessage -LogMessage "Check express configuration Vulnerability Assessment setting for '$($ServerName)' server"
        $Response = GetSqlVulnerabilityAssessmentServerSetting -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName
        if ($Response.Content.Contains("Enabled")) {
            return $true
        }
    
        return $false
    }
    
    function GetBlobsFromStorage($ContainerName, $Context) {
        # Use Get-AzStorageBlob to retrieve a list of blobs from the container
        $blobs = Get-AzStorageBlob -Container $ContainerName -Context $Context
    
        if ([string]::IsNullOrEmpty($blobs)) {
            return
        }
    
        return $blobs
    }
    
    function ExtractBaselineBlobName($ServerName, $DatabaseName) {
        $prefix = "scans/$ServerName/$DatabaseName/baseline"
    
        # Filter the list to include only blobs with names starting with "baseline"
        $baselineBlobs = $blobs | Where-Object { $_.Name.StartsWith($prefix) }
    
        if ($baselineBlobs.Count -eq 0) {
            return
        }
        else {
            # Sort the list by LastModified descending and get the first item
            $mostRecentBlob = $baselineBlobs | Sort-Object LastModified -Descending | Select-Object -First 1
    
            # Get the name of the most recent blob
            $mostRecentBlobName = $mostRecentBlob.Name
    
            LogMessage -LogMessage "The most recent blob with the 'baseline' prefix is: $mostRecentBlobName"
    
            return $mostRecentBlobName
        }
    }
    
    function ReadDatabaseBaselineFromStorage($ServerName, $BlobName, $ContainerName, $Context) {
        # Use Get-AzStorageBlobContent to retrieve the blob content as a string
        $blobContent = Get-AzStorageBlobContent -Blob $BlobName -Container $ContainerName -Context $Context
        $blobContent = $blobContent.ICloudBlob.DownloadText()
    
        if ([string]::IsNullOrEmpty($blobContent)) {
            return
        }
    
        return $blobContent
    }
    
    function GetBaselineConfigurationForDatabase($ServerName, $DatabaseName, $VaDatabasePolicyStorage, $VaDatabasePolicyContainer) {
        # Get storage account context
        $ctx = New-AzStorageContext -StorageAccountName $VaDatabasePolicyStorage
    
        # Extract baseline blob name
        $blobs = GetBlobsFromStorage -ContainerName $VaDatabasePolicyContainer -Context $ctx
        if ([string]::IsNullOrEmpty($blobs)) {
            $ex = New-Object System.Exception("Failed to get blobs from $($VaDatabasePolicyStorage) storage. Verify that you have Storage Blob Data Reader role assignment on the storage.")
            throw $ex
        }
    
        # Extract baseline blob name
        $blobName = ExtractBaselineBlobName -ServerName $ServerName -DatabaseName $DatabaseName
        if ([string]::IsNullOrEmpty($blobName)) {
            LogMessage -LogMessage "No baseline blob was found for $($DatabaseName) database."
            return
        }
    
        # Extract the baseline
        $baseline = ReadDatabaseBaselineFromStorage -ServerName $ServerName -BlobName $blobName -ContainerName $VaDatabasePolicyContainer -Context $ctx
        if ([string]::IsNullOrEmpty($baseline)) {
            $ex = New-Object System.Exception("Failed to get blobs from $($VaDatabasePolicyStorage) storage. Verify that you have Storage Blob Data Reader role assignment on the storage.")
            throw $ex
        }
    
        LogMessage -LogMessage "Found baseline for $($DatabaseName) database."
        return $baseline
    }
    
    function ClearBaselineFolder() {
        # Clean baseline folder
        $scriptPath = Get-Location
        $folderPath = Join-Path -Path $scriptPath.Path -ChildPath "scans"
    
        if (Test-Path $folderPath) {
            # Remove folder from previous runs
            Remove-Item $folderPath -Recurse
        }
    }
    
    #######################################################
    
    $VABinaryRules = @("VA1018", "VA1022", "VA1023", "VA1024", "VA1043", "VA1044", "VA1045", "VA1051", "VA1052", "VA1053", "VA1058", "VA1059", "VA1067", "VA1071", "VA1072", "VA1091", "VA1092", "VA1093", "VA1102", "VA1143", "VA1219", "VA1230", "VA1235", "VA1245", "VA1264", "VA1265", "VA1277", "VA1279", "VA1283", "VA2060", "VA2061", "VA2121", "VA2122", "VA2128")
    
    #######################################################


    # Authenticate using Service Principal
    $securePassword = ConvertTo-SecureString -String $ClientSecret -AsPlainText -Force
    $credential = New-Object -TypeName "System.Management.Automation.PSCredential" -ArgumentList $ClientId, $securePassword
    $subscription = Connect-AzAccount -ServicePrincipal -Tenant $TenantId -Credential $credential -Subscription $SubscriptionId
    
    if ([string]::IsNullOrEmpty($subscription))
    {
        LogError "Failed to get the subscription. Migration cancelled. Fix errors and try again later."
        return
    }
    
    $srv = Get-AzSqlServer -ResourceGroupName $ResourceGroupName -ServerName $ServerName
    if ([string]::IsNullOrEmpty($srv))
    {
        LogError "The server was not found. Migration cancelled. Fix errors and try again later."
        return
    }
    
    ClearBaselineFolder
    
    $baselines = @{}
    #$databases = Get-AzSqlDatabase -ResourceGroupName
    $databases = Get-AzSqlDatabase -ResourceGroupName $ResourceGroupName -ServerName $ServerName | Where-Object { $_.DatabaseName -ne "master" }
$haveVaSetting = HaveVulnerabilityAssessmentSetting -ResourceGroupName $ResourceGroupName -ServerName $ServerName -Databases $databases
$haveExpressConfigurationVA = HaveExpressConfigurationVulnerabilityAssessmentSetting -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName

if ($haveExpressConfigurationVA) {
    LogMessage -LogMessage "Express configuration vulnerability assessment setting is already exist on this server. Cancelling script."
    return
}

if ($haveVaSetting) {
    # Get server policy container path
    $vaServerSetting = Get-AzSqlServerVulnerabilityAssessmentSetting -ResourceGroupName $ResourceGroupName -ServerName $ServerName
    $vaServerPolicyStorage = $vaServerSetting.StorageAccountName
    $vaServerPolicyContainer = $vaServerSetting.ScanResultsContainerName

    $canRemoveVa = $true

    # Go over each database and get the baseline (is exist).
    $i = 0
    foreach ($database in $databases) {
        $i += 1
        $completed = ($i/$databases.count) * 100
        Write-Progress -Activity "Processing" -Status "Progress:" -PercentComplete $completed
        LogMessage -LogMessage "Starting to fetch baseline for $($database.DatabaseName) database."
        $vaDatabaseSetting = Get-AzSqlDatabaseVulnerabilityAssessmentSetting -ResourceGroupName $ResourceGroupName -ServerName $ServerName -DatabaseName $database.DatabaseName
        $adsDatabasePolicy = Get-AzSqlDatabaseAdvancedThreatProtectionSetting -ResourceGroupName $ResourceGroupName  -ServerName $ServerName -DatabaseName $database.DatabaseName
        $adsDatabasePolicyEnabled = ($null -ne $adsDatabasePolicy.ThreatDetectionState -and $adsDatabasePolicy.ThreatDetectionState -eq "Enabled") -or ($null -ne $adsDatabasePolicy.AdvancedThreatProtectionState -and $adsDatabasePolicy.AdvancedThreatProtectionState -eq "Enabled") # Handle breaking changes in the command.
        $containsDatabasePolicy = $adsDatabasePolicyEnabled -and ![string]::IsNullOrEmpty($vaDatabaseSetting.StorageAccountName)

        if ($containsDatabasePolicy) {
            # The database has database policy. Using the database policy storage.
            $vaDatabasePolicyStorage = $vaDatabaseSetting.StorageAccountName
            $vaDatabasePolicyContainer = $vaDatabaseSetting.ScanResultsContainerName
        }
        else {
            $vaDatabasePolicyStorage = $vaServerPolicyStorage
            $vaDatabasePolicyContainer = $vaServerPolicyContainer
        }

        try {
            $baseline = GetBaselineConfigurationForDatabase -ServerName $ServerName -DatabaseName $database.DatabaseName -VaDatabasePolicyStorage $vaDatabasePolicyStorage -VaDatabasePolicyContainer $vaDatabasePolicyContainer
            $baselines[$database.DatabaseName] = $baseline
            LogMessage -LogMessage "Finished to fetch baseline for $($database.DatabaseName) database."
        }
        catch {
            LogError "An error occurred: $($_.Exception.Message) while handling $($database.DatabaseName) database."
            $canRemoveVa = $false
        }
    }
    Write-Progress -Activity "Processing" -Status "Ready" -Completed

    LogMessage -LogMessage "Starting to fetch baseline for master database."
    $vaSettingMaster =  GetSqlVulnerabilityAssessmentMasterSetting -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName
    $vaSettingMaster = ConvertFrom-Json $vaSettingMaster.Content

    $adsDatabasePolicy = Get-AzSqlDatabaseAdvancedThreatProtectionSetting -ResourceGroupName $ResourceGroupName  -ServerName $ServerName -DatabaseName "master"
    $adsDatabasePolicyEnabled = ($null -ne $adsDatabasePolicy.ThreatDetectionState -and $adsDatabasePolicy.ThreatDetectionState -eq "Enabled") -or ($null -ne $adsDatabasePolicy.AdvancedThreatProtectionState -and $adsDatabasePolicy.AdvancedThreatProtectionState -eq "Enabled") # Handle breaking changes in the command.
    $containsDatabasePolicy = $adsDatabasePolicyEnabled -and ![string]::IsNullOrEmpty($vaSettingMaster.properties.storageContainerPath)

    if ($containsDatabasePolicy -and ![string]::IsNullOrEmpty($vaServerPolicyStorage)) {
        # The database has database policy. Using the database policy storage.
        $vaDatabasePolicyStorage = $vaSettingMaster.properties.storageContainerPath.Split("/")[2].Split(".")[0]
        $vaDatabasePolicyContainer = $vaSettingMaster.properties.storageContainerPath.Split("/")[3]
    }
    else {
        $vaDatabasePolicyStorage = $vaServerPolicyStorage
        $vaDatabasePolicyContainer = $vaServerPolicyContainer
    }

    try {
        $baseline = GetBaselineConfigurationForDatabase -ServerName $ServerName -DatabaseName "master" -VaDatabasePolicyStorage $vaDatabasePolicyStorage -VaDatabasePolicyContainer $vaDatabasePolicyContainer
        $baselines["master"] = $baseline
        LogMessage -LogMessage "Finished to fetch baseline for master database."
    }
    catch {
        LogError "An error occurred: $($_.Exception.Message) while handling master database."
        $canRemoveVa = $false
    }

    ClearBaselineFolder

    if ($canRemoveVa) {
        if (!$Force) {
            LogMessage -LogMessage "We are going to remove the current Vulnerability Assessment settings for this server and underlying databases."
            $Confirmation = Read-Host -Prompt "Do you approve (y/n)?"
            if ($Confirmation -ne "y") {
                LogMessage -LogMessage "You chose to stop the migration process. Existing VA settings will not be changed."
                return
            }
        }

        # Clear all server and databases policies
        $i = 0
        foreach ($database in $databases) {
            $i += 1
            $completed = ($i/$databases.count) * 100
            Write-Progress -Activity "Processing" -Status "Progress:" -PercentComplete $completed
            LogMessage -LogMessage "Clear Vulnerability Assessment setting for '$($database.DatabaseName)' database."
            Clear-AzSqlDatabaseVulnerabilityAssessmentSetting -ResourceGroupName $ResourceGroupName -ServerName $ServerName -DatabaseName $database.DatabaseName
        }
        Write-Progress -Activity "Processing" -Status "Ready" -Completed

        # Removing Vulnerability Assessment settings from master db
        Retry -action { ClearSqlVulnerabilityAssessmentMasterSetting -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName }

        # Removing old server Vulnerability Assessment setting
        LogMessage -LogMessage "Clear Vulnerability Assessment setting for '$($ServerName)' server."
        Clear-AzSqlServerVulnerabilityAssessmentSetting -ResourceGroupName $ResourceGroupName -ServerName $ServerName
    }
    else {
        LogError "Migration cancelled. Fix errors and try again later."
        return
    }
}

# Set new SQL Vulnerability Assessment Setting
LogMessage -LogMessage "Add express configuration Vulnerability Assessment feature setting for '$($ServerName)' server."
$Response = SetSqlVulnerabilityAssessmentServerSetting -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName
$successStatusCodes = @(200, 201, 202)
if ($Response.StatusCode -in $successStatusCodes) {
    LogMessage -LogMessage "Congratulations, your server '$($ServerName)' server is set up with express configuration Vulnerability Assessment feature"
}
else {
    LogMessage -LogMessage "There was a problem to enable express configuration Vulnerability Assessment feature on the '$($ServerName)' server. Error '$($Response.StatusCode)': '$($Response.Content)'"
    return
}

# Run a new scan on all the databases
$i = 0
foreach ($database in $databases) {
    $i += 1
    $completed = ($i/$databases.count) * 100
    Write-Progress -Activity "Processing" -Status "Progress:" -PercentComplete $completed
    LogMessage -LogMessage "Run scan on '$($database.DatabaseName)' database."
    Retry -action { RunSqlVulnerabilityAssessmentScanOnUserDatabase -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName -DatabaseName $database.DatabaseName }
}
Write-Progress -Activity "Processing" -Status "Ready" -Completed

LogMessage -LogMessage "Run scan on 'master' database."
Retry -action { RunSqlVulnerabilityAssessmentScanOnSystemDatabase -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName -DatabaseName "master" }

LogMessage -LogMessage "Wait for scan results.."
Start-Sleep 60

# Wait for scan results
$i = 0
foreach ($database in $databases) {
    $i += 1
    $completed = ($i/$databases.count) * 100
    Write-Progress -Activity "Processing" -Status "Progress:" -PercentComplete $completed
    try {
        LogMessage -LogMessage "Waiting for results for $($database.DatabaseName) database."
        Retry -action { GetSqlVulnerabilityAssessmentScanOnUserDatabase -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName -DatabaseName $database.DatabaseName }
        LogMessage -LogMessage "Received results for $($database.DatabaseName) database."
    }
    catch {
        LogMessage -LogMessage "Failed to get latest scan results for $($database.DatabaseName). Stopping the migration."
        LogMessage -LogMessage "You can revert back to classic configuration. For more information: https://learn.microsoft.com/azure/defender-for-cloud/sql-azure-vulnerability-assessment-manage?tabs=express#revert-back-to-the-classic-configuration"
        return
    }
}
Write-Progress -Activity "Processing" -Status "Ready" -Completed

try {
    LogMessage -LogMessage "Waiting for results for master database"
    Retry -action { GetSqlVulnerabilityAssessmentScanOnSystemDatabase -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName -DatabaseName "master" }
}
catch {
    LogMessage -LogMessage "Failed to get latest scan results for master. Stopping the migration"
    LogMessage -LogMessage "You can revert back to classic configuration. For more information: https://learn.microsoft.com/azure/defender-for-cloud/sql-azure-vulnerability-assessment-manage?tabs=express#revert-back-to-the-classic-configuration"
    return
}

# Apply baselines from each database
$successMigration = @()
$failedMigration = @()
if (!$haveVaSetting) {
    # no need to migrate baseline as there is no baseline to extract.
    $successMigration = $databases
}
else {
    $i = 0
    foreach ($database in $databases) {
        $i += 1
        $completed = ($i/$databases.count) * 100
        Write-Progress -Activity "Processing" -Status "Progress:" -PercentComplete $completed
        try {
            if (![string]::IsNullOrEmpty($baselines[$database.DatabaseName])) {
                LogMessage -LogMessage "Applying baseline for '$($database.DatabaseName)' database."
                Retry -action { SetSqlVulnerabilityAssessmentBaselineOnUserDatabase -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName -DatabaseName $database.DatabaseName -Baseline $baselines[$database.DatabaseName] }
            }
            LogMessage -LogMessage "Baseline was successfully applied for '$($database.DatabaseName)' database."
            $successMigration += $database.DatabaseName
        }
        catch {
            LogError "Failed to set baseline for $($database.DatabaseName) database."
            $failedMigration += $database.DatabaseName
        }
    }
    Write-Progress -Activity "Processing" -Status "Ready" -Completed

    try {
        if (![string]::IsNullOrEmpty($baselines["master"])) {
            LogMessage -LogMessage "Applying baseline for 'master' database."
            Retry -action { SetSqlVulnerabilityAssessmentBaselineOnSystemDatabase -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName -DatabaseName "master" -Baseline $baselines["master"] }
            $successMigration += "master"
        }
    }
    catch {
        LogError "Failed to set baseline for master database."
        $failedMigration += "master"
    }
}

if ($successMigration.Count -eq 0) {
    LogError "Failed to set baseline for all the databases."
    LogMessage -LogMessage "You can revert back to classic configuration. For more information: https://learn.microsoft.com/azure/defender-for-cloud/sql-azure-vulnerability-assessment-manage?tabs=express#revert-back-to-the-classic-configuration"
}
elseif ($failedMigration.Count -eq 0) {
    LogMessage -LogMessage  "The migration process completed successfully."
}
else {
    LogMessage -LogMessage  "The migration process completed. The migration was successful for $($successMigration -join ',') and unsuccessful for $($failedMigration -join ',')"
    LogMessage -LogMessage "You can revert back to classic configuration. For more information: https://learn.microsoft.com/azure/defender-for-cloud/sql-azure-vulnerability-assessment-manage?tabs=express#revert-back-to-the-classic-configuration"
}
```

Sample output <br>
![image](https://github.com/user-attachments/assets/19c93408-a20c-4143-8892-383c27c21429)

### Script(enable express configuration directly)
```powershell
# Requires -Modules @{ ModuleName="Az.Sql"; ModuleVersion="3.11.0" }
# Requires -Modules @{ ModuleName="Az.Accounts"; ModuleVersion="2.9.1" }
# Requires -Version 5.1
 
<#
.SYNOPSIS
    This script configures an Azure SQL Server to use the express configuration Vulnerability Assessment feature.
 
.DESCRIPTION
This script enables the express configuration Vulnerability Assessment feature by executing the following steps:
- It checks if the express configuration is already enabled.
- If not, it enables the express configuration for the specified SQL Server.
 
.PARAMETER ServerSubscriptionId
    The Subscription id that the server belongs to
 
.PARAMETER ServerResourceGroupName
    The Resource Group that the server belongs to
 
.PARAMETER ServerName
    The SQL server name that we want to apply the new SQL Vulnerability Assessment policy to.
 
.PARAMETER TenantId
    The Tenant ID for the Azure AD.
 
.PARAMETER ClientId
    The Client ID of the service principal.
 
.PARAMETER ClientSecret
    The Client Secret of the service principal.
 
.EXAMPLE
    .\EnableExpressConfiguration.ps1 -SubscriptionId "25b642fc-05c3-11ed-b939-0242ac120002" -ResourceGroupName "ResourceGroup01" -ServerName "Server01" -TenantId "your-tenant-id" -ClientId "your-client-id" -ClientSecret "your-client-secret"
#>
 
param
(
    [Parameter(Mandatory = $True)]
    [string]$SubscriptionId,
 
    [Parameter(Mandatory = $True)]
    [string]$ResourceGroupName,
 
    [Parameter(Mandatory = $True)]
    [string]$ServerName,
 
    [Parameter(Mandatory = $True)]
    [string]$TenantId,
 
    [Parameter(Mandatory = $True)]
    [string]$ClientId,
 
    [Parameter(Mandatory = $True)]
    [string]$ClientSecret
)
 
function GetSqlVulnerabilityAssessmentServerSetting($SubscriptionId, $ResourceGroupName, $ServerName) {
    $Uri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Sql/servers/$ServerName/sqlVulnerabilityAssessments/default?api-version=2022-02-01-preview"
    return SendRestRequest -Method "Get" -Uri $Uri
}
 
function SetSqlVulnerabilityAssessmentServerSetting($SubscriptionId, $ResourceGroupName, $ServerName) {
    $Uri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Sql/servers/$ServerName/sqlVulnerabilityAssessments/default?api-version=2022-02-01-preview"
    $Body = @{
        properties = @{
            state = "Enabled"
        }
    }
 
    $Body = $Body | ConvertTo-Json
    return SendRestRequest -Method "Put" -Uri $Uri -Body $Body
}
 
function SendRestRequest(
    [Parameter(Mandatory = $True)]
    [string] $Method,
    [Parameter(Mandatory = $True)]
    [string] $Uri,
    [parameter( Mandatory = $false )]
    [string] $Body = "DEFAULT") {
        $Params = @{
            Method       = $Method
            Path         = $Uri
        }
        if (!($Body -eq "DEFAULT")) {
            $Params = @{
                Method       = $Method
                Path         = $Uri
                Payload      = $Body
            }
        }
        Invoke-AzRestMethod @Params
    }
 
function LogMessage {
    [CmdletBinding()]
    Param
    (
        [Parameter(Mandatory=$true, Position=0)]
        [string]$LogMessage
    )
 
    Write-Host ("{0} - {1}" -f (Get-Date), $LogMessage)
}
 
function LogError {
    [CmdletBinding()]
    Param
    (
        [Parameter(Mandatory=$true, Position=0)]
        [string]$LogMessage
    )
 
    Write-Error ("{0} - {1}" -f (Get-Date), $LogMessage)
}
 
function HaveExpressConfigurationVulnerabilityAssessmentSetting($SubscriptionId, $ResourceGroupName, $ServerName) {
    # Check if we have a server setting.
    LogMessage -LogMessage "Check express configuration Vulnerability Assessment setting for '$($ServerName)' server"
    $Response = GetSqlVulnerabilityAssessmentServerSetting -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName
    if ($Response.Content.Contains("Enabled")) {
        return $true
    }
 
    return $false
}
 
# Authenticate using Service Principal
$securePassword = ConvertTo-SecureString -String $ClientSecret -AsPlainText -Force
$credential = New-Object -TypeName "System.Management.Automation.PSCredential" -ArgumentList $ClientId, $securePassword
$subscription = Connect-AzAccount -ServicePrincipal -Tenant $TenantId -Credential $credential -Subscription $SubscriptionId
 
if ([string]::IsNullOrEmpty($subscription))
{
    LogError "Failed to get the subscription. Migration cancelled. Fix errors and try again later."
    return
}
 
$srv = Get-AzSqlServer -ResourceGroupName $ResourceGroupName -ServerName $ServerName
if ([string]::IsNullOrEmpty($srv))
{
    LogError "The server was not found. Migration cancelled. Fix errors and try again later."
    return
}
 
$haveExpressConfigurationVA = HaveExpressConfigurationVulnerabilityAssessmentSetting -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName
 
if ($haveExpressConfigurationVA) {
    LogMessage -LogMessage "Express configuration vulnerability assessment setting is already exist on this server. Cancelling script."
    return
}
 
# Set new SQL Vulnerability Assessment Setting
LogMessage -LogMessage "Add express configuration Vulnerability Assessment feature setting for '$($ServerName)' server."
$Response = SetSqlVulnerabilityAssessmentServerSetting -SubscriptionId $SubscriptionId -ResourceGroupName $ResourceGroupName -ServerName $ServerName
$successStatusCodes = @(200, 201, 202)
if ($Response.StatusCode -in $successStatusCodes) {
    LogMessage -LogMessage "Congratulations, your server '$($ServerName)' server is set up with express configuration Vulnerability Assessment feature"
}
else {
    LogMessage -LogMessage "There was a problem to enable express configuration Vulnerability Assessment feature on the '$($ServerName)' server. Error '$($Response.StatusCode)': '$($Response.Content)'"
    return
}
```