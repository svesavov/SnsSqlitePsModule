SnsSqlitePsModule PowerShell Module
=============

This is a PowerShell module for working with [SQLite](https://www.sqlite.org) DataBases, based on Warren Frame's (RamblingCookieMonster) and his contributors project, named ["PSSQLite"](https://github.com/RamblingCookieMonster/PSSQLite).
The reason to make my own Binary PSModule written on C# is mainly related with performance as it can be seen on the screenshot:
![Performance Test Generate Data](/Media/GenerateData.JPG)
![Performance Test Insert Data Via Pipeline](/Media/InsertViaPipeline.JPG)
As it can be seen binary CmdLet is much faster than PowerShell ForEach loop, having in mind that this CmdLet have 1000+ lines.
Sending data via the Pipeline boosts the performance additionally.
ProgressBar can't be used when objects are coming from Pipeline, because the CmdLet can’t estimate their number.
The PowerShell "extras" are the factors that decrease the performance by a lot, such like ProgressBar, Verbose Stream, Debug Stream and every information displayed on the host window.
It is highly recommended, whenever large number of objects is processed, to avoid using Verbose, Debug and possibly send the data via the Pipeline.
The following screenshot contains the same test, but the data is provided to the CmdLet via parameters instead of Pipeline:
![Performance Test ProgressBar](/Media/ProgressBar.JPG)
![Performance Test Insert Data Via Parameters](/Media/InsertDataViaParam.JPG)


## Functionality

Create a SQLite database and table:
  * ![Create a SQLite database and table](/Media/CreateDbAndTable.JPG)

Query a SQLite database. The results are filtered in PS to be shown the time required to select the previously inserted amount of data:
  * ![Query a SQLite database](/Media/QueryData.JPG)

Create a SQLite connection, use it for maintenance or subsequent queries to avoid verifications repetition:
  * ![Create 2 SQLite connections with new DataBase creation and Backup](/Media/DbBackup.JPG)

Built-in performance measurement accessible in Verbose stream.
  * ![Truncate SQLite table](/Media/TruncateTable.JPG)

Re-designed Pipeline input - using the pipeline can be performed bulk updates. There is no need of additional command for bulk upload.
  * ![Pipeline Input Using ValueFromPipelineByPropertyName](/Media/InsertViaPropertyNamePipeline.JPG)

For additional information, please use the CmdLets built-in help.
```powershell
Get-Help New-SnsSqliteConnection -Full; 
Get-Help Invoke-SnsSqliteQuery -Full;
```


## Requirements

.NET Framework 4.5
PowerShell 4


## Instructions

1. Download SnsSqlitePsModule.zip
2. Don't forget to check the .ZIP file for viruses and etc.
3. File MD5 hash: 9BD48464E3AA6D8DDD7C85F6F55D727E
4. Unzip in one of the following folders depending of the preference:
* C:\Users\UserName\Documents\WindowsPowerShell\Modules - Replace "UserName" with the actual username. Use this location if you want the module available for specific user
* C:\Program Files\WindowsPowerShell\Modules - Use this location if you want the module available for all users on the machine.
* Or any other location present in $env:PSModulePath
5. Run the following command replacing "PathWhereModuleIsInstalled" with the actual path where the module files were unzipped.
```powershell
Get-ChildItem -Path "PathWhereModuleIsInstalled" -Recurse | Unblock-File
```
6. PowerShell Examples:

* Import the module, create a DataSource, and create a table inside the DataBase file.
```powershell


# Import the Module
Import-Module "SnsSqlitePsModule";


# Create "temp.sqlite" DataBase in the working folder and "tbl" table inside the DB
Invoke-SnsSqliteQuery `
	-DataBase "temp.sqlite" `
	-Query "CREATE TABLE tbl (ID INTEGER, Event VARCHAR(20), Date DATETIME);" `
	-Password "Pass" `
	-ReadOnly `
	-Verbose;


# Verify the "tbl" table creation
$Output = Invoke-SnsSqliteQuery `
	-DataBase "temp.sqlite" `
	-Query "PRAGMA table_info(tbl);" `
	-Password "Pass" `
	-ReadOnly `
	-Verbose;
$Output | ft


```

* Generate large amount of data, provide it to "Invoke-SnsSqliteQuery" via the Pipeline, and verify the insert.
```powershell


# Generate large amount of data with PowerShell and measure the time.
$CmdStart = [System.DateTime]::now;
[System.Collections.Hashtable[]]$Params = @();
1..100000 | ForEach `
{
	$Params += `
	@{
		"ID" = "$($_)";
		"Event" = "Error";
		"Date" = [System.DateTime]::UtcNow.AddDays(0 - $_).ToString([SnsSqlitePsModule.TimeFormat]::Utc);
	};
}
[System.DateTime]::now - $CmdStart;


# Insert the large amount of data in the DataBase
$CmdStart = [System.DateTime]::now;
$Params | Invoke-SnsSqliteQuery `
	-DataBase "temp.sqlite" `
	-Query "INSERT INTO tbl (ID, Event, Date) VALUES (@ID, @Event, @Date);" `
	-Password "Pass";
[System.DateTime]::now - $CmdStart;


# Query the DataBase about the previously inserted data
# The output is limited in PowerShell, not in SELECT, to allow all the data to be retrieved from the DataBase
# Internal measuring accessible via "Verbose" stream is not used because it will generate thousands of screens
# and will be unable to make the screenshot
$CmdStart = [System.DateTime]::now;
$Output = Invoke-SnsSqliteQuery `
	-DataBase "temp.sqlite" `
	-Query "SELECT * FROM tbl;" `
	-Password "Pass";
[System.DateTime]::now - $CmdStart;
$Output | Select-Object -First 10;


```

* Insert data without using the Pipeline, and verify the insert.
```powershell


# Truncate the table
Invoke-SnsSqliteQuery `
	-DataBase "temp.sqlite" `
	-Query "DELETE FROM tbl;" `
	-Password "Pass" `
	-Verbose;


# Insert the large amount of data from previous example in the DataBase using parameters
$CmdStart = [System.DateTime]::now;
Invoke-SnsSqliteQuery `
	-DataBase "temp.sqlite" `
	-Query "INSERT INTO tbl (ID, Event, Date) VALUES (@ID, @Event, @Date);" `
	-SqlParameters $Params `
	-Password "Pass";
[System.DateTime]::now - $CmdStart;


# Query the DataBase about the previously inserted data
$CmdStart = [System.DateTime]::now;
$Output = Invoke-SnsSqliteQuery `
	-DataBase "temp.sqlite" `
	-Query "SELECT * FROM tbl;" `
	-Password "Pass";
[System.DateTime]::now - $CmdStart;
$Output | Select-Object -First 10;


```

* Backup DataBase
```powershell


# Create permanent SQLiteConnection object to the existing DataBase
# Create the Backup DataBase protected with the same Password and connect to it
$ProdConn, $BkpConn = New-SnsSqliteConnection `
	-DataBase "temp.sqlite", "Backup.sqlite" `
	-Password "Pass" `
	-ReadOnly;


#Backup the old DataBase into the newly created one
$CmdStart = [System.DateTime]::now;
$ProdConn.BackupDatabase($BkpConn, "main", "main", -1, $null, 0);
[System.DateTime]::now - $CmdStart;


# Verify whether the data is copied to the new database using the existing permanent connection
Invoke-SnsSqliteQuery `
	-SQLiteConnection $BkpConn `
	-Query "SELECT COUNT(*) FROM tbl;" `
	-Verbose;


```

* Testing the ValueFromPipelineByPropertyName Pipeline input
```powershell


# Create an empty array to hold the InputObjects which will be send to "Invoke-SnsSqliteQuery"
[System.Object[]]$arrInput = @();


# Generate InputObject for the first query to delete all the entries in "tbl" Table
# The object must have property names matching the CmdLet parameters or their aliases.
[System.Object]$objObject = New-Object -TypeName "System.Object";
$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "Query" -Value "DELETE FROM tbl;";
[System.Object[]]$arrInput += $objObject;


# Generate InputObject for the second query to insert some data in "tbl" Table
# The object must have property names matching the CmdLet parameters or their aliases.
[System.Object]$objObject = New-Object -TypeName "System.Object";
$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "Query" -Value "INSERT INTO tbl (ID, Event, Date) VALUES (@ID, @Event, @Date);";
$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "SqlParameters" -Value `
(
	@{ "ID" = 1; "Event" = "Error"; "Date" = "2020-04-20 10:00:00.000Z"; },
	@{ "ID" = 2; "Event" = "Warning"; "Date" = "2020-04-20 11:00:00.000Z"; }
);
[System.Object[]]$arrInput += $objObject;


# Generate InputObject for the third query to verify the inserting the entry from the second query
[System.Object]$objObject = New-Object -TypeName "System.Object";
$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "Query" -Value "SELECT * FROM tbl;";
[System.Object[]]$arrInput += $objObject;


# Run All The 3 Queries
$arrInput | Invoke-SnsSqliteQuery `
	-DataBase "temp.sqlite" `
	-Password "Pass" `
	-Verbose | ft;


```

* Example how to use "Invoke-SnsSqliteQuery" for bulk upload
```powershell


# Truncate the table
Invoke-SnsSqliteQuery `
	-DataBase "temp.sqlite" `
	-Query "DELETE FROM tbl;" `
	-Password "Pass" `
	-Verbose;


# Example how to convert objects to HashTables
# Keep in mind that the table where they have to be inserted should be already created
# And the table should have columns like the objects properties
# SQLite have some limitations related with the column names
# Please refer to SQLite documentation about those
[System.Object]$objObject = New-Object -TypeName "System.Object";
$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "ID" -Value 1;
$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "Event" -Value "Warning";
$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "Date" -Value ([System.DateTime]"2020-04-20 10:00:00");


# ToHashTbl() custom method works with PSCustomObject only
# From other hand any .NET object can be converted to PSCustomObject using "Select-Object *" command
# The ToHashTbl() method converts single object. Nest it in a loop to convert collection of objects.
[SnsSqlitePsModule.PsObjectToHashTbl]::ToHashTbl($($objObject | Select-Object *)) | `
	Invoke-SnsSqliteQuery `
		-DataBase "temp.sqlite" `
		-Query "INSERT INTO tbl (ID, Event, Date) VALUES (@ID, @Event, @Date);" `
		-Password "Pass" `
		-Verbose;

# Verify the data inserting
Invoke-SnsSqliteQuery `
	-DataBase "temp.sqlite" `
	-Query "SELECT * FROM tbl;" `
	-Password "Pass" | ft;


```
If I manage to find time in the future, might create a CmdLet for bulk insert using the logic in the example. Moreover "Invoke-SnsSqliteQuery" uses SQLiteDataAdapter objects which leaves some room to squeeze more performance on insert queries using ExecuteNonQuery() Method.


## External Links

- svesavov on GitHub: [https://github.com/svesavov](https://github.com/svesavov)
- Svetoslav Savov on LinkedIn [https://www.linkedin.com/in/svetoslavsavov](https://www.linkedin.com/in/svetoslavsavov)
- RamblingCookieMonster / PSSQLite: [https://github.com/RamblingCookieMonster/PSSQLite](https://github.com/RamblingCookieMonster/PSSQLite)
- SQLite V3 Data Types: [https://www.sqlite.org/datatype3.html](https://www.sqlite.org/datatype3.html)
- SQLite V3 Supported SQL Syntax: [https://www.sqlite.org/lang.html](https://www.sqlite.org/lang.html)
- SQLite V3 Pragma Statements: [http://www.sqlite.org/pragma.html](http://www.sqlite.org/pragma.html)

[![paypal](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](paypal.me/svesavov)