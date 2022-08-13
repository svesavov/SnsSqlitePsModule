
###### If you like it, please consider buy me a beer :beer:
###### [![paypal](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=6NKR7XQH5E2P2&source=url)


# SnsSqlitePsModule PowerShell Module

This is a PowerShell module for working with [SQLite](https://www.sqlite.org) DataBases, based on Warren Frame's (RamblingCookieMonster) and his contributors project, named ["PSSQLite"](https://github.com/RamblingCookieMonster/PSSQLite).
The reason to make my own binary PSModule written on C# is related with performance. The binary CmdLets written on C# are much lighter and much faster compared with the ones written on PowerShell. This can be seen on the following screenshots:
![Performance Test Generate Data](/Media/NewHashTableData.jpg)
![Performance Test Insert Data Via Pipeline](/Media/InvokeSnsSqliteQueryInsertPipeline.jpg)
![Performance Test Generate Objects Collection](/Media/NewObjectsData.jpg)
![Performance Test Insert Objects Via Pipeline](/Media/InvokeSnsSqliteObjectInsertPipeline.jpg)
Although those CmdLets have 1000+ lines each, they are much faster than PowerShell ForEach loop.
Additional performance gain is the sending of the input via the Pipeline.
It is highly recommended, whenever substantial number of objects is processed, to avoid using "Verbose", "Debug" and whenever possible to send the input via the Pipeline.
The following screenshot contains the same test, but the input is provided to the CmdLet via parameters instead of using Pipeline:
![Performance Test ProgressBar](/Media/ProgressBar.jpg)
![Performance Test Insert Hashtable Collection Via Parameters](/Media/InvokeSnsSqliteQueryInsertParam.jpg)
![Performance Test Insert Object Collection Via Parameters](/Media/InvokeSnsSqliteObjectInsertParam.jpg)
When the CmdLets are called from a script run as a service not in interactive mode, they have no Progress Bar, this allows further gain in the performance.


## Functionality

Create an SQLite database and table:
  * ![Create a SQLite database and table](/Media/CreateDbAndTable.jpg)

Query an SQLite database. The results are filtered in PS to be shown the time required to select the previously inserted amount of data:
  * ![Query a SQLite database](/Media/InvokeSnsSqliteQuerySelect.jpg)

Bulk updates using Invoke-SnsSqliteQuery. Provides greater flexibility.
  * ![Pipeline Input Using ValueFromPipelineByPropertyName](/Media/InvokeSnsSqliteQueryBulkInsert.jpg)

Bulk updates using Invoke-SnsSqliteObjectInsert. Requires little to none knowledge of Structured Query Language.
  * ![Bulk Insert Of PowerShell Objects](/Media/InvokeSnsSqliteObjectInsertPipeline.jpg)

Create SQLite connections, use them for either backup or maintenance or subsequent queries:
  * ![Create 2 SQLite connections with new DataBase creation and Backup](/Media/ManualDbBackup.jpg)

Backup an SQLite database using Backup-SnsSqliteDataBase CmdLet.
  * ![Backup an SQLite database](/Media/BackupSnsSqliteDataBase.jpg)


Built-in performance measurement accessible in Verbose stream.
  * ![Truncate SQLite table](/Media/TruncateTable.jpg)

For additional information, please use the CmdLets built-in help.
```powershell
Get-Help Invoke-SnsSqliteQuery -Full;
Get-Help Invoke-SnsSqliteObjectInsert -Full;
Get-Help Backup-SnsSqliteDataBase -Full;
Get-Help New-SnsSqliteConnection -Full;
```


## Requirements

* .NET Framework 4.5.2
* PowerShell 4


## Instructions

Simply run
```powershell
Install-Module "SnsSqlitePsModule" -Scope "AllUsers";
```
OR
1. Download SnsSqlitePsModule.zip.
2. Don't forget to check the .ZIP file for viruses and etc.
3. File MD5 hash: `95BD11D8F34EA4276185C02E31B7EABE`
4. Unzip in one of the following folders depending of your preference:
* `C:\Users\UserName\Documents\WindowsPowerShell\Modules` - Replace "UserName" with the actual username, If you want the module to be available for specific user.
* `C:\Program Files\WindowsPowerShell\Modules` - If you want the module to be available for all users on the machine.
* Or any other location present in `$env:PSModulePath`
5. Run the following command replacing "PathWhereModuleIsInstalled" with the actual path where the module files were unzipped.
```powershell
Get-ChildItem -Path "PathWhereModuleIsInstalled" -Recurse | Unblock-File
```


## PowerShell Examples:

* Import the module, create a DataSource, and create a table inside the DataBase file.
```powershell


# Import the Module
Import-Module "SnsSqlitePsModule";
cd C:\TempDB\


# Create "temp.sqlite" DataBase in the working folder and "tbl" table inside the DB
Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -ReadOnly -Verbose `
	-Query "CREATE TABLE tbl ([ID] INTEGER, [Message] VARCHAR(500), [Severity] VARCHAR(10), [Date] DATETIME);";


# Verify the "tbl" table creation
$Output = Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "PRAGMA table_info(tbl);" -ReadOnly -Verbose;
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
		"Message" = "Fake Event Message";
		"Severity" = "Error";
		"Date" = [System.DateTime]::UtcNow.AddMinutes($_ - 100000);
	};
}
[System.DateTime]::now - $CmdStart;


# Insert the large amount of data in the DataBase
$CmdStart = [System.DateTime]::now;
$Params | Invoke-SnsSqliteQuery -DataBase "temp.sqlite" `
	-Query "INSERT INTO tbl (ID, Message, Severity, Date) VALUES (@ID, @Message, @Severity, @Date);";
[System.DateTime]::now - $CmdStart;


# Query the DataBase about the previously inserted data
# The output is limited in PowerShell, not in SELECT, to allow all the data to be retrieved from the DataBase
$Output = Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "SELECT ID, Message, Severity FROM tbl;" -Verbose;
$Output.Count
$Output | Select-Object -First 2;


```

* Insert data without using the Pipeline, and verify the insert.
```powershell


# Truncate the table
Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "DELETE FROM tbl;" -Verbose;


cls;
# Insert the large amount of data from previous example in the DataBase using parameters
$CmdStart = [System.DateTime]::now;
Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -SqlParameters $Params `
	-Query "INSERT INTO tbl (ID, Message, Severity, Date) VALUES (@ID, @Message, @Severity, @Date);";
[System.DateTime]::now - $CmdStart;


# Query the DataBase about the previously inserted data
$Output = Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "SELECT ID, Message, Severity FROM tbl;" -Verbose;
$Output.Count
$Output | Select-Object -First 2;


```

* Backup DataBase using "BackupDatabase" method of SQLiteConnection object
```powershell


# Create permanent SQLiteConnection object to the existing DataBase
# Create the Backup DataBase and connect to it
$ProdConn, $BkpConn = New-SnsSqliteConnection -DataBase "temp.sqlite", "Backup.sqlite" -ReadOnly;


#Backup the old DataBase into the newly created one
$CmdStart = [System.DateTime]::now;
$ProdConn.BackupDatabase($BkpConn, "main", "main", -1, $null, 0);
[System.DateTime]::now - $CmdStart;


# Verify whether the data is copied to the new database using the existing permanent connection
Invoke-SnsSqliteQuery -SQLiteConnection $BkpConn -Query "SELECT COUNT(*) FROM tbl;" -Verbose;


# Close Both Connections
$ProdConn.Close();
$ProdConn.Dispose();
$BkpConn.Close();
$BkpConn.Dispose();


```


* Backup DataBase using Backup-SnsSqliteDataBase CmdLet
```powershell


# Backup The Database
# Initially I've thought that there will be no need of such command
# However when backing up using the "BackupDatabase" method in scripts, they have to handle so many exceptions
# So I've created the command to not handle the same exceptions over and over again inside scripts
$Output = Backup-SnsSqliteDataBase -DataBase "temp.sqlite" -Destination "Backup2.sqlite" -Force -PassThru -Verbose;


# When PassThru is specified to Backup-SnsSqliteDataBase CmdLet
# It reverts a string with the absolute path of the Backup DataBase
# If backup has failed the string would be "Backup Filed"
# The verification is made via comparing the source and the backup DataBases file size in bytes
# If PassThru is not specified no verification is performed
$Output


# Verify whether the data is copied to the new database
Invoke-SnsSqliteQuery -DataBase "Backup2.sqlite" -Query "SELECT COUNT(*) FROM tbl;" -Verbose;


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
$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "Query" -Value "INSERT INTO tbl (ID, Message, Severity, Date) VALUES (@ID, @Message, @Severity, @Date);";
$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "SqlParameters" -Value `
(
	@{ "ID" = 1; "Message" = "Fake Event Message"; "Severity" = "Error"; "Date" = [System.DateTime]::UtcNow.AddMinutes(-1); },
	@{ "ID" = 2; "Message" = "Fake Event Message"; "Severity" = "Warning"; "Date" = [System.DateTime]::UtcNow; }
);
[System.Object[]]$arrInput += $objObject;


# Generate InputObject for the third query to verify the inserting the entry from the second query
[System.Object]$objObject = New-Object -TypeName "System.Object";
$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "Query" -Value "SELECT ID, Message, Severity FROM tbl;";
[System.Object[]]$arrInput += $objObject;


# Run All The 3 Queries
$arrInput | Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Verbose | ft;


```

* Example how to use "Invoke-SnsSqliteQuery" for bulk upload
```powershell


# Truncate the table
Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "DELETE FROM tbl;" -Verbose;


# Example how to convert objects to HashTables
# Keep in mind that the table where they have to be inserted should be already created
# And the table should have columns matching the objects properties
# SQLite have some limitations related with the column names
# Please refer to SQLite documentation about those

# Generate some test data
[System.DateTime]$cmdStart = [System.DateTime]::Now;
[System.Array]$arrInput = @();
0..100000 | ForEach `
{
	[System.Int32]$intI = $_;
	[System.Object]$objObject = New-Object -TypeName "System.Object";
	$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "ID" -Value $intI;
	$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "Message" -Value "Fake Event Message";
	$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "Severity" -Value "Warning";
	$objObject | Add-Member -Force -MemberType "NoteProperty" -Name "Date" -Value ([System.DateTime]::UtcNow.AddMinutes($intI - 100000));
	[System.Object[]]$arrInput += $objObject
}
[System.DateTime]::Now - $cmdStart;


# ToHashTbl() custom method works with PSCustomObject only
# From other hand any .NET object can be converted to PSCustomObject using "Select-Object *" command
# The ToHashTbl() method converts single object. Nest it in a loop to convert collection of objects.
[System.DateTime]$cmdStart = [System.DateTime]::Now;
[SnsSqlitePsModule.PsObjToHashTbl]::ToHashTbl($($arrInput | Select-Object *)) | `
	Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Verbose `
		-Query "INSERT INTO tbl (ID, Message, Severity, Date) VALUES (@ID, @Message, @Severity, @Date);";
[System.DateTime]::Now - $cmdStart;


# Query the DataBase about the previously inserted data
$Output = Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "SELECT ID, Message, Severity FROM tbl;" -Verbose;
$Output.Count
$Output | Select-Object -First 2;


```

* Example of how to insert a collection of objects without to convert them to hash tables in advance.
The object properties must match the destination table column names exactly. The values in the object properties must have type, either some of the struct types or string class. Values from any other classes might lead to unexpected results as for example instead of the actual value to be inserted the object type.
As it can be seen on the screenshots above, using of dedicated Cmdlet for SQL INSERT using ExecuteNonQuery() method, does not provide any performance benefits, than using SQLiteDataAdapter object, but dedicated CmdLet for SQL INSERT at least simplify the scripts which will use it by removing the additional overhead to convert the objects to hash tables.
```powershell


# Truncate the table
Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "DELETE FROM tbl;" -Verbose;


# Insert the test objects collection without using the pipeline
[System.DateTime]$cmdStart = [System.DateTime]::Now;
Invoke-SnsSqliteObjectInsert -DataBase "temp.sqlite" -Table "tbl" -InputObject $arrInput;
[System.DateTime]::Now - $cmdStart;


# Query the DataBase about the previously inserted data
$Output = Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "SELECT ID, Message, Severity FROM tbl;" -Verbose;
$Output.Count
$Output | Select-Object -First 2;

# Truncate the table to prepare it for the next test
Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "DELETE FROM [tbl];" -Verbose;


# Insert the same objects collection using Pipeline
# The performance boost is significant
[System.DateTime]$cmdStart = [System.DateTime]::Now;
$arrInput | Invoke-SnsSqliteObjectInsert -DataBase "temp.sqlite" -Table "tbl";
[System.DateTime]::Now - $cmdStart;


# Query the DataBase about the previously inserted data
$Output = Invoke-SnsSqliteQuery -DataBase "temp.sqlite" -Query "SELECT ID, Message, Severity FROM tbl;" -Verbose;
$Output.Count
$Output | Select-Object -First 2;


```


## External Links

- svesavov on GitHub: [https://github.com/svesavov](https://github.com/svesavov)
- svesavov on PowerShell Gallery: [https://www.powershellgallery.com/packages/SnsSqlitePsModule/](https://www.powershellgallery.com/packages/SnsSqlitePsModule/)
- Svetoslav Savov on LinkedIn: [https://www.linkedin.com/in/svetoslavsavov](https://www.linkedin.com/in/svetoslavsavov)
- RamblingCookieMonster / PSSQLite: [https://github.com/RamblingCookieMonster/PSSQLite](https://github.com/RamblingCookieMonster/PSSQLite)
- SQLite V3: [https://sqlite.org/index.html](https://sqlite.org/index.html)
- SQLite V3 Data Types: [https://www.sqlite.org/datatype3.html](https://www.sqlite.org/datatype3.html)
- SQLite V3 Supported SQL Syntax: [https://www.sqlite.org/lang.html](https://www.sqlite.org/lang.html)
- SQLite V3 Pragma Statements: [http://www.sqlite.org/pragma.html](http://www.sqlite.org/pragma.html)
- SQLite Tutorials: [https://www.sqlitetutorial.net/](https://www.sqlitetutorial.net/)
- SQLite Studio: [https://github.com/pawelsalawa/sqlitestudio/releases](https://github.com/pawelsalawa/sqlitestudio/releases)

