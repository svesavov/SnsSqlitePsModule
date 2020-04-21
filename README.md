SnsSqlitePsModule PowerShell Module
=============

This is a PowerShell module for working with [SQLite](https://www.sqlite.org) DataBases, based on Warren Frame's (RamblingCookieMonster) and his contributors project, named ["PSSQLite"](https://github.com/RamblingCookieMonster/PSSQLite).
The reason to make my own Binary PSModule written on C# is mainly related with performance as it can be seen on the screenshots:
![Performance Test Generate Data](/Media/GenerateInput.JPG)
![Performance Test Inserting Process](/Media/InvokeQryInAction.JPG)
![Performance Test Insert Data](/Media/InsertData.JPG)
As it can be seen binary CmdLet is much faster than PowerShell foreach. Considering the fact that the binary CmdLet have almost thousand lines.

## Functionality

Create a SQLite database and table:
  * ![Create a SQLite database and table](/Media/CreateEventTable.JPG)

Query a SQLite database. The results are filtered in PS to be shown the time required to select the previously inserted amount of data:
  * ![Query a SQLite database](/Media/QueryData.JPG)

Create a SQLite connection, use it for maintenance or subsequent queries to avoid verifications repetition:
  * ![Create 2 SQLite connections with new DataBase creation and Backup](/Media/DbBackup.JPG)

Built-in performance measurement accessible in Verbose stream.
  * ![Truncate SQLite table](/Media/TruncateTable.JPG)

Re-designed Pipeline input - using the pipeline can be performed bulk updates. There is no need of additional command for bulk upload.
  * ![Pipeline Input Using ValueFromPipelineByPropertyName](/Media/PipelineInput.JPG)

## Instructions

1. Download SnsSqlitePsModule.zip
2. Don't forget to check the .ZIP file for viruses and etc.
3. File MD5 hash: 4C8788D9F4D84C9C454422503FD77636
4. Unzip in one of the following folders depending of the preference:
- C:\Users\UserName\Documents\WindowsPowerShell\Modules - Replace "UserName" with the actual username, If you want the module available for specific user
- C:\Program Files\WindowsPowerShell\Modules - If you want the module to be available for all user on the machine
- Or any other location present in $env:PSModulePath
5. Run the following command replacing "PathWhereModuleIsInstalled" with the actual path where the module files were unzipped.
```powershell
Get-ChildItem -Path "PathWhereModuleIsInstalled" -Recurse | Unblock-File
```
6. PowerShell Examples:

```powershell

# Import the Module
Import-Module SnsSqlitePsModule;





# Create "temp.sqlite" DataBase in the working folder and "tbl" table inside the DB
Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "CREATE TABLE tbl (ID INTEGER, Event VARCHAR(20), Date DATETIME)" -Verbose;


# Verify the "tbl" table creation
$Output = Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "PRAGMA table_info(tbl)" -Verbose;
$Output





# Generate large amount of data with PowerShell and measure the time.
$CmdStart = [System.DateTime]::now;
[System.String[]]$Query = @("INSERT INTO tbl (ID, Event, Date) VALUES (@ID, @Event, @Date);");
[System.Collections.Hashtable[]]$Params = @();
1..100000 | ForEach `
{
	$Params += `
	@{
		"ID" = "$($_)";
		"Event" = "Error";
		"Date" = [System.DateTime]::UtcNow.AddDays(0 - $_).ToString([SnsSqlitePsModule.TimeFormat]::Utc)
	};
}
[System.DateTime]::now - $CmdStart;


# Insert the large amount of data in the DataBase
$CmdStart = [System.DateTime]::now;
Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query $Query -SqlParameters $Params;
[System.DateTime]::now - $CmdStart;


# Query the DataBase about the previously inserted data
# The output is limited in PowerShell to allow all the data to be retrieved from the DataBase
# Internal measuring accessible via "Verbose" stream is not used because it will generate thousands of screens
# and will be unable to make the screenshot above
$CmdStart = [System.DateTime]::now;
$Output = Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "SELECT * FROM tbl";
[System.DateTime]::now - $CmdStart;
$Output | Select-Object -First 10;





# Create permanent SQLiteConnection object to the DataBase above
$ProdDbConn = New-SnsSqliteConnection -DataBase "temp.sqlite";


# Create new DataBase and permanent connection to it
$BkpDbConn = New-SnsSqliteConnection -DataBase "Backup.sqlite";


#Backup the old DataBase into the newly created one
$CmdStart = [System.DateTime]::now;
$ProdDbConn.BackupDatabase($BkpDbConn, "main", "main", -1, $null, 0);
[System.DateTime]::now - $CmdStart;


# Verify whether the data is copied to the new database using the existing permanent connection
Invoke-SnsSqliteQuery -SQLiteConnection $BkpDbConn -Query "SELECT COUNT(*) FROM tbl" -Verbose;


# Truncate the data from the table
Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "DELETE FROM tbl;" -Verbose;





# Generate a collection from custom objects which properties to be send via the Pipeline
# to test whether sending via "ValueFromPipelineByPropertyName" is working
[System.Object[]]$Query = @();
for ($intI = 0; $intI -lt 10; $intI++)
{
	[System.Object]$objObject = New-Object -TypeName 'System.Object';
	
	$objObject | Add-Member -Force -MemberType 'NoteProperty' -Name 'Query' -Value `
		"INSERT INTO tbl (ID, Event, Date) VALUES (@ID, @Event, @Date);";
	
	$objObject | Add-Member -Force -MemberType 'NoteProperty' -Name 'SqlParameters' -Value `
	(@{
		"@ID" = $intI
		"Event" = "Error";
		"Date" = [System.DateTime]::UtcNow.AddDays(0 - $intI).ToString([SnsSqlitePsModule.TimeFormat]::Utc)
	});
	
	$Query += $objObject;
}


# Insert the generated data using Pipeline input via "ValueFromPipelineByPropertyName"
$Query | Invoke-SnsSqliteQuery -DataBase "temp.sqlite";


# Verify the Inserting
Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "SELECT COUNT(*) FROM tbl";


```

## External Links

- RamblingCookieMonster / PSSQLite: [https://github.com/RamblingCookieMonster/PSSQLite](https://github.com/RamblingCookieMonster/PSSQLite)
- SQLite V3 Data Types: [https://www.sqlite.org/datatype3.html](https://www.sqlite.org/datatype3.html)
- SQLite V3 Supported SQL Syntax: [https://www.sqlite.org/lang.html](https://www.sqlite.org/lang.html)
- SQLite V3 Pragma Statements: [http://www.sqlite.org/pragma.html](http://www.sqlite.org/pragma.html)

