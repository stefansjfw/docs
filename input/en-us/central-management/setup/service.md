---
Order: 20
xref: ccm-service
Title: Service
Description: Information on how to setup the CCM Service
RedirectFrom: docs/central-management-setup-service
---

This is the service that the agents (chocolatey-agent) communicates with. You could install one or more of these depending on the size of your environment (not multiple on one machine though). The FQDN and certificate used determine what the URL will be for the agents to check into Central Management.

> :choco-warning: **WARNING**
>
> Unless otherwise noted, please follow these steps in **exact** order. These steps build on each other and need to be completed in order.

> :choco-warning: **WARNING**
>
> In order to run the CCM Service a user with Administrator access is required.  By default, a new user named `ChocolateyLocalAdmin` will be created and configured to run the CCM Service.  In addition, `Logon as Service` and `Logon as Batch` privileges will be asserted for this user.  If attempting to run the CCM Service as a different user, these permissions will be required.

## Step 1: Complete Prerequisites

> :choco-warning: **WARNING**
>
> The [database](xref:ccm-database) must be setup and available, along with [logins and access](xref:ccm-database#step-2-set-up-sql-server-logins-and-access).

* Windows Server 2016
* PowerShell 5.1

## Step 2: Install Central Management Service Package

By default the service will install as a local administrative user `ChocolateyLocalAdmin` (and manage the password as well). However you can specify your own user with package parameters (such as using a domain account). You will need to specify credentials to the database as we'll see in scenarios below.

> :choco-warning: **WARNING**
>
> Timezones are super important here and time synchronization is really important when generating SSL Certificates. You want to make sure you have this correct and good. Otherwise there is a potential edge case you could generate an SSL Certificate that is not yet valid. As the service package could generate an SSL certificate if you don't pass an existing thumbprint, its best to ensure that time synchronization is not an issue with the machine you are installing this on.

### FQDN Usage

When installing the CCM Service, the default is to use the Fully Qualified Domain Name (FQDN) of the machine that it is being installed on.  As a result, there is an expectation that the certificate (either the self signed certificate that is created during installation, or the existing certificate which is configured with the [CertificateThumbprint](#package-parameters-1) parameter) that is used to secure the transport layer of this service, also uses the same FQDN.

```powershell
# Find FDQN for current machine
$hostName = [System.Net.Dns]::GetHostName()
$domainName = [System.Net.NetworkInformation.IPGlobalProperties]::GetIPGlobalProperties().DomainName

if(-Not $hostName.endswith($domainName)) {
  $hostName += "." + $domainName
}

choco config set --name="'centralManagementServiceUrl'" --value="'https://$($hostname):24020/ChocolateyManagementService'"
```

If this is not the case, it will be necessary to provide the information to the package about the actual name for the machine that is being used.  When using a self signed certificate, this can be specified using the `CertificateDnsName`, and when using an existing certificate, no additional parameters are required.  In both cases, it will be necessary to also set the `centralManagementServiceUrl` [configuration parameter](#centralmanagementserviceurl).  This can be done using the following command:

```powershell
choco config set --name="'centralManagementServiceUrl'" --value="'https://<accessible_url>:24020/ChocolateyManagementService'"
```

### Package Parameters

Note items with "`:`" mean a value should be provided, items without are simply switches.

* `/Username:` - Username to run the service under. Defaults to `ChocolateyLocalAdmin`. This should be a local Administrator - this is typically ensured during installation. `Logon as Service` and `Logon as Batch` privileges are also ensured.
* `/Password:` - Password for the user. Default is the [Chocolatey Managed Password](#chocolatey-managed-password).
* `/EnterPassword` - Receive the password at runtime as a secure string. Requires input at runtime whe installing/upgrading the package.
* `/NoRestartService` - Do not shut down and restart the service. You will need to restart later to take advantage of new service information.
* `/DoNotReinstallService` - Do not re-install the service.
* `/PortNumber:` - The port the Chocolatey Management Service will listen on. This will automatically create a rule to open the firewall on the port specified. Defaults to `24020`.
* `/CertificateDnsName:` - The DNS name of the self-signed certificate that is generated if no existing certificate thumbprint is provided using the `/CertificateThumbprint` parameter (below). Defaults to `<LOCAL COMPUTER FQDN NAME>`.
* `/CertificateThumbprint:` - Provide the thumbprint of an existing certificate (already installed in `LocalMachine\TrustedPeople` certificate store) to use for secure communication with clients. Defaults to a new self-signed SSL certificate on first installation / reuses existing on upgrades.
* `/ConnectionString:` - The SQL Server database connection string to be used to connect to the Chocolatey Central Management database. Defaults to default or explicit values for `/SqlServiceInstance` and `/Database`, along with Integrated Security (`Server=<LOCAL COMPUTER FQDN NAME>; Database=ChocolateyManagement; Trusted_Connection=True;`). The account should have `db_datareader`/`db_datawriter` access to the database ([data reader / data writer](https://docs.microsoft.com/en-us/sql/relational-databases/security/authentication-access/database-level-roles#fixed-database-roles)).
* `/SqlServerInstance:` - Instance name of the SQL Server database to connect to. Alternative to passing full connection string with `/ConnectionString`. Uses `/Database` (below) to build a connection string. Defaults to `<LOCAL COMPUTER FQDN NAME>`.
* `/Database:` - Name of the SQL Server database to use. Alternative to passing full connection string with `/ConnectionString`. Uses `/SqlServerInstance` (above) to build a connection string. Defaults to `ChocolateyManagement`.

> :choco-info: **NOTE**
>
> Items suffixed with "`:`" mean a value should be provided, items without are simply switches.

### Service Settings

* Service Name:                         **chocolatey-management-service**
* Service Displayname:                  **Chocolatey Management Service**
* Description:                          **Chocolatey Management Service is a service for Chocolatey**
* Service Startup:                      **Automatic**
* Service Username:                     **ChocolateyLocalAdmin** or '`/Username:`'

### Chocolatey Central Management Service Configuration

For most installations, the default values for the Chocolatey Central Management Service configuration should be all that is required.  However, if you need to modify this configuration, then full details of what can be controlled is detailed on the [Chocolatey Central Management Service configuration page](xref:ccm-usage-service-configuration#chocolatey-configuration-file).

### Chocolatey Configuration

There are a number of different Chocolatey Configuration values that can be set for the Chocolatey Central Management Service.  The main setting that _has_ to be set is detailed below, and you can find a complete list on the [Chocolatey Central Management Service configuration page](xref:ccm-usage-service-configuration#chocolatey-configuration-file).

* `centralManagementServiceUrl` = **' '** (empty) - The URL that should be used to communicate with Chocolatey Central Management. It should look something like `https://servicemachineFQDN:24020/ChocolateyManagementService`. See [FQDN usage](xref:ccm#fqdn-usage). Defaults to '' (empty). NOTE: Chocolatey Agent and CCM Service share this value on a machine that contains both. If blank, the CCM Service will construct a URL based on defaults of the machine, but is required to be set for Agents.

> :choco-warning: **WARNING**
>
> The Chocolatey Agent installed on the same machine that has the CCM Service installed will share the `centralManagementServiceUrl` setting, so that agent can only report into that CCM Service.

### Chocolatey Managed Password

When Chocolatey manages the password for a local Administrator, it creates a very complex password:

* It is 32 characters long.
* It uses uppercase, lowercase, numbers, and symbols to meet very stringent complexity requirements.
* The password is different for every machine.
* Due to the way that it is generated, it is completely unguessable.
* No one at Chocolatey Software could even tell you what the password is for a particular machine without local access.

### Chocolatey Central Management Service Windows Account Considerations

* Windows Account (required, defaults to `ChocolateyLocalAdmin`)
  * The Chocolatey Central Management Service requires **an** administrative account, whether that is a domain account or a local account - it just needs to be a local admin (a member of the Administrators group).
  * The Chocolatey Central Management Service doesn't specifically require the `ChocolateyLocalAdmin` account, any Windows account can be used. The `ChocolateyLocalAdmin` is used as the default if one is not specified.
  * Upon use of an account during installation, it will make that account a member of the Administrators account.
  * The account used will also be granted LogonAsService and LogonAsBatch privileges.
* Managed Password (optional, default)
  * When the `ChocolateyLocalAdmin` account is used, it generates a managed password that is different on every machine, 32 characters long, meets complexity requirements, and basically very strong.
  * To determine the managed password, it would take access to the box and someone from Chocolatey Software who has access to the algorithm used to generate the password (more information in the FAQs below).
* Rotating/Updating Passwords
  * If a different account with a rotating password is used, the service will need to be updated with the new credentials and restarted soon after changing that password.
  * The managed password is not currently updated/rotated, but it is something we are looking at how best to implement.

### Scenarios

#### SQL Server Windows Authentication

##### Use Active Directory Domain Account

Scenario 1: Active Directory - you have set up the [database](xref:ccm-database) to use Windows Authentication (or Mixed Mode Authentication).

```powershell
choco install chocolatey-management-service -y --package-parameters="'/ConnectionString:Server=<RemoteSqlHost>;Database=ChocolateyManagement;Trusted_Connection=True; /Username:<DomainAccount>'" --package-parameters-sensitive="'/Password:<domain account password>'"
```

> :choco-warning: **WARNING**
>
> Please ensure the user `<DomainAccount>` has been given `db_datareader` and `db_datawriter` access to the database. See [logins and access](xref:ccm-database#step-2-set-up-sql-server-logins-and-access).

> :choco-info: **NOTE**
>
> Note the connection string doesn't include credentials. That's because Windows Authentication for SQL Server uses the context of what is running it and why the service itself needs the right user/password.

##### Use Local Windows Account to Local SQL Server

Scenario 2: Monolithic - you have set up the [database](xref:ccm-database) to use Windows Authentication (or Mixed Mode Authentication). You wish to use a local Windows account to connect to the local database.

* Specify User:

```powershell
choco install chocolatey-management-service -y --package-parameters="'/ConnectionString:Server=<Localhost\SQLEXPRESS>;Database=ChocolateyManagement;Trusted_Connection=True; /Username:<LocalWindowsAccount>'" --package-parameters-sensitive="'/Password:<Local account password>'"
```

> :choco-warning: **WARNING**
>
> Please ensure the user `<LocalWindowsAccount>` has been given `db_datareader` and `db_datawriter` access to the database. See [logins and access](xref:ccm-database#step-2-set-up-sql-server-logins-and-access).

* ChocolateyLocalAdmin User:

```powershell
choco install chocolatey-management-service -y --package-parameters="'/ConnectionString:Server=<Localhost\SQLEXPRESS>;Database=ChocolateyManagement;Trusted_Connection=True;'"
```

> :choco-warning: **WARNING**
>
> Please ensure the user `ChocolateyLocalAdmin` has been given `db_datareader` and `db_datawriter` access to the database. See [logins and access](xref:ccm-database#step-2-set-up-sql-server-logins-and-access).

> :choco-info: **NOTE**
>
> The connection string doesn't include credentials. That's because Windows Authentication for SQL Server uses the context of what is running it and why the service itself needs the right user/password.

##### Use Windows Account to Attach SQL Server

You are using AttachDBFile or User Instance in your Connection String. This is effectively asking to attach a database file to the User's Data directory.

```powershell
choco install chocolatey-management-service -y --package-parameters="'/ConnectionString:Data Source=.\SQLEXPRESS;Integrated Security=SSPI;AttachDBFilename=|DataDirectory|SomeDbFile.mdf;User Instance=true;'"
```

> :choco-warning: **WARNING**
>
> STOP right here. This is an unsupported scenario.
>
> While it may work, it's a really bad idea. Please look at one of the other options.
> It can also result in "Failed to generate a user instance of SQL Server due to failure in retrieving the user's local application data path."

##### Use Local Windows Account to Remote SQL Server

Scenario 3: you have set up the [database](xref:ccm-database) to use Windows Authentication (or Mixed Mode Authentication). You wish to use a local Windows account to connect to a remote database (on another computer).

> :choco-warning: **WARNING**
>
> STOP right here.
> This is an invalid scenario and will not work. Please look at one of the other options. If you don't have LDAP, you will want to look at [SQL Server Account Authentication](#sql-server-account-authentication) below.

It's worth noting here that the local Windows user `ChocolateyLocalAdmin` on two boxes is NOT the same account, so there is no way for Windows to recognize the account from a different box.

#### SQL Server Account Authentication

##### Use SQL Server Authentication Locally

Scenario 4: Monolithic - you are installing the management service on the same machine as a SQL Server Express instance. You likely have a smaller environment where you have up to 1,000 machines. You have set up the [database](xref:ccm-database) to use Mixed Mode Authentication.

```powershell
choco install chocolatey-management-service -y --package-parameters-sensitive="'/ConnectionString:Server=Localhost;Database=ChocolateyManagement;User ID=ChocoUser;Password=Ch0c0R0cks;'"
```

> :choco-warning: **WARNING**
>
> Please ensure the login has been given `db_datareader` and `db_datawriter` access to the database. See [logins and access](xref:ccm-database#step-2-set-up-sql-server-logins-and-access).

* SQL Server Express:

```powershell
choco install chocolatey-management-service -y --package-parameters-sensitive="'/ConnectionString:Server=Localhost\SQLEXPRESS;Database=ChocolateyManagement;User ID=ChocoUser;Password=Ch0c0R0cks;'"
```

> :choco-warning: **WARNING**
>
> Please ensure the login has been given `db_datareader` and `db_datawriter` access to the database. See [logins and access](xref:ccm-database#step-2-set-up-sql-server-logins-and-access).

##### Use SQL Server Account to Remote SQL Server

Scenario 5: Split - you are installing the management service(s) on a server, and targeting an existing SQL Server instance in your organization. You have set up the [database](xref:ccm-database) to use Mixed Mode Authentication.

```powershell
choco install chocolatey-management-service -y --package-parameters-sensitive="'/ConnectionString:Server=<RemoteSqlHost>;Database=ChocolateyManagement;User ID=ChocoUser;Password=Ch0c0R0cks;'"
```

> :choco-warning: **WARNING**
>
> Please ensure the login has been given `db_datareader` and `db_datawriter` access to the database. See [logins and access](xref:ccm-database#step-2-set-up-sql-server-logins-and-access).

## Step 3: Verify Installation

The `chocolatey-management-service` is responsible for making a number of changes to your system.  A successful installation of this package can be verified by:

* Open the services snap-in (services.msc) and check for the presence of the `Chocolatey Management Service` which should be in the "Started" state.
* The installation folder for `chocolatey-management-service` is at `$env:ChocolateyInstall\lib\chocolatey-management-service\tools\service`.
* Open the Service log file located at `$env:ChocolateyInstall\logs\ccm-service.log` and verify that there are no recently reported errors. If you are on a version of CCM prior to 0.2.0, the log will be located at `$env:ChocolateyInstall\lib\chocolatey-management-service\tools\service\logs\chocolatey.service.host.log`.

## FAQ

### Why is the CCM Service not marked as 'Stopped' when there is an error during its execution?

By default, all exceptions that occur within the CCM Service are logged to the applications log file (ccm-service.log), however, these exceptions do not result in actually physically stopping the service.
In other words, if an exception is thrown within the application, if you checked the CCM Service in the Windows Services snap-in, you would still see that the service is running.
Due to the nature of how exceptions occur within the CCM Service, this is normally perfectly fine, since operations are attempted again at a later date.
However, there are occasions, for example when something in the appsettings.json file has been misconfigured, that an exception can be thrown, and the service doesn't start up correctly, but it is still marked as Running.
This situation can occur with any version of CCM up to and including v0.6.2, but it is something that we are looking to address in a future release.
### What is the minimum required configuration for the appsettings.json file?

As of CCM v0.6.2, the default configuration values in the `appsettings.json` for the CCM service are:

```json
{
  "ConnectionStrings": {
    "Default": "Server=<HOST_NAME_OF_MACHINE_BEING_INSTALLED_ONTO>; Database=ChocolateyManagement; Trusted_Connection=True;"
  },
  "CertificateThumbprint": "<THUMBPRINT_OF_CERTIFICATE_FOUND_DURING_INSTALLATION>"
}
```

> :choco-info: **NOTE**
>
> This file will usually be condensed into a single line, with the values encrypted.

If these values are removed or incorrect, the CCM service may fail to start.
To correct this, ensure all configuration is present and correct and then restart the CCM service.

```powershell
Get-Service -Name chocolatey-central-management | Restart-Service
```

### How can we increase the level of logging for Chocolatey Central Management?

This can be done by changing the level value, which should be currently INFO, to use DEBUG, as per the following:

```xml
<root>
  <level value="DEBUG" />
  <appender-ref ref="ColoredConsoleAppender" />
  <appender-ref ref="RollingLogFileAppender" />
</root>
```

In the following files:

* `$env:ChocolateyInstall\lib\chocolatey-management-service\tools\service\log4net.config`. If you are on a version less than 0.2.0, then it will be in `$env:ChocolateyInstall\lib\chocolatey-management-service\tools\service\chocolatey-central-management.exe.config`.
* `$env:ChocolateyInstall\lib\chocolatey-agent\tools\service\chocolatey-agent.exe.config`

When the value is changed, the services may also need restarted.

### How can we view what SSL registrations have been made by the installation of chocolatey-management-service

By default, the installation of the `chocolatey-management-service` package will register a single netsh binding between a self-signed certificate (created at the point of installation) and port 24020.  This can be verified using the following command:

```powershell
netsh http show sslcert
```

<?! Include "../../../shared/netsh-bindings-note.txt" /?>

### How can we remove a netsh binding that has been created

If you need to remove a netsh binding, you can do that using the following command:

```powershell
netsh http delete sslcert ipport=0.0.0.0:<port_number>
```

> :choco-info: **NOTE**
>
> Here `<port_number>` should be replaced with the Port Number that has been registered

<?! Include "../../../shared/netsh-bindings-note.txt" /?>

### Can we manually create an SSL binding?

If required, it is possible to manually create a netsh binding.  This is done using the following command:

```powershell
netsh http add sslcert ipport=0.0.0.0:<port_number> certhash=<certificate_thumbprint> appid={<random_guid>}
```

> :choco-info: **NOTE**
>
> Here, `<port_number>` should be replaced with the Port Number to be used for the registration.  `<certificate_thumbprint>` should be replaced with the thumbprint for the certificate that is to be used for the registration.  `<random_guid>` should be replaced with a random guid in the following format `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

<?! Include "../../../shared/netsh-bindings-note.txt" /?>

### Can we install Central Management Service behind a load balancer?

Unfortunately, it's not a supported scenario. If you are trying to load balance requests to CCM service, you should install multiple instances on multiple machines and point clients explicitly to an instance so they can work together. If you are trying to load balance other things on a machine and CCM service just happens to be there (like with QDE), move CCM service to a different machine or allow direct connections to the box for CCM.

> :choco-info: **NOTE**
>
> If you are an expert in managing X509 certificates with load balancing, you can certainly set this up, but if you can't get it to work, move to a supported scenario. Support folks will tell you the same.

### We want to set up the Chocolatey Central Management service to use a domain account that will have local admin on each box. Can we do this?

Yes, absolutely. You will pass those credentials through at install/upgrade time, and you will also want to turn on the feature useRememberedArgumentsForUpgrades (see [configuration](xref:configuration#features)) so that future upgrades will have that information available. The remembered arguments are stored encrypted on the box (that encryption is reversible so you may opt to pass that information each time).

* `/Username`: - provide username - instead of using the default 'ChocolateyLocalAdmin' user. This user should be a local Administrator.
* `/Password`: - optional password for the user.
* `/EnterPassword` - receive the password at runtime as a secure string

You would pass something like `choco install chocolatey-management-service -y --params="'/Username:domain\account /EnterPassword'"` to securely pass the password at runtime. You could also run `choco install chocolatey-management-service -y --params="'/Username:<domain\account>'" --package-parameters-sensitive="'/Password:<password>'"` (or do it as part of `choco upgrade`).

### Is the password stored anywhere?

No, that would reduce the security of the password. It exists in memory long enough to set the value on user and the service and then it is cleared.

There is no storage of the password anywhere other than how Windows stores passwords.

### We are going to use our own account with a rotating password. When we rotate the password for the account that we use for the Chocolatey Management Service, what do we need to do?

Like with any service that uses rotating passwords, you will need to redeploy the service or go into the services management console and update the password. As it is much faster to deploy out that update, you can do something like `choco upgrade chocolatey-management-service -y --params="'/Username:domain\account'" --package-parameters-sensitive="'/Password:newpassword'" --force` (the `--force` ensures the code is redeployed).

### Tell me more about the Chocolatey managed password.

So you've seen from above that

* It is 32 characters long.
* It uses uppercase, lowercase, numbers, and symbols to meet very stringent complexity requirements.
* The password is different for every machine.
* Due to the way that it is generated, it is completely unguessable.
* No one at Chocolatey Software could even tell you what the password is for a particular machine without local access.

Chocolatey uses something unique about each system, along with an encrypted value in the licensed code base to generate base password, then it makes some other changes to ensure that the password meets complexity requirements. We won't give you the full algorithm of how the password is generated as knowing the algorithm would be a security issue - like having a partial picture of a key, you could start working on how to break in. Unlike a picture of a key, even knowing the full algorithm doesn't get you everything you need as you would need local access to each box to determine the password for **each** machine.

### Is the managed password stored or logged anywhere?

No, that would reduce the security of the password. It exists in memory long enough to set the value on user and the service and then it is cleared.

There is no storage of the password anywhere other than how Windows stores passwords.

### Is the managed password the same on every machine?

No, it is different for every machine it is deployed to.

### How would someone potentially get access to the managed password?

The Chocolatey licensed code base is encrypted, so only people that work at Chocolatey Software would be able to determine the password for a particular box (just that one) **IF** they have local access to that box. Even with all of the information and the algorithm, it's still going to take our folks a while to determine the password. That gets them access to one machine. Of course, Chocolatey folks are not going to do this for obvious reasons.

So let's realize this to its full potential - If someone were able to hack the Chocolatey licensed codebase, they would be able to determine the full password algorithm. Then they'd also need to hack into your infrastructure and get local access to every box that they wanted to get the Chocolatey-managed password so they could get admin access to just that box. Taking this out a bit further, it's reasonable to assume that if someone has hacked into your infrastructure, it's highly unlikely they are going to be using a non-Administrator account to get local access to a box so they can get the password for an Administrator account for just that one box. It's more likely they would would already have a local admin account for the boxes they are attacking, and are likely to seek other attack vectors that are much less sophisticated.

### Do you rotate the managed password on a schedule?

We are looking to do this in a future release. We may make the schedule configurable.

### Can I take advantage of Chocolatey managed passwords with my own Windows services?

Yes, absolutely. If you use C4B's PowerShell Windows Services code, you will be able to install services and have Chocolatey manage the password for those as well.

### What is the CCM compatibility matrix?

Central Management has specific compatibility requirements with quite a few moving parts. It is important to understand that there are some Chocolatey Agent versions that may not be able to communicate with some versions of CCM and vice versa.  Please see the [CCM Component Compatibility Matrix](xref:central-management#ccm-component-compatibility-matrix) for details.

### I entered incorrect database details on install, do I need to reinstall to fix that?

It depends. You can simply go to the appsettings.json file and adjust the connection string to be plaintext. It will remain in plaintext though (at least until upgrade), so if you have actual password details you need to keep secure, you should do a force installation.

1. The file is located at `$env:ChocolateyInstall\lib\chocolatey-management-service\tools\service\appsettings.json`.
1. You would open that up in an editor and modify the `"Default"` connection string.
1. It would look something like the following (adjust the connection string as necessary):

```json
{
  "ConnectionStrings": {
        "Default": "Server=<HOST_NAME_OF_MACHINE_BEING_INSTALLED_ONTO>; Database=ChocolateyManagement; Trusted_Connection=True;"
  }
}
```

1. You may find it all on a single line in the file, and that is okay.
1. Then restart the service by running the following from an admin powershell session: `Get-Service chocolatey-management-service | Stop-Service; Get-Service chocolatey-management-service | Start-Service`

> :choco-warning: **WARNING**
>
> Do not put `sec:` or `secure-` at the start (prefix) of any values that you are adding/modifying directly. That tells Chocolatey components they are encrypted and it will attempt to decrypt them for use. If that is done incorrectly, it will cause things to crash.

### Can we use an account for the service that is not a local Administrator?

This is not a supported scenario, especially considering the installation will attempt to ensure that the user becomes an Administrator if they are not, in addition to `Logon as Service` and `Logon as Batch` privileges.

### Where is the management service installed?

The installation folder for `chocolatey-management-service` is at `$env:ChocolateyInstall\lib\chocolatey-management-service\tools\service`.

___

## Common Errors and Resolutions

### Cannot convert the "System.Object[]" value of type "System.Object[]" to type "System.Int32"

You may see this error message when attempting to install version 0.6.0 of the `chocolatey-management-service` package.

Please see [Licensed Issue #242](https://github.com/chocolatey/chocolatey-licensed-issues/issues/242) for details on a workaround that can be used.  This bug will be addressed in the 0.6.1 release of Chocolatey Central Management.

### Agents are unable to communicate with Chocolatey Central Management Service, CCM service log shows "Unable to start Kestrel"

This is a known issue with CCM v0.6.0 and v0.6.1 due to changes in how the CCM service is hosted.
`netsh` bindings are no longer required for the CCM service, and you may also notice the bindings for the CCM service removed during upgrade.
The identifying symptom of this issue is the following in the CCM service log file:

```log
[FATAL] Microsoft.AspNetCore.Server.Kestrel (0): Unable to start Kestrel
```

In these versions, the best workaround is to ensure that your `LocalMachine\TrustedPeople` certificate store contains only one certificate with a DNS name that matches the CCM service URL setting in your `chocolatey.config` file, and that this certificate has the `ServerAuthentication` usage applied to it.

Starting in v0.6.2, it is possible to configure the CCM service to select a specific certificate to use.
The CCM service log file error will also now contain slightly more information, looking something like this:

```powershell
[FATAL] ChocolateyServiceManagementTask: Microsoft.AspNetCore.Server.Kestrel [0]
Unable to start Kestrel.
System.InvalidOperationException: Certificate 3edebdfee63b57b0a0a12a079ed8da791da03ba7 cannot be used as an SSL server certificate. It has an Extended Key Usage extension but the usages do not include Server Authentication (OID 1.3.6.1.5.5.7.3.1).
```

You may also receive a differently-worded error if the certificate used by the CCM service in the `LocalMachine\TrustedPeople` store is removed.

Starting in CCM v0.6.2, the CCM Service package will attempt to select an appropriate certificate during installation, and store the thumbprint in the `appsettings.json` file.
You can also specify the thumbprint for the certificate to use as the `/CertificateThumbprint` package parameter during installation or upgrade.

If you need to change the certificate you're using after installation, you can modify the entry in the `appsettings.json` file for the CCM service, which looks like this:

```json
{
  "ConnectionStrings": {
    "Default": "Server=<HOST_NAME_OF_MACHINE_BEING_INSTALLED_ONTO>; Database=ChocolateyManagement; Trusted_Connection=True;"
  },
  "CertificateThumbprint": "<THUMBPRINT_OF_CERTIFICATE_FOUND_DURING_INSTALLATION>"
}
```

Altering the `CertificateThumbprint` value will cause the CCM service to select the corresponding certificate from the `LocalMachine\TrustedPeople` store instead.
You will need to restart the CCM service after changing this value for it to take effect:

```powershell
Get-Service -Name chocolatey-central-management | Restart-Service
```

### Chocolatey Agent Service is unable to communicate with Chocolatey Central Management Service

There is a known issue with the beta release of Chocolatey Central Management where an inconsistent Port Number is used between these two services.  One used 24020 and the other used 24040.  The correct default Port Number is 24020, and this is used in the 0.1.0 release of Chocolatey Central Management.  If required, the Port Number can be explicitly set during the installation of the Chocolatey Central Management packages using the following option when installing `chocolatey-management-service`:

```powershell
--params="'/PortNumber=24020'"
```

### Unable to report computer information to CCM

You may see messaging like the following in the chocolatey-agent.log:

```sh
[INFO ] - Creating secure channel to https://ccmserver:24020/ChocolateyManagementService ahead of CCM check-in...
[ERROR] - Unable to report computer information to CCM.:
 The message with Action 'http://tempuri.org/IChocolateyManagementService/report_computer_information' cannot be
 processed at the receiver, due to a ContractFilter mismatch at the EndpointDispatcher. This may be because of either a
 contract mismatch (mismatched Actions between sender and receiver) or a binding/security mismatch between the sender
 and the receiver.  Check that sender and receiver have the same contract and the same binding (including security
 requirements, e.g. Message, Transport, None).
```

This is due to having a Chocolatey Agent that is v0.10.0+ versus an older Central Management Service (< v0.2.0). Newer agents are incompatible because they use newer and more secure methods of communication. Please upgrade Central Management to v0.2.0+ at your earliest convenience. Or if you are on CCM v0.3.0+, your agents need to be on v0.11.0+. Please refer to the [CCM Compatibility Matrix](xref:central-management#ccm-component-compatibility-matrix).

### Unable to check for deployments from CCM

This will provide similar messaging as the above. The fix is the same, upgrade Chocolatey Central Management to v0.2.0+. Or if you are on CCM v0.3.0+, your agents need to be on v0.11.0+. Please refer to the [CCM Compatibility Matrix](xref:central-management#ccm-component-compatibility-matrix). You may need to be on at least v0.3.0 and agents on v0.11.0+ if you are experiencing improper passphrase issues noted below, it means you need to likely upgrade to v0.3.0+ / v0.11.0 across your infrastructure.

### We are seeing the error "attempted to call report_computer_information with an improper passphrase" in the CCM Service log

If you are in the CCM service logs, you may be seeing the above error. That is a bug that was found with the communication of CCM v0.2.0 and Chocolatey Agent v0.10.0. That was resolved in CCM v0.3.0 and Chocolatey Agent v0.11.0. Please see the [CCM Component Compatibility Matrix](xref:central-management#ccm-component-compatibility-matrix) and [Licensed Issue #152](https://github.com/chocolatey/chocolatey-licensed-issues/issues/152) for more details.

### The client reports successful check-in, but nothing is showing up in CCM

You need to check the CCM service logs. The agent will always report success when it communicates with the service successfully. The service may reject what it receives, but due to security settings, it won't tell the client about that.

The logs are located at `$env:ChocolateyInstall\logs\ccm-service.log`. If you are on a version of CCM prior to 0.2.0, the log will be located at `$env:ChocolateyInstall\lib\chocolatey-management-service\tools\service\logs\chocolatey.service.host.log`.

### A parameter cannot be found that matches parameter name KeyUsage

This is known issue with the beta release of Chocolatey Central Management regarding the creation of a Self Signed Certificate.  You may see the error:

`A parameter cannot be found that matches parameter name KeyUsage`

This happens when installing Chocolatey Central Management on a machine that has PowerShell 4 or earlier.  This is corrected in the 0.1.0 release of Chocolatey Central Management.

To work around this issue, you can use the following script to manually create a Self Signed Certificate, which will then be used to continue the installation:

```powershell
$hostName = [System.Net.Dns]::GetHostName()
$domainName = [System.Net.NetworkInformation.IPGlobalProperties]::GetIPGlobalProperties().DomainName

if(-Not $hostName.endswith($domainName)) {
  $hostName += "." + $domainName
}

$certificateDnsName = $hostName

$newCert = New-SelfSignedCertificate -CertStoreLocation cert:\LocalMachine\My -DnsName $certificateDnsName

# move the certificate to 'TrustedPeople'
$certPath = Get-ChildItem -Path 'Cert:\\LocalMachine\\My' | Where-Object subject -like "*$certificateDnsName"
$null = Move-Item -Path $certPath.PsPath -Destination 'Cert:\\LocalMachine\\TrustedPeople'
```

### The term 'Install-ChocolateyAppSettingsJsonFile' is not recognized as the name of a cmdlet, function, script file, or operable program.

In the beta version of Chocolatey.Extension, there was a Cmdlet named Install-ChocolateyAppSettingsJsonFile and this was used in the 0.1.0-beta-20181009 release of the Chocolatey Central Management components. In the final released version of the Chocolatey.Extension, this was renamed to Install-AppSettingsJsonFile.

As a result, the Chocolatey Central Management beta no longer works with the released version of Chocolatey.Extension. This will be corrected once the next release of the Chocolatey Central Management components is completed.

### Cannot process command because of one or more missing mandatory parameters: FilePath

During the creation of Chocolatey Central Management, some additional PowerShell cmdlets were created, and these are installed as part of the Chocolatey Extension package.  These cmdlets went through a number of iterations, and as a result, different combinations of Chocolatey Central Management packages were incompatible with the Chocolatey Extension package, resulting in the error:

`Cannot process command because of one or more missing mandatory parameters: FilePath`

The guidance in this case is either to pin to the specific version of the Chocolatey Extension package required by the version of Chocolatey Central Management being used, or, update to the latest versions of all packages, where the situation should be addressed.

### The remote server returned an unexpected response: (413) Request Entity Too Large

When reporting a larger number of packages (approximately 200), this error may be reported.  This is due to the size of the information, in bytes, being too large to send between the Chocolatey Agent Service and the Chocolatey Central Management Service.  This has been identified as a [bug](https://github.com/chocolatey/chocolatey-licensed-issues/issues/95), which is due to be corrected in version 0.1.1 of Chocolatey Central Management

### ERROR: Cannot index into a null array

This error can be reported when installing the Chocolatey Central Management Service.  This can happen depending on the netsh binding that are currently present on the machine that is being installed on.  If for example, you have enabled SNI on a website on the machine that you are installing onto, then this error may occur.  This has been identified as a [bug](https://github.com/chocolatey/chocolatey-licensed-issues/issues/96), which is due to be corrected in version 0.1.1 of Chocolatey Central Management.

This could also be when you are providing an existing certificate - see <https://github.com/chocolatey/chocolatey-licensed-issues/issues/143>. This was fixed in v0.2.0.

There is a known issue with CCM v.0.3.0 affecting at least French machines. Please reach out to support if you are experiencing this (run `choco support` for options).

### The new license is not being picked up

You need to restart services and CCM web to pick up the license. Here's a handy script:

```powershell
Get-Service chocolatey-* | Stop-Service
Get-Process -Name "ChocolateySoftware.ChocolateyManagement.Web.Mvc" -ErrorAction SilentlyContinue | Stop-Process
Get-Service chocolatey-* | Start-Service
```

### Failed to generate a user instance of SQL Server due to failure in retrieving the user's local application data path.

You may see the following: "System.Data.SqlClient.SqlException (0x80131904): Failed to generate a user instance of SQL Server due to failure in retrieving the user's local application data path. Please make sure the user has a local user profile on the computer. The connection will be closed."

This means you are attempting to attach a Local DB file as part of your connection. This is an invalid scenario as noted at [Use Windows Account to Attach SQL Server](#use-windows-account-to-attach-sql-server).

This could also mean that there is something wrong with your connection strings format in the `appsettings.json`.

### System.ServiceModel.AddressAccessDeniedException HTTP could not register URL

You may see the following: `System.ServiceModel.AddressAccessDeniedException: HTTP could not register URL https://+:24020/ChocolateyManagementService/. Your process does not have access rights to this namespace (see http://go.microsoft.com/fwlink/?LinkId=70353 for details). ---> System.Net.HttpListenerException: Access is denied`

You are attempting to set up a user that is not in the local Administrators group with the Central Management Service. While this is not officially supported, you can use the following to ensure all is good:

* Ensure the user has `Logon As Service` privilege
* Ensure the user has `Logon as Batch` privilege
* Run `netsh http add urlacl url=https://+:24020/ChocolateyManagementService user=<DOMAIN\USERNAME>` from an elevated shell (replacing `<DOMAIN\USERNAME>` with the account)

<?! Include "../../../shared/netsh-bindings-note.txt" /?>

[Central Management Setup](xref:ccm-setup) | [Chocolatey Central Management](xref:central-management)
