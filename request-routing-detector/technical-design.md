# Technical Design - Request Routing Detector

## Architecture Overview

```
┌─────────────────┐
│   Controller    │
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│ IRequestRoutingDetector │ ◄── Interface
└─────────────────────────┘
         │
         ▼
┌─────────────────────────┐
│ RequestRoutingDetector  │ ◄── Implementation
└─────────────────────────┘
         │
         ├──────────────────┐
         ▼                  ▼
┌──────────────────┐  ┌─────────────────┐
│ IRegistrationCache│  │ IVehicleRegApi  │
└──────────────────┘  └─────────────────┘
```

## Interface Definitions

### IRequestRoutingDetector
```csharp
public interface IRequestRoutingDetector
{
    /// <summary>
    /// Determines if a registration should be routed to Drive system
    /// </summary>
    /// <param name="customerNumber">Customer identifier</param>
    /// <param name="registrationId">Registration GUID (could be encoded legacy ID)</param>
    /// <param name="cancellationToken">Cancellation token</param>
    /// <returns>true for Drive, false for Legacy, null if unable to determine</returns>
    Task<bool?> IsRegistrationDriveCompatible(
        string customerNumber, 
        Guid registrationId, 
        CancellationToken cancellationToken);
}
```

### IRegistrationCache
```csharp
public interface IRegistrationCache
{
    // HTTP Response Cache (5-10 seconds TTL)
    Task<List<VehicleRegistrationDto>?> GetRegistrationsAsync(
        string customerNumber, 
        CancellationToken cancellationToken);
    
    Task SetRegistrationsAsync(
        string customerNumber, 
        List<VehicleRegistrationDto> registrations,
        TimeSpan ttl,
        CancellationToken cancellationToken);
    
    // Routing Decision Cache (20 minutes TTL)
    Task<bool?> GetRoutingDecisionAsync(
        string cacheKey,
        CancellationToken cancellationToken);
    
    Task SetRoutingDecisionAsync(
        string cacheKey,
        bool isDriveCompatible,
        TimeSpan ttl,
        CancellationToken cancellationToken);
}
```

## Implementation Details

### RequestRoutingDetector Logic Flow

```csharp
public async Task<bool?> IsRegistrationDriveCompatible(
    string customerNumber, 
    Guid registrationId, 
    CancellationToken cancellationToken)
{
    // 1. Check routing decision cache
    var cacheKey = GenerateCacheKey(customerNumber, registrationId);
    var cachedDecision = await _cache.GetRoutingDecisionAsync(cacheKey, cancellationToken);
    if (cachedDecision.HasValue)
        return cachedDecision;
    
    // 2. Get registrations (with HTTP cache)
    var registrations = await GetRegistrationsWithCache(customerNumber, cancellationToken);
    if (registrations == null)
        return null; // Unable to determine
    
    // 3. Find matching registration
    var registration = FindMatchingRegistration(registrations, registrationId);
    if (registration == null)
        return null; // Registration not found
    
    // 4. Cache and return decision
    await _cache.SetRoutingDecisionAsync(
        cacheKey, 
        registration.IsDriveCompatible,
        TimeSpan.FromMinutes(20),
        cancellationToken);
    
    return registration.IsDriveCompatible;
}

private VehicleRegistrationDto? FindMatchingRegistration(
    List<VehicleRegistrationDto> registrations,
    Guid registrationId)
{
    // Direct GUID match
    var directMatch = registrations.FirstOrDefault(r => r.RegistrationId == registrationId);
    if (directMatch != null)
        return directMatch;
    
    // Try legacy ID match
    if (registrationId.TryParseLong(out var legacyId))
    {
        return registrations.FirstOrDefault(r => r.LegacyRegistrationId == legacyId);
    }
    
    return null;
}
```

## Caching Strategy

### Cache Keys
- **HTTP Cache Key**: `registrations:{customerNumber}`
- **Routing Cache Key**: `routing:{customerNumber}:{registrationId}`

### TTL Configuration
```json
{
  "CacheSettings": {
    "HttpResponseTtlSeconds": 10,
    "RoutingDecisionTtlMinutes": 20,
    "CacheProvider": "InMemory" // or "Redis" or "AzureStorage"
  }
}
```

## Dependency Injection Setup

```csharp
// In Program.cs or ServiceExtensions
services.AddSingleton<IRegistrationCache>(provider =>
{
    var config = provider.GetRequiredService<IConfiguration>();
    var cacheProvider = config["CacheSettings:CacheProvider"];
    
    return cacheProvider switch
    {
        "Redis" => new RedisRegistrationCache(/* redis config */),
        "AzureStorage" => new AzureStorageRegistrationCache(/* azure config */),
        _ => new InMemoryRegistrationCache()
    };
});

services.AddScoped<IRequestRoutingDetector, RequestRoutingDetector>();

// Feature flag for gradual rollout
services.AddScoped<ILegacySystemDetector>(provider =>
{
    var featureFlags = provider.GetRequiredService<IFeatureFlagService>();
    if (featureFlags.IsEnabled("UseNewRoutingDetector"))
    {
        return new RequestRoutingDetectorAdapter(
            provider.GetRequiredService<IRequestRoutingDetector>());
    }
    return provider.GetRequiredService<LegacySystemDetector>();
});
```

## Error Handling

### Fallback Strategy
1. If cache is unavailable → fetch from API directly
2. If API call fails → return null (let caller decide fallback)
3. If registration not found → return null
4. Log all errors with appropriate context

### Logging
```csharp
_logger.LogInformation(
    "Routing decision for registration {RegistrationId} of customer {CustomerNumber}: {Decision}",
    registrationId, customerNumber, isDriveCompatible ? "Drive" : "Legacy");
```

## Performance Considerations

### Expected Performance Metrics
- Cache hit latency: < 1ms
- Cache miss latency: < 100ms (API call)
- Memory usage per cached item: ~2KB
- Cache size limit: 10,000 items (configurable)

### Monitoring
- Track cache hit/miss ratio
- Monitor API call frequency
- Measure decision latency percentiles
- Alert on high error rates

## Testing Strategy

### Unit Tests
```csharp
[Test]
public async Task Should_Return_Drive_For_DriveCompatible_Registration()
[Test]
public async Task Should_Return_Legacy_For_NonDriveCompatible_Registration()
[Test]
public async Task Should_Match_By_Legacy_Id_When_Guid_Is_Encoded_Long()
[Test]
public async Task Should_Use_Cached_Decision_When_Available()
[Test]
public async Task Should_Handle_Api_Failure_Gracefully()
```

### Integration Tests
- Test with real cache providers
- Verify feature flag switching
- Test concurrent access patterns
- Validate cache expiration

## Migration Plan

### Phase 1: Parallel Run
- Deploy new service alongside old
- Log decisions from both
- Compare results for discrepancies

### Phase 2: Gradual Rollout
- 10% traffic → monitor for 1 day
- 50% traffic → monitor for 3 days
- 100% traffic → monitor for 1 week

### Phase 3: Cleanup
- Remove old LegacySystemDetector
- Remove feature flag
- Optimize cache settings based on metrics