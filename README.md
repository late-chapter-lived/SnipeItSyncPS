# SnipeItSyncPS
 User and asset syncing solution in PowerShell utilizing Snipe-It's REST API.

All functions have been tested working successfully in our environment for syncing users from AD and assets from exported SCCM reports and from AD. Your mileage may vary.

The scripts require snazy2000's excellent [SnipeitPS](https://github.com/snazy2000/SnipeitPS) PowerShell module.

## Usage
`SnipeIt-AD-Sync.ps1` and `SnipeIt-Asset-Sync.ps1` are both example scripts which sync users and assets respectively. 

`SnipeIt-Import-Assets.ps1` is another example script that will sync information from a given CSV, acting as an alternative to the built-in importer.

You must set `$CREDXML_PATH='\path\to\your\creds.xml'` at the top of each of the two Sync files, as well setting `$ENABLE_SYNC=$true` when ready to start syncing. You may use `Export-SnipeItCredentials -File "\path\to\your\creds.xml" -URL "<URL>" -APIKey "<APIKEY>"` to export the credentials encrypted under your current user account.

`Connect-SnipeIt` will connect to your Snipe-It instance using the given credentials. This function also setups the cache. It optionally supports a `-IgnoreSelfSignedCert` parameter for instances using self-signed certificates.

**At minimum, to start each script syncing it requires setting at the top of the file:**
- A valid `$CREDXML_PATH`.
- `$ENABLE_SYNC=$true`

### SnipeIt-AD-Sync.ps1
The `SnipeIt-AD-Sync.ps1` script syncs AD users with snipe-it. It should work with minimal setup required.

### SnipeIt-Asset-Sync.ps1
`SnipeIt-Asset-Sync.ps1` must be tailored to each environment. It was tested with SCCM reports exported to a fileshare as described [here](https://docs.microsoft.com/en-us/mem/configmgr/core/servers/manage/operations-and-maintenance-for-reporting#bkmk_subscription), but you could also do a WQL query direct against the SCCM server (an example is given in `SnipeIt-Asset-Sync.ps1`), or query computers by WMI directly (using `Import-AssetFromWMI` found in the same script).

### SnipeIt-Import-Assets.ps1
`SnipeIt-Import-Assets.ps1` syncs information from a given CSV file. Results will be emailed to the file's creator/owner by default. This allows more fine-tuned control over importing assets. By editing the script one can possibly set criteria required for editing certain fields or assets. In a future release the script will have built-in support for restricting access based on group membership.

## Design
### Working off Cache
The functions are designed to work off cache as much as possible to minimize REST API calls. The maximum age of the cache can be set when calling `Initialize-SnipeItCache` for multiple entities, or `Get-SnipeItEntityAll` for a single one. Default is 120 minutes.

You may also give `-NoCache` to most function calls to avoid using cache, though this isn't recommended except for testing purposes.

### Exceptions and Errors
Functions were designed to throw an error if they reach an invalid state. The most common one would be `[System.Net.WebException]` thrown from an error StatusCode returned from the API, often from invalid input or attempting to insert a duplicate value into a unique field. The function help for each function details all possible custom exceptions thrown. Both example scripts use a try/catch block to allow syncing to continue on error, but you're also free to use use `-ErrorAction Continue` if you just want to output the errors.

All API calls will be retried up to 3 times by default if the following status codes are returned: "429", "Too Many Requests", "422", or "Unprocessable Entity". From my own experience sometimes instances can return one of these codes when the data is otherwise good. You can adjust the behavior with the `-OnErrorRetry` parameter, which will then be passed to any functions called internally. The `-SleepMS` parameter also determines how long to sleep between each API call (default is 1000ms).

### Logging
To log messages for each user or asset synced, it's recommended to use the `-Verbose` switch with every `Sync-` and `Remove-` command.

### String Decoding & Trimming
All strings are run through `HTMLDecode` before comparion, since most (if not all) will be returned by the API encoded. All names and other supposedly unique values (such as username, asset_tag, etc.) are also trimmed. The `-Trim` parameter may be given to `Sync-SnipeItUser` and `Sync-SnipeItAsset` to trim ALL strings.

### Filtering by name and creating new Snipe-It entities if not found
Snipe-It "Entities" (aka "objects") like "departments", "companies", etc. (basically anything with an ID) will be created automatically if given the right parameters. For most objects this is just the name, but some require more information, such as Categories, Models, Users, and Assets.

In case searching by name returns multiple results, it will use any other parameters given to try and narrow down the search. So if you have say two models with the same model name, it will try filtering by `model_number`, if given. If it still has multiple results it will use the first one found in most cases (except for users and assets).

## Function Help
Use `get-help <Function> -Full` for more information about each function (e.g. `get-help Sync-SnipeItUser -Full`). You must dot-source `SnipeIt-Sync-PS.ps1` at least once to register its function help.

## Main Functions in `SnipeIt-Sync-PS.ps1`
- `Export-SnipeItCredentials` -- Used for exporting your Snipe-It API credentials to an encrypted XML file.
- `Connect-SnipeIt` -- Used to connect to your Snipe-It instance.
- `Initialize-SnipeItCache` -- Used to fill the given caches with values from Snipe-It.
- `Sync-SnipeItUser` -- For syncing users.
- `Remove-SnipeItInactiveUsers` -- Used to flag/purge/report inactive users.
- `Sync-SnipeItDeptUsers` -- Allows for creating special users for each department, to allow assigning assets to departments.
- `Sync-SnipeItAsset` -- For syncing assets. This can also checkout synced assets as well.
- `Format-SnipeItAsset` -- For formatting assets for output to file (IE, CSV).
- `Format-SnipeItEntity` -- For formatting other snipe-it entities for output to file (IE, CSV).
- `Remove-SnipeItInactiveEntity` -- Removing inactive/unassigned snipe-it objects other than users and assets (models, departments, etc.)
- `Update-SnipeItInactiveUserReassignment` -- Reassigns assets from inactive users to departmental users, allowing the inactive user to be deleted from snipe-it.

## All Other Functions in `SnipeIt-Sync-PS.ps1`
These functions are mainly used internally.

- `Get-SnipeItCustomFieldMap`
- `Restore-SnipeItAssetCustomFields`
- `Get-SnipeItEntityAll`
- `Update-SnipeItCache`
- `Get-SnipeItApiEntityByName`
- `Select-SnipeItFilteredEntity`
- `Get-SnipeItEntityByName`
- `Get-SnipeItCompanyByName`
- `Get-SnipeItDepartmentByName`
- `Get-SnipeItLocationByName`
- `Get-SnipeItCategoryByName`
- `Get-SnipeItManufacturerByName`
- `Get-SnipeItSupplierByName`
- `Get-SnipeItFieldsetByName`
- `Get-SnipeItCustomFieldByName`
- `Get-SnipeItStatuslabelByName`
- `Get-SnipeItModelByName`
- `Get-SnipeItEntityByID`
- `Get-SnipeItAssetEx`
- `Get-SnipeItUserEx`

## Reporting bugs and issues
 Please use both `-Verbose` and `-Debug` switches with your function calls and include the information in your ticket (feel free to redact any identifable information). You may use `$DebugPreference='Continue'` at the top of your script to avoid breaking on each `Write-Debug` call.
