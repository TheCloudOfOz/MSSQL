# Lab 2

> All PowerShell and Command-line consoles should be opened with elevated Administrator privileges. Unless otherwise stated, all exercises will be completed from the **MIA-SQL** server with the **AdventureWorks\Student** credentials. All user account passwords are the same (**Pa55w.rd**).

## Objectives

Manage access to SQL Server instances & the performance and availability of databases. We will also work with Integration Services and Analysis Services components. To benefit fully from these exercises, take the time to examine the PowerShell scripts used. They will prove useful if you decide to do the optional lab.

# Exercise 1

We will create Windows & SQL Login accounts using SQL Server Management Objects (SMO).

> 90 minitues

## Task 1 : Create Database

1. Login to **MIA-SQL** with the **AdventureWorks\Student** account

2. Start **Azure Data Studio** by right clicking and selecting **Run As Administrator**

> We will be using SMO components to create and assign permissions to login and user accounts. :

3. Load the SMO assembly.

> PowerShell 5.1 and newer version use this instead

```powershell
using namespace System.Reflection
using namespace Microsoft.SqlServer.Management.Smo

[Assembly]::LoadWithPartialname('Microsoft.SQLServer.SMO')
```

> Older Version PowerShell

```powershell
[System.Reflection.Assembly]::LoadWithPartialname('Microsoft.SQLServer.SMO')
```

4. Create a variable named **$Instance** to refer to the name of the **MIA-SQL** server:

```powershell
$Instance = "MIA-SQL"
```

5. Create a variable to be used for the **AdventureWorks\User1** user account:

```powershell
$User1 = "AdventureWorks\User1"
```

6. Use SMO components to connect to the **MIA-SQL** instance using the $Server variable: 

> Older Version PowerShell

```powershell
$Server = new-object Microsoft.SqlServer.Management.Smo.Server $Instance
```

> New Version PowerShell, Use This Style

```powershell
$Server = [Server]::new($Instance)
```

7. Use the $Server variable to create a new database:

```powershell
$Server.ConnectionContext.ExecuteWithResults("CREATE DATABASE AWDB")
```

> Completed Create Database Script

```powershell
$Instance = "MIA-SQL"   
$User1 = "AdventureWorks\User1"
$Server = [Server]::new($Instance)
$Server.ConnectionContext.ExecuteWithResults("CREATE DATABASE AWDB")
```

## Task 2 : Create Windows Login

1. Open SQL Server Management Studio with **Run as different user**

2. Type **AdventureWorks\User1** and Press Tab.

3. Type the user pasword and press Enter: **Pa55w.rd**

> Important : If you dont have **User1** test account in your Active Directory use this script to create

```powershell
Enter-PSSession "MIA-DC"
Import-Module -Name "ActiveDirectory"
$User = @{
    Name              = "User1"
    GivenName         = "Test"
    Surname           = "User"
    SamAccountName    = "User1"
    UserPrincipalName = "user1@adventureworks.msft"
    Enabled           = $true
}
New-ADUser @User -AccountPassword(ConvertTo-SecureString -AsPlainText 'Pa55w.rd' -Force)
Exit-PSSession
```

4. Verify that **AdventureWorks\User1** **cannot** connect to MIA-SQL

5. Define a Login object to be used for the AdventureWorks\User1 account: 

> Older Version PowerShell

```powershell
$Login = New-Object -TypeName Microsoft.SQLServer.Management.SMO.Login -ArgumentList $Instance,$User1
```

> New Version PowerShell, Use This Style

```powershell
$Login = [Login]::new($Instance,$User1)
```

6. Configure the $Login variable as a Windows account: 

```powershell
$Login.LoginType = "WindowsUser"
```

7. Create the login account: 

```powershell
$Login.Create()
```

8. Add the dbcreator role to the login account: 

```powershell
$Login.AddToRole("dbcreator")
```

> Completed Windows Login Script

```powershell
$Login = [Login]::new($Instance,$User1)
$Login.LoginType = "WindowsUser"
$Login.Create()
$Login.AddToRole("dbcreator")
```

## Task 3 : Create SQL Login with Database Role Membership

1. We will now create a SQL Server login account. First, define a variable for the login account:

```powershell
$SQLLogin1 = "SQLLogin1"
```

2. Create a variable to hold the password. 

> Note: To create the password without the need of the escape character (`), encapsulate the information with single quotes instead of double quotes: 

```powershell
$Password = 'Pa55w.rd'
```

3. Define a login object for the new SQL Server login account: 

> Older Version PowerShell

```powershell
$LoginSQL = New-Object -TypeName Microsoft.SQLServer.Management.SMO.Login -ArgumentList $Instance,$SQLLogin1
```

> New Version PowerShell, Use This Style

```powershell
$LoginSQL = [Login]::new($Instance,$SQLLogin1)
```

4. Configure the new login object as a SQL Server login: 

```powershell
$LoginSQL.LoginType = "SQLLogin"
```

5. Create the new login and assign the password: 

```powershell
$LoginSQL.Create($Password)
```

6. We will now assign the SQL Login access to the AWDB database. First, create a variable for the AWDB database: 

```powershell
$AWDB = $Server.Databases["AWDB"]
```

7. Define a new user object in AWDB: 

> Older Version PowerShell

```powershell
$DBUser = New-Object -TypeName "Microsoft.SqlServer.Management.Smo.User" -ArgumentList $AWDB, $SQLLogin1
```

> New Version PowerShell, Use This Style

```powershell
$DBUser = [User]::new($AWDB, $SQLLogin1)
```

8. Link the user account to the login account: 

```powershell
$DBUser.Login = $SQLLogin1
```

9. Assign the dbo schema as the default for the account: 

```powershell
$DBUser.DefaultSchema = "dbo"
```

10. Create the user account: 

```powershell
$DBUser.Create()
```

11. Create a variable for the db_ddladmin role in the AWDB database:

```powershell
$db_ddladmin = $AWDB.roles["db_ddladmin"]
```

12. Add the SQLLogin1 account to the db_ddladmin role: 

```powershell
$db_ddladmin.AddMember($SQLLogin1)
```

13. Create a variable for the db_datawriter role in the AWDB database: 

```powershell
$db_datawriter = $AWDB.roles["db_datawriter"]
```

14. Add the SQLLogin account to the db_datawriter role in the AWDB database:

```powershell
$db_datawriter.AddMember($SQLLogin1)
```

> Completed SQL Login Script

```powershell
$SQLLogin1 = "SQLLogin1"
$Password = 'Pa55w.rd'

$LoginSQL = [Login]::new($Instance,$SQLLogin1)
$LoginSQL.LoginType = "SQLLogin"
$LoginSQL.Create($Password)

$AWDB = $Server.Databases["AWDB"]
$DBUser = [User]::new($AWDB, $SQLLogin1)
$DBUser.Login = $SQLLogin1
$DBUser.DefaultSchema = "dbo"
$DBUser.Create()

$db_ddladmin = $AWDB.roles["db_ddladmin"]
$db_ddladmin.AddMember($SQLLogin1)
$db_datawriter = $AWDB.roles["db_datawriter"]
$db_datawriter.AddMember($SQLLogin1)
```

## Task 4 : Create Objects for Test

1. Create a new table to test **SQLLogin1**’s **db_ddladmin** credentials: 

```sql
Create Table Contacts (ID INT, FirstName NVarChar(50), LastName NVarChar(50))
```

2. Define a variable for the **Invoke-SQLCMD** connection

```powershell
$Connection = @{
    ServerInstance = "MIA-SQL"
    UserName = "SQLLogin1"
    Password  = 'Pa55w.rd'
    Database = "AWDB"
}
```

3. Execute Create Table SQL Query by using **Invoke-SQLCMD**

```powershell
Invoke-SQLCMD @Connection -Query 'Create Table Contacts (ID INT, FirstName NVarChar(50), LastName NVarChar(50))'
```

4. Insert a row of information into the newly created table: 

```sql
Insert Contacts Values(1,'John', 'Brown')
```

5. Execute Insert SQL Query by using **Invoke-SQLCMD**

```powershell
Invoke-SQLCMD @Connection -Query "Insert Contacts Values(1,'John', 'Brown')"
```

6. Try to query the new Contacts table: 

```sql
Select * From Contacts
```

7. Execute Select SQL Query by using **Invoke-SQLCMD**

```powershell
Invoke-SQLCMD @Connection -Query "SELECT * FROM Contacts"
```

8. The query was **unsuccessful** because **SQLLogin1** was not given permissions to query data in this database (e.g. **db_datareader**).

> Invoke-SQLCMD : The SELECT permission was denied on the object 'Contacts', database 'AWDB', schema 'dbo'

> Completed Test Script
```powershell
Import-Module -Name 'SQLServer'
$Connection = @{
    ServerInstance = "MIA-SQL"
    UserName = "SQLLogin1"
    Password  = 'Pa55w.rd'
    Database = "AWDB"
}
Invoke-SQLCMD @Connection -Query 'Create Table Contacts (ID INT, FirstName NVarChar(50), LastName NVarChar(50))'
Invoke-SQLCMD @Connection -Query "Insert Contacts Values(1,'John', 'Brown')"
Invoke-SQLCMD @Connection -Query "SELECT * FROM Contacts"
```