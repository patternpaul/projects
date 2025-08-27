# Request Routing Detector Between Drive and Legacy

## Project Overview
- **Date**: 2025-08-27
- **Purpose**: Implement a detector service to intelligently route requests between Drive (new system) and Legacy systems
- **Context**: Creating a routing mechanism to determine which backend system should handle specific requests based on various criteria

## Problem Statement
Need to create a service that can:
- Detect whether a request should be routed to Drive or Legacy system
- Provide clear routing decisions based on configurable rules
- Support gradual migration from Legacy to Drive system
- Enable A/B testing and phased rollouts

## Solution Approach
- Create an interface `ILegacySystemDetector` for routing detection
- Implement `LegacySystemDetector` with configurable routing rules
- Support multiple detection strategies (feature flags, user segments, request attributes)
- Provide clear logging and metrics for routing decisions

## Implementation Details
- **Language**: C#/.NET
- **New Service Name**: RequestRoutingDetector
- **Location**: AF.CustomerVehicleRegistrations.BFF/Services/Routing/
- **Pattern**: Repository pattern with caching decorator
- **Cache Library**: Microsoft.Extensions.Caching with abstraction for providers
- **Integration**: Injected via DI, replacing existing LegacySystemDetector

## Files to Create/Modify
- AF.CustomerVehicleRegistrations.BFF/Services/Routing/IRequestRoutingDetector.cs (new)
- AF.CustomerVehicleRegistrations.BFF/Services/Routing/RequestRoutingDetector.cs (new)
- AF.CustomerVehicleRegistrations.BFF/Services/Routing/Cache/IRegistrationCache.cs (new)
- AF.CustomerVehicleRegistrations.BFF/Services/Routing/Cache/InMemoryRegistrationCache.cs (new)
- AF.CustomerVehicleRegistrations.BFF/Services/Routing/Cache/DistributedRegistrationCache.cs (new)
- Update dependency injection in Program.cs
- Add unit tests in corresponding test project

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
- Project focuses on implementing ILegacySystemDetector interface
- Initial routing will be based on configuration and feature flags
- Tests will validate both Drive and Legacy routing paths