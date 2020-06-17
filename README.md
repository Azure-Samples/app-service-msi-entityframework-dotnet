---
services: app-service, sql
platforms: dotnet
author: nonik0
---

# Use Azure SQL Database from App Service with Managed Identity (Without Code Changes)

## Background
For AAD-based authentication to Azure SQL Database, developers who wanted their existing SQL applications to use managed identities and AAD-based authentication were required to make code changes to retrieve and set the access token used for authentication. As many developers are familiar with the App Authentication library, they opted to use it to retrieve the access token, as below:  

```csharp    
SqlConnection connection = new SqlConnection(connectionString); 
connection.AccessToken = await (new AzureServiceTokenProvider()).GetAccessTokenAsync("https://database.windows.net/",<Tenant-ID>); 
connection.Open(); 
```

However, implementing and testing these code changes were often not straightforward. This was compounded further with EntityFramework, where these code changes were more difficult and error-prone for developers. Consequently, many developers of existing SQL applications opted to continue to use insecure connection strings containing credentials for authentication, rather than dealing with the code churn that would be required to switch to AAD-based authentication.

With new functionality in the AppAuthentication library, these problems are solved. This sample is a slight modification of an [existing EntityFramework sample](https://docs.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-dotnet-sqldatabase), with only configuration changes. It uses the new SqlAppAuthenticationProvider functionality in the AppAuthentication library to authenticate to your Azure SQL Database, removing the need to either use insecure connection strings or make code changes in order to use AAD-based authentication.


## Prerequisites
To run and deploy this sample, you need the following:
1. An Azure subscription to create an App Service and a Key Vault.
2. [Visual Studio 2017 v15.6](https://blogs.msdn.microsoft.com/visualstudio/2017/12/07/visual-studio-2017-version-15-6-preview/) or greater to run the application on your local development machine.
3. [>=.NET Framework 4.7.2](https://dotnet.microsoft.com/download)
4. [SQL Server Management Studio](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-2017) (SSMS)

## Step 1: Create an App Service with a Managed Identity
<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure-Samples%2Fapp-service-msi-entityframework-dotnet%2Fmaster%2Fazuredeploy.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>

Use the "Deploy to Azure" button to deploy an ARM template to create the following resources:
1. App Service with Managed Identity.
2. SQL server with SQL database.
>Note: When filling out the template you will see a textbox labelled 'Web Site Name'. This will be the name of your deployed App Service and you should record this name for a later step.

## Step 2: Grant yourself access to the SQL server
Using the Azure Portal, go to the settings page of your SQL server. Set your own account as the Active Directory admin and create a firewall rule allowing the client IP of your developer machine to access the SQL server.

To set admin and add firewall rule:
1.	Search for your SQL server in “Search Resources dialog box” in Azure Portal.
2.	In the left hand list of settings, click on "Admin Directory admin"
3.	Click on "Set admin", search for and select your Active Directory account, and then click "Save".
4.	Now, in the left hand list of settings, click on "Firewalls and virtual networks"
5. Click "Add client IP" and then click "Save" to add a firewall rule (or manually create for your client machine).

## Step 3: Clone the repo
Clone the repo to your development machine.

The project has two relevant NuGet packages:
1. Microsoft.Azure.Services.AppAuthentication (preview) - makes it easy to fetch access tokens for Service-to-Azure-Service authentication scenarios.
2. EntityFramework - Object Relational Mapper (ORM) framework that makes it easy to map objects to SQL database

This code is mostly unchanged from an [existing EntityFramework sample](https://docs.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-dotnet-sqldatabase). There are only three changes from the original sample that have already been made:
1. The project was updated to target .NET Framework 4.7.2
2. AppAuthentication Nuget package has been added to packages.config
3. Web.config has been updated with 3 specific changes, outlined below.

### Web.config changes
The Web.config file's location is WebAppEntityFramework/WebAppEntityFramework/Web.config. The three changes are outlined below:
	
#### Web.config Change #1
A new custom configuration setting element is added to the configSections element. This element adds a new SqlAuthenticationProviders configuration section, where a SQL authentication provider can be specified. This is new to System.Data.SqlClient in .NET 4.7.2.

```xml
<section name="SqlAuthenticationProviders" type="System.Data.SqlClient.SqlAuthenticationProviderConfigurationSection, System.Data, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" />
```

#### Web.config Change #2
A new SqlAuthenticationProviders configuration section is added, which specifies "SqlAppAuthenticationProvider" as a SQL authentication provider. The "SqlAppAuthenticationProvider" is a new class, available only in the .NET 4.7.2 targeted package of the AppAuthentication library. It uses the AzureServiceTokenProvider to acquire an access token for SQL.
```xml
  <SqlAuthenticationProviders>
    <providers>
      <add name="Active Directory Interactive" type="Microsoft.Azure.Services.AppAuthentication.SqlAppAuthenticationProvider, Microsoft.Azure.Services.AppAuthentication" />
    </providers>
  </SqlAuthenticationProviders>
```

#### Web.config Change #3
The connection string has been updated. Note that the value of the "Authentication" parameter matches the name of the SQL Authentication Provider in the SqlAuthenticationProviders configuration section.
```xml
<add name="MyDbConnection" connectionString="server=[Your SQL Server Name].database.windows.net;database=[Your SQL DB name];UID=AnyString;Authentication=Active Directory Interactive" providerName="System.Data.SqlClient" />
```

## Step 4: Update the connection string in the Web.config
In the Web.config file, update the "server" and "database" parameter values to the name of your SQL server and SQL database, respectively.

## Step 5: Run the application on your local development machine
When running your sample, the application will use the AppAuthentication library's SqlAppAuthenticationProvider that was configured in Web.config, which uses the developer's security context to get a token to authenticate to Azure SQL Database. This removes the need to use a connection string that contains credentials and share it with the development team. It also prevents such connection strings from being checked in to source code. For local development, the SqlAppAuthenticationProvider will use **Visual Studio**, **Azure CLI**, or **Active Directory Integrated Authentication** to authenticate to Azure AD to get a token. That token will be used to both send and receive data from your Azure SQL database. When running your application for the first time, it will create the schema for your Azure SQL database that your application will use and thus will take longer than usual to load.

Visual Studio authentication will work if the following conditions are met:
 1. You have installed [Visual Studio 2017 v15.6](https://blogs.msdn.microsoft.com/visualstudio/2017/12/07/visual-studio-2017-version-15-6-preview/) or later.
 2. You are signed in to Visual Studio and have selected an account to use for local development. Use Tools > Options > Azure Service Authentication to choose a local development account. 

Azure CLI will work if the following conditions are met:
 1. You have [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) installed. Version 2.0.12 supports the get-access-token option used by AzureServiceTokenProvider. If you have an earlier version, please upgrade. 
 2. You are logged into Azure CLI. You can login using **az login** command.
 
Azure Active Directory authentication will only work if the following conditions are met:
 1. Your on-premise active directory is synced with Azure AD. 
 2. You are running this code on a domain joined machine.   

Since your developer account has SQL admin access to your SQL Server instance, you should be able to run queries against your SQL Server instance using the web app's interface.

You can also use a service principal to run the application on your local development machine. See the section "Running the application using a service principal" in the [documentation for the AppAuthentication library](https://docs.microsoft.com/en-us/azure/key-vault/service-to-service-authentication#running-the-application-using-a-service-principal) for how to do this.
>Note: It is recommended to use your developer context for local development, since you do not need to create or share a service principal for that. If that does not work for you, you can use a service principal, but do not check in the certificate or secret in source repos, and share them securely.

## Step 6: Grant App Service Managed Identity access to the SQL server
Using SQL Server Management Studio, authenticate using your developer account to gain access your Azure SQL database. Then edit the below query, replacing the App Service name with the name you used in Step #1, and execute it. This will grant your App Service permission to read and write to your database.

```sql
CREATE USER [<App Service Name>] FROM EXTERNAL PROVIDER
CREATE USER [<App Service Name>/slots/<slotName>] FROM EXTERNAL PROVIDER -- For Web App in slots

ALTER ROLE db_datareader ADD MEMBER [[<App Service Name>] -- gives permission to read to database
ALTER ROLE db_datawriter ADD MEMBER [[<App Service Name>] -- gives permission to write to database
```

For more detailed instructions on using SQL Server Management Studio to connect to your database, please refer [here](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-connect-query-ssms).

## Step 7: Deploy the EntityFramework Web App to Azure
Use any of the methods outlined on [Deploy your app to Azure App Service](https://docs.microsoft.com/en-us/azure/app-service-web/web-sites-deploy) to publish the Web App to Azure. 
After you deploy it, browse to the web app. The web app should load and you should be able to create/manage the TODO list, and also see your previous changes when you ran the web app locally. However, different from Step 5, the SqlAppAuthenticationProvider will use the web app's own Managed Identity to authenticate to SQL instead of your local developer context. This did not require any code or configuration changes between local development and being published to Azure.

## Troubleshooting

Please see the [troubleshooting section](https://docs.microsoft.com/en-us/azure/key-vault/service-to-service-authentication#appauthentication-troubleshooting) of the AppAuthentication library documentation for troubleshooting of common issues.
