# Project Context

## Background
The system is transitioning from a Legacy backend (MySGI Classic) to a new Drive system. During this transition period, we need intelligent routing to determine which backend should handle each registration-related request.

## Technical Implementation Details

### Core Logic
- **Primary Decision Factor**: The `isDriveCompatible` flag on a registration
  - `true` = typically route to Drive
  - `false` = route to Legacy system
- **Edge Cases**: Even when `isDriveCompatible = true`, some requests still go to Legacy (e.g., cancellations for Legacy-owned registrations)
- **Source of Truth**: Drive is the authoritative source for registration data

### Registration ID Handling
- **Drive Registrations**: Have standard GUIDs as registration IDs
- **Legacy Registrations**: 
  - Return empty GUID from Drive API
  - Have a `legacyRegistrationId` field (long type)
  - We encode the long as a GUID for consistency
  - Detection logic must handle both cases:
    1. Direct GUID match for Drive registrations
    2. Decode GUID to long and match against legacyRegistrationId

### GUID/Long Conversion
- **Encoding**: `GuidConversion.ToGuid(long value)` - converts long to GUID
- **Decoding**: `GuidConversion.ToLong(Guid guid)` - converts GUID back to long
- **Validation**: `GuidExtensions.TryParseLong(this Guid guid, out long result)` - safely checks if GUID can be decoded
- **Pattern**: Legacy GUIDs end with "-0000-0000-0000-000000000000"

## Caching Strategy

### Two-Level Caching
1. **HTTP Client Cache** (5-10 seconds TTL)
   - Cache the `GetVehicleRegistrationsByCustomer` API calls
   - Short TTL because registration details change frequently
   - Reduces redundant API calls in quick succession

2. **Response Cache** (20 minutes TTL)
   - Cache the `isDriveCompatible` decision per registration
   - Longer TTL because Drive compatibility rarely changes
   - Key: Registration ID + Customer Number
   - Value: isDriveCompatible boolean

### Cache Implementation Requirements
- Start with in-memory caching
- Use abstraction layer to allow swapping providers
- Support for distributed caching (Azure Storage or Redis)
- Multiple service instances must share cache

## API Integration

### VehicleRegistrationApiClient
- Method: `GetVehicleRegistrationsByCustomer(string customerNumber, CancellationToken cancellationToken)`
- Returns: List of `VehicleRegistrationDto` objects
- Each DTO contains:
  - `RegistrationId` (GUID)
  - `LegacyRegistrationId` (long?)
  - `IsDriveCompatible` (bool)
  - `IsSamPolicy` (bool) - SAM policies have special handling

## New Implementation: RequestRoutingDetector

### Responsibilities
1. Accept registration ID and customer number
2. Fetch registrations for customer (with caching)
3. Find matching registration by:
   - Direct GUID match, OR
   - Decoded GUID to legacy ID match
4. Return `isDriveCompatible` flag (with caching)
5. Handle errors gracefully with fallback logic

### Method Signature
```csharp
public interface IRequestRoutingDetector
{
    Task<bool?> IsRegistrationDriveCompatible(
        string customerNumber, 
        Guid registrationId, 
        CancellationToken cancellationToken);
}
```

## Gradual Replacement Strategy
1. Create new `IRequestRoutingDetector` and implementation
2. Register both old and new services in DI container
3. Use feature flag to control which service is used
4. Monitor metrics to compare performance
5. Gradually migrate endpoints to new service
6. Remove old service once fully migrated