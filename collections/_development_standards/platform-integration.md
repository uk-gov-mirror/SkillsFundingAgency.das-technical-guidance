---
layout: default
title:  "Platform Integration"
category: development_standards
---

## Platform Integration

Below are described the patterns currently in-use across the Service for integrating applications with Azure resources.

### Managed Identities (MI)

Using a managed identity allows an application to authenticate to any service that supports Azure AD authentication without having credentials.  This has several key benefits:
* Improved security - credentials don't exist so are not accessible to anyone
* Applications won't experience expired secrets resulting in:
  * Improved development/testing experience
  * No management overhead on redeploying applications in order to renew secrets
  * No risk of downtime due to expiring secrets
* Simplified configuration to reference for applications

Examples where Managed Identity Azure AD authentication should be used:
* Acquiring a bearer token for an internal API behind Azure AD authentication
* Connecting to a SQL database
* Connecting to a Service Bus namepsace


The [Microsoft.Azure.Services.AppAuthentication](https://www.nuget.org/packages/Microsoft.Azure.Services.AppAuthentication) library manages authentication automatically

#### Example - Acquiring a bearer token for an internal API behind Azure AD authentication:

```csharp
public class ApiClient
{
    private readonly HttpClient _httpClient;

    public ApiClient(IHttpClientFactory httpClientFactory) {
        _httpClient = httpClientFactory.CreateClient();
    }

    public async Task<string> GetAccessTokenAsync(string identifier)
    {
        var azureServiceTokenProvider = new AzureServiceTokenProvider();
        var accessToken = await azureServiceTokenProvider.GetAccessTokenAsync(identifier);
        
        return accessToken;
    }

    # Acquire a bearer token
    var identifier = "https://tenant.onmicrosoft.com/das-env-api-as-ar" # IdentifierUri of Azure AD App registration
    var accessToken = await GetAccessTokenAsync(identifier);

    # Add the token as a header
    _httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);

    # Make an API call
    var response = await _httpClient.GetAsync("https://azurewebsites.net/resource/resourceid");
}
```

#### Example - Connecting to a SQL database:

Creating a SqlConnection:

```csharp
public AppDataContext()
{
}

public AppDataContext(DbContextOptions options) : base(options)
{
}

public AppDataContext(IOptions<AppConfiguration> config, DbContextOptions options, AzureServiceTokenProvider azureServiceTokenProvider) :base(options)
{
    _configuration = config.Value;
    _azureServiceTokenProvider = azureServiceTokenProvider;
}  

protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    if (_configuration == null || _azureServiceTokenProvider == null)
    {
        return;
    }
    
    var connection = new SqlConnection
    {
        ConnectionString = _configuration.ConnectionString,
        AccessToken = _azureServiceTokenProvider.GetAccessTokenAsync(AzureResource).Result,
    };
}
```
As used in: [Courses API CoursesDataContext.cs](https://github.com/SkillsFundingAgency/das-courses-api/blob/91459aadbf90ff7101022d49b008b18169968fca/src/SFA.DAS.Courses.Data/CoursesDataContext.cs)

An example of the database extension being added for the AppStart:
```csharp
public static class AddDatabaseExtension
{
    public static void AddDatabaseRegistration(this IServiceCollection services, AppConfiguration config, string environmentName)
    {
        if (environmentName.Equals("DEV", StringComparison.CurrentCultureIgnoreCase))
        {
            services.AddDbContext<AppDataContext>(options => options.UseInMemoryDatabase("SFA.DAS.App"), ServiceLifetime.Transient);
        }
        else if (environmentName.Equals("LOCAL", StringComparison.CurrentCultureIgnoreCase))
        {
            services.AddDbContext<AppDataContext>(options=>options.UseSqlServer(config.ConnectionString),ServiceLifetime.Transient);
        }
        else
        {
            services.AddSingleton(new AzureServiceTokenProvider());
            services.AddDbContext<AppDataContext>(ServiceLifetime.Transient);    
        }

        services.AddTransient<IAppDataContext, AppDataContext>(provider => provider.GetService<AppDataContext>());
        services.AddTransient(provider => new Lazy<AppDataContext>(provider.GetService<AppDataContext>()));
    }
}
```

As used in: [Courses API AddDatabaseExtension.cs](https://github.com/SkillsFundingAgency/das-courses-api/blob/2c83a21b65bde54ee8ceed9d29c812880cf401bf/src/SFA.DAS.Courses.Api/AppStart/AddDatabaseExtension.cs)

#### References

* [Managed Identity Documentation](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/)
* [MI Overview - Microsoft](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)
* [MI Obtain tokens for Azure resources - Microsoft](https://docs.microsoft.com/en-us/azure/app-service/overview-managed-identity?tabs=dotnet#asal)
* [MI Connecting to a SQL database - Microsoft](https://docs.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-connect-msi)

### ASP.NET Core Data Protection

Web applications often need to store security-sensitive data. The ASP.NET Core data protection stack is designed to serve as the long-term replacement for the `<machineKey>` element in ASP.NET 1.x - 4.x. One common use of Data Protection is the storing of cookies for apps using the standard ASP.NET Core cookie authentication.

By default, apps hosted on app services have data protection keys that are persisted to the `%HOME%\ASP.NET\DataProtection-Keys` folder. This folder is backed by network storage and is synchronized across all machines hosting the app.

Separate deployment slots (Staging and Production) don't share a key ring, meaning the standard slot swapping deployment that takes place in releases would result in an app using Data Protection to not be able to decrypt stored data using the key ring inside the previous slot.

Additionally, Data Protection exceptions are seen on apps that do not persist to Redis Cache on Azure instances:

| Log property name | Log property value  |
|---|---|
| Exception.type   | AntiforgeryValidationException  |
| Exception.innerException.source  | Microsoft.AspNetCore.DataProtection  |
| Exception.innerException.message  | The key {GUID} was not found in the key ring.  |

These exceptions correlate with users receiving an error page when signing in.

To maintain zero-downtime releases for Apprenticeship Service applications and prevent other Data Protection exceptions, Data Protection keys should be persisted to the environment's Redis Cache, where the redis connection string and database number are in the application's configuration.

#### Example

```csharp
using Microsoft.AspNetCore.DataProtection;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using SFA.DAS.AppName.Configuration;
using StackExchange.Redis;

namespace SFA.DAS.AppName.Web.Startup
{
    public static class DataProtectionStartupExtensions
    {
        public static IServiceCollection AddDataProtection(this IServiceCollection services, IConfiguration configuration, IHostingEnvironment environment)
        {
            if (!environment.IsDevelopment())
            {
                var redisConfiguration = configuration.GetSection(ConfigurationKeys.ConnectionStrings)
                    .Get<AppNameSettings>();

                if (redisConfiguration != null)
                {
                    var redisConnectionString = redisConfiguration.RedisConnectionString;
                    var dataProtectionKeysDatabase = redisConfiguration.DataProtectionKeysDatabase;

                    var redis = ConnectionMultiplexer
                        .Connect($"{redisConnectionString},{dataProtectionKeysDatabase}");

                    services.AddDataProtection()
                        .SetApplicationName("RepositoryName")
                        .PersistKeysToStackExchangeRedis(redis, "DataProtection-Keys");
                }
            }
            return services;
        }
    }
}
```

[Example](https://github.com/SkillsFundingAgency/das-employercommitments-v2/pull/137/files) of persisting data protection keys in an Apprenticeship Service app.

#### References

* [ASP.NET Core Data Protection overview](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/introduction?view=aspnetcore-5.0)
* [Configuring ASP.NET Core Data Protection](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/overview?view=aspnetcore-3.1)
* [Data Protection key management and lifetime in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/default-settings?view=aspnetcore-3.1)

### Logging to Redis and not to File

All applications use NLog to write logs to logging Redis Cache on Azure instances.

The following flow is applied:

1. App Services use NLog to write to Redis Caches.
1. Each Redis Cache buffers the logs.
1. Logstash indexes the logs from Redis Cache and performs data transformation and processing.
1. Kibana is used to visualise the data in Elasticsearch.

Logging to File, as configured in the NLog configuration, results in a storage exception such as `There is not enough space on the disk : 'D:\home\site\wwwroot\logs\app-name.YYYY-MM-DD.log'` seen in a profiler trace due to a local cache limit of 1GB being reached.

These exceptions correlate with users receiving 504 Gateway Timeout error pages.

Only Redis should be logged to and not File as:
* Kibana is reliably available.
* Application Insights is the backup solution, Azure's [Application Insights SLA](https://azure.microsoft.com/en-us/support/legal/sla/application-insights/v1_2/) guarantees query availability will not fall below 99.9%.
* Checking logs on individual app services is impractical for debugging.
* High level Contributor permissions are needed to access the App Services' files, these permissions should be restricted.
* Not logging to File prevents the storage exceptions.

For local development, it is useful to log to File.

[Example](https://github.com/SkillsFundingAgency/das-providercommitments/pull/176/files) of NLog configuration, with logging to File for develeopment mode and logging to Redis for non-development mode (running on Azure app services).

#### References

* [App Service local cache size limits](https://docs.microsoft.com/en-us/azure/app-service/overview-local-cache#how-the-local-cache-changes-the-behavior-of-app-service)
