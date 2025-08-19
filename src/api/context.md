# .NET API Agent Context

## Agent Role & Responsibilities
**Backend API & Authentication Specialist**

**Scope**: REST API, Azure AD integration, Data access layer

## Primary Tasks
1. Build .NET 8 Web API with clean architecture
2. Implement Azure AD authentication with proper scopes  
3. Create data access layer connecting to Gold layer tables
4. Add caching, logging, and performance monitoring
5. Comprehensive testing with xUnit

## Context Requirements

### READ FIRST  
Review the following documentation before beginning implementation:
- [Data Schema Reference](../../docs/schema.md) - Gold layer table definitions and API models
- [Technology Stack](../../docs/tech-stack.md) - .NET configuration and authentication setup

### Architecture Pattern
**Clean Architecture** with dependency injection:
- **Controllers**: HTTP request handling and validation
- **Services**: Business logic implementation  
- **Repositories**: Data access abstraction
- **Models**: DTOs and domain entities

### Security Requirements
- **Azure AD OAuth2** with role-based access control
- **Managed Identity** for database connections
- **Rate limiting**: 100 requests/minute per user
- **Input validation** on all endpoints

### Performance Standards
- **Response time**: <1 second for 95% of requests
- **Concurrent users**: Support 1000+ simultaneous connections
- **Caching**: 5-minute TTL for frequently accessed data
- **Connection pooling**: Optimize database connections

## API Specifications

### Authentication Scopes
```json
{
  "scopes": {
    "api://dataplatform/read": {
      "description": "Read-only access to customer and sales data",
      "endpoints": ["GET /api/customers/*", "GET /api/sales/*"]
    },
    "api://dataplatform/write": {
      "description": "Write access to update customer information", 
      "endpoints": ["POST /api/customers", "PUT /api/customers/*"]
    },
    "api://dataplatform/admin": {
      "description": "Administrative operations and device management",
      "endpoints": ["GET /api/devices/alerts", "POST /api/admin/*"]
    }
  }
}
```

### Core API Endpoints

#### Customer Endpoints
```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class CustomersController : ControllerBase
{
    // GET /api/customers/{id} - Customer 360 view
    [HttpGet("{id}")]
    [RequiredScope("api://dataplatform/read")]
    public async Task<CustomerProfileResponse> GetCustomerProfile(string id);
    
    // GET /api/customers/{id}/devices - Customer devices
    [HttpGet("{id}/devices")]
    [RequiredScope("api://dataplatform/read")]  
    public async Task<List<DeviceSummary>> GetCustomerDevices(string id);
}
```

#### Sales Endpoints  
```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class SalesController : ControllerBase
{
    // GET /api/sales/daily?region={region} - Daily sales aggregates
    [HttpGet("daily")]
    [RequiredScope("api://dataplatform/read")]
    public async Task<DailySalesResponse> GetDailySales(string? region = null);
    
    // GET /api/sales/trends?days={days} - Sales trends
    [HttpGet("trends")]
    [RequiredScope("api://dataplatform/read")]
    public async Task<SalesTrendResponse> GetSalesTrends(int days = 30);
}
```

#### Device Endpoints
```csharp
[ApiController] 
[Route("api/[controller]")]
[Authorize]
public class DevicesController : ControllerBase
{
    // GET /api/devices/{id}/status - Latest device status
    [HttpGet("{id}/status")]
    [RequiredScope("api://dataplatform/read")]
    public async Task<DeviceStatusResponse> GetDeviceStatus(string id);
    
    // GET /api/devices/alerts - Devices needing attention
    [HttpGet("alerts")]
    [RequiredScope("api://dataplatform/admin")]
    public async Task<DeviceAlertsResponse> GetDeviceAlerts();
}
```

## Data Models

### Response DTOs
```csharp
public class CustomerProfileResponse
{
    public string CustomerId { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public string Region { get; set; }
    public string SubscriptionTier { get; set; }
    public decimal LifetimeValue { get; set; }
    public int DeviceCount { get; set; }
    public DateTime? LastOrderDate { get; set; }
    public List<DeviceSummary> Devices { get; set; }
    public List<RecentOrder> RecentOrders { get; set; }
}

public class DeviceAlertsResponse
{
    public List<DeviceAlert> Alerts { get; set; }
    public int TotalCount { get; set; }
    public DateTime GeneratedAt { get; set; }
}

public class DeviceAlert
{
    public string DeviceId { get; set; }
    public string CustomerId { get; set; }
    public string AlertType { get; set; }
    public string Severity { get; set; }
    public int? BatteryLevel { get; set; }
    public DateTime LastSeen { get; set; }
    public string RecommendedAction { get; set; }
}
```

### Domain Entities (matching Gold layer)
```csharp
public class Customer
{
    public string CustomerId { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public string Region { get; set; }
    public string SubscriptionTier { get; set; }
    public DateTime SignupDate { get; set; }
    public decimal LifetimeValue { get; set; }
}

public class Device  
{
    public string DeviceId { get; set; }
    public string CustomerId { get; set; }
    public string ProductId { get; set; }
    public DateTime PurchaseDate { get; set; }
    public DateTime? LastEventTime { get; set; }
    public string Status { get; set; } // "Healthy", "Needs Service", "Offline"
    public int HealthScore { get; set; }
}
```

## Data Access Patterns

### Repository Pattern Implementation
```csharp
public interface ICustomerRepository
{
    Task<Customer?> GetByIdAsync(string customerId);
    Task<CustomerProfileResponse> GetCustomerProfileAsync(string customerId);
    Task<List<Customer>> GetByRegionAsync(string region);
    Task<bool> UpdateCustomerAsync(Customer customer);
}

public class CustomerRepository : ICustomerRepository
{
    private readonly IDbConnection _connection;
    private readonly IMemoryCache _cache;
    private readonly ILogger<CustomerRepository> _logger;
    
    public CustomerRepository(IDbConnection connection, IMemoryCache cache, ILogger<CustomerRepository> logger)
    {
        _connection = connection;
        _cache = cache;
        _logger = logger;
    }
    
    public async Task<CustomerProfileResponse> GetCustomerProfileAsync(string customerId)
    {
        // Check cache first
        var cacheKey = $"customer-profile-{customerId}";
        if (_cache.TryGetValue(cacheKey, out CustomerProfileResponse cachedResult))
        {
            return cachedResult;
        }
        
        // Query Gold layer tables with joins
        const string sql = @"
            SELECT c.customer_id, c.name, c.email, c.region, c.subscription_tier,
                   c.lifetime_value, c.signup_date,
                   d.device_id, d.status, d.health_score, d.last_event_time,
                   s.order_id, s.amount_usd, s.order_date
            FROM gold.dim_customer c
            LEFT JOIN gold.dim_device d ON c.customer_id = d.customer_id  
            LEFT JOIN gold.fact_sales s ON c.customer_id = s.customer_id 
                AND s.order_date >= DATEADD(month, -6, GETDATE())
            WHERE c.customer_id = @CustomerId
            ORDER BY s.order_date DESC";
            
        // Execute query and map to response object
        var result = await _connection.QueryAsync<CustomerProfileResponse, DeviceSummary, RecentOrder, CustomerProfileResponse>(
            sql, 
            (customer, device, order) => 
            {
                // Mapping logic
                return customer;
            },
            new { CustomerId = customerId },
            splitOn: "device_id,order_id"
        );
        
        // Cache for 5 minutes
        var profile = result.FirstOrDefault();
        if (profile != null)
        {
            _cache.Set(cacheKey, profile, TimeSpan.FromMinutes(5));
        }
        
        return profile;
    }
}
```

### Connection String Management
```csharp
public class DatabaseSettings
{
    public string ConnectionString { get; set; }
    public int CommandTimeout { get; set; } = 30;
    public int MaxPoolSize { get; set; } = 100;
    public bool EnableRetryOnFailure { get; set; } = true;
}

// Startup.cs configuration
services.Configure<DatabaseSettings>(configuration.GetSection("Database"));
services.AddScoped<IDbConnection>(provider =>
{
    var settings = provider.GetRequiredService<IOptions<DatabaseSettings>>().Value;
    return new SqlConnection(settings.ConnectionString);
});
```

## Performance Optimization

### Caching Strategy
```csharp
public class CacheService : ICacheService
{
    private readonly IMemoryCache _memoryCache;
    private readonly IDistributedCache _distributedCache;
    
    public async Task<T> GetOrSetAsync<T>(string key, Func<Task<T>> getItem, TimeSpan? expiry = null)
    {
        // Try memory cache first (fastest)
        if (_memoryCache.TryGetValue(key, out T cachedValue))
        {
            return cachedValue;
        }
        
        // Try distributed cache (Redis)
        var distributedValue = await _distributedCache.GetStringAsync(key);
        if (distributedValue != null)
        {
            var result = JsonSerializer.Deserialize<T>(distributedValue);
            _memoryCache.Set(key, result, expiry ?? TimeSpan.FromMinutes(5));
            return result;
        }
        
        // Generate new value
        var newValue = await getItem();
        var serialized = JsonSerializer.Serialize(newValue);
        
        // Store in both caches
        await _distributedCache.SetStringAsync(key, serialized, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = expiry ?? TimeSpan.FromMinutes(15)
        });
        _memoryCache.Set(key, newValue, expiry ?? TimeSpan.FromMinutes(5));
        
        return newValue;
    }
}
```

### Rate Limiting
```csharp
// Configure rate limiting in Program.cs
services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("UserPolicy", configure =>
    {
        configure.PermitLimit = 100;
        configure.Window = TimeSpan.FromMinutes(1);
        configure.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        configure.QueueLimit = 10;
    });
});

// Apply to controllers
[EnableRateLimiting("UserPolicy")]
[ApiController]
public class CustomersController : ControllerBase { }
```

## Configuration & Deployment

### Application Settings
```json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "Domain": "yourdomain.onmicrosoft.com",
    "TenantId": "your-tenant-id",
    "ClientId": "your-client-id"
  },
  "Database": {
    "ConnectionString": "@Microsoft.KeyVault(SecretUri=https://kv-dataplatform-dev.vault.azure.net/secrets/db-connection)",
    "CommandTimeout": 30,
    "MaxPoolSize": 100
  },
  "Cache": {
    "Redis": "@Microsoft.KeyVault(SecretUri=https://kv-dataplatform-dev.vault.azure.net/secrets/redis-connection)",
    "DefaultExpiration": "00:05:00"
  },
  "ApplicationInsights": {
    "ConnectionString": "@Microsoft.KeyVault(SecretUri=https://kv-dataplatform-dev.vault.azure.net/secrets/appinsights-connection)"
  }
}
```

### Dependency Injection Setup
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

// Data access
builder.Services.Configure<DatabaseSettings>(builder.Configuration.GetSection("Database"));
builder.Services.AddScoped<ICustomerRepository, CustomerRepository>();
builder.Services.AddScoped<ISalesRepository, SalesRepository>();
builder.Services.AddScoped<IDeviceRepository, DeviceRepository>();

// Services  
builder.Services.AddScoped<ICustomerService, CustomerService>();
builder.Services.AddScoped<ISalesService, SalesService>();
builder.Services.AddScoped<IDeviceService, DeviceService>();

// Caching
builder.Services.AddMemoryCache();
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
});

// Monitoring
builder.Services.AddApplicationInsightsTelemetry();
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("Database"))
    .AddRedis(builder.Configuration.GetConnectionString("Redis"));
```

## Deliverables

### 1. Web API Project (`src/api/DataPlatform.API/`)
- **Controllers/**: HTTP endpoints for each domain
- **Models/**: DTOs and request/response objects  
- **Services/**: Business logic implementation
- **Configuration/**: Settings and dependency injection
- **Program.cs**: Application startup and middleware
- **DataPlatform.API.csproj**: Project dependencies

### 2. Testing Project (`src/api/DataPlatform.Tests/`)
- **Controllers/**: Controller unit tests
- **Services/**: Service layer tests
- **Integration/**: End-to-end API tests
- **DataPlatform.Tests.csproj**: Test dependencies

### 3. API Documentation
- **Swagger/OpenAPI**: Automatically generated from code annotations
- **README.md**: Authentication setup and endpoint examples

## Quality Gates

Before marking deliverables complete:

- ✅ All endpoints properly authenticated with Azure AD
- ✅ Response times under 1 second for 95% of requests
- ✅ Proper error handling and logging implemented  
- ✅ API documentation generated with Swagger
- ✅ Unit and integration tests achieve >85% coverage
- ✅ Security scanning passes (no high/critical issues)
- ✅ Performance testing validates concurrent user load

## Dependencies & Handoffs  

### Prerequisites
- **Infrastructure Agent**: App Service and Key Vault deployed
- **Databricks Agent**: Gold layer tables available with sample data

### Provides to Other Agents
**File**: `src/api/outputs/api-specification.json`
```json
{
  "baseUrl": "https://app-dataplatform-dev.azurewebsites.net",
  "endpoints": {
    "customerProfile": "GET /api/customers/{id}",
    "dailySales": "GET /api/sales/daily?region={region}",
    "deviceAlerts": "GET /api/devices/alerts"
  },
  "authentication": {
    "type": "AzureAD",
    "scopes": ["api://dataplatform/read", "api://dataplatform/write", "api://dataplatform/admin"]
  }
}
```

## Monitoring & Troubleshooting

### Key Metrics  
```csharp
// Custom telemetry tracking
public class TelemetryService
{
    private readonly TelemetryClient _telemetryClient;
    
    public void TrackCustomerLookup(string customerId, TimeSpan duration, bool success)
    {
        _telemetryClient.TrackEvent("CustomerLookup", new Dictionary<string, string>
        {
            ["CustomerId"] = customerId,
            ["Success"] = success.ToString()
        }, new Dictionary<string, double>
        {
            ["Duration"] = duration.TotalMilliseconds
        });
    }
}
```

### Health Checks
```csharp
// Custom health check for Gold layer connectivity
public class GoldLayerHealthCheck : IHealthCheck
{
    private readonly ICustomerRepository _customerRepository;
    
    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            await _customerRepository.GetHealthCheckAsync();
            return HealthCheckResult.Healthy("Gold layer accessible");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Gold layer unavailable", ex);
        }
    }
}
```