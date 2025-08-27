# Request Routing Detector Between Drive and Legacy

## Project Overview
- **Date**: 2025-08-27
- **Purpose**: Implement a detector service to intelligently route requests between Drive (new system) and Legacy systems
- **Context**: Creating a routing mechanism to determine which backend system should handle specific requests based on various criteria

## Problem Statement

The existing LegacySystemDetector has an incomplete implementation at line 22 where it attempts to determine if a registration is Drive-compatible but doesn't return the result. Need to create a proper routing detector that:

- Determines if a registration should be handled by Drive or Legacy system based on isDriveCompatible flag
- Handles both regular GUIDs (Drive) and encoded long IDs (Legacy)
- Implements proper caching to reduce API calls
- Provides clear routing decisions with proper error handling

## Solution Approach

- Create new `IRequestRoutingDetector` interface to replace incomplete implementation
- Implement proper GUID/Long ID conversion logic for legacy registrations
- Use two-level caching strategy:
  - HTTP response cache (5-10s TTL) for API responses
  - Routing decision cache (20min TTL) for isDriveCompatible results
- Support distributed cache backends (Redis, Azure Storage)

## Implementation Details

### Technical Context
- **Language**: C#/.NET
- **New Service Name**: RequestRoutingDetector
- **Location**: AF.CustomerVehicleRegistrations.BFF/Services/RequestRouting/
- **Pattern**: Service with injected cache abstractions
- **Cache Strategy**: Two-level caching with different TTLs
- **Integration**: Gradual replacement of LegacySystemDetector via feature flags

### Key Technical Details
- Registration IDs can be either regular GUIDs (Drive) or encoded longs (Legacy)
- Legacy IDs are detected using the existing `TryParseLong` extension method
- The `TryParseLong` method checks if GUID ends with "-0000-0000-0000-000000000000" and can convert to long
- Must check both RegistrationId match AND decoded legacy ID match
- Drive is source of truth for all registration data
- `isDriveCompatible` flag determines routing in most cases

## Files Created/To Create

### Created (2025-08-27)
- ✅ AF.CustomerVehicleRegistrations.BFF/Services/RequestRouting/IRequestRoutingDetector.cs
- ✅ AF.CustomerVehicleRegistrations.BFF/Services/RequestRouting/IRegistrationCache.cs
- ✅ AF.CustomerVehicleRegistrations.BFF/Services/RequestRouting/IDistributedCacheProvider.cs
- ✅ AF.CustomerVehicleRegistrations.BFF/Services/RequestRouting/RequestRoutingDetector.cs (Core implementation - no cache)
- ✅ Updated Program.cs with service registration

### To Create (In Order)
1. **Core Implementation (COMPLETED - No Cache)**
   - ✅ RequestRoutingDetector.cs with direct API calls
   - ✅ Dependency injection registration
   
2. **Unit Tests**
   - Test core logic thoroughly before adding complexity
   - Ensure all edge cases work
   
3. **Cache Implementation (LAST)**
   - AF.CustomerVehicleRegistrations.BFF/Services/RequestRouting/InMemoryRegistrationCache.cs
   - AF.CustomerVehicleRegistrations.BFF/Services/RequestRouting/DistributedRegistrationCache.cs
   - Cache decorator/wrapper around core implementation
   
4. **Dependency Injection**
   - Update Program.cs after core implementation is tested

## Testing Strategy
- Unit tests for each detection strategy
- Integration tests for routing decisions
- Performance tests for detection overhead
- A/B testing capability validation

## Known Limitations
- Initial implementation will support basic routing rules
- Configuration hot-reload may be added in future iterations
- Metrics collection to be enhanced based on operational needs

## Future Enhancements
- Machine learning-based routing decisions
- Dynamic configuration updates without restarts
- Advanced routing rules based on historical performance
- Circuit breaker patterns for system failures

## References
- Existing BFF structure in AF.CustomerVehicleRegistrations.BFF
- Drive API documentation
- Legacy system interfaces

## Session Recovery

Key information for resuming work:
- Main issue: LegacySystemDetector has incomplete implementation on line 22
- Registration IDs can be regular GUIDs or encoded longs
- Need to check both RegistrationId and decoded legacy ID

## Design Completed (2025-08-27)

### Interfaces Created

1. **IRequestRoutingDetector** (`/Services/RequestRouting/IRequestRoutingDetector.cs`)
   - Main service interface for routing detection
   - Single public method:
     - `IsRegistrationDriveCompatible()` - Determines Drive compatibility
   - Note: Other methods (GetVehicleRegistration, IsLegacyRegistrationFormat) are implementation details, not part of public API

2. **IRegistrationCache** (`/Services/RequestRouting/IRegistrationCache.cs`)
   - Cache abstraction for two-level caching
   - HTTP response cache (10s TTL)
   - Routing decision cache (20min TTL)
   - Includes CacheConfiguration class

3. **IDistributedCacheProvider** (`/Services/RequestRouting/IDistributedCacheProvider.cs`)
   - Abstraction for distributed cache backends
   - Support for Redis, Azure Storage, etc.
   - Includes CacheKeyBuilder for consistent keys

### Design Decisions

1. **Separation of Concerns**
   - New RequestRouting namespace separate from Legacy
   - Single public method interface - everything else is implementation detail
   - Cache abstractions created but will be implemented LAST

2. **Implementation Strategy (User Direction)**
   - Core functionality FIRST without any caching
   - Direct API calls in initial implementation
   - Cache layer added only after core logic is proven working
   - Unit tests before optimization

3. **API Design**
   - Single method `IsRegistrationDriveCompatible` is the only public API
   - Internal helper methods for registration lookup and format checking
   - Consistent cancellation token support