# Project Context

## Background
The system is transitioning from a Legacy backend to a new Drive system. During this transition period, we need intelligent routing to determine which backend should handle each request.

## Key Requirements

### Routing Criteria
- Feature flags for gradual rollout
- User segment identification (beta users, specific organizations)
- Request type classification (read vs write operations)
- System health checks (fallback to Legacy if Drive is unhealthy)
- Performance-based routing (route to faster system)

### Non-Functional Requirements
- Sub-millisecond detection overhead
- Clear audit trail of routing decisions
- Support for override mechanisms (force Legacy or Drive)
- Backward compatibility with existing code

## Integration Points

### Current Architecture
- BFF (Backend for Frontend) layer handles API requests
- Services are injected via dependency injection
- Controllers call services to process requests
- Both Legacy and Drive services exist in parallel

### Detection Flow
1. Request arrives at BFF controller
2. Controller calls ILegacySystemDetector
3. Detector evaluates routing rules
4. Returns routing decision (Drive or Legacy)
5. Controller calls appropriate service based on decision

## Configuration Strategy

### Feature Flags
- Use existing feature flag system
- Flags per operation type (e.g., "UseDriveForRenewals")
- Percentage-based rollout support

### User Segments
- Beta users list in configuration
- Organization-based routing rules
- VIP customer special handling

## Monitoring and Metrics

### Key Metrics to Track
- Routing decision distribution (% to Drive vs Legacy)
- Detection latency
- Override usage frequency
- Error rates per system
- Performance comparison between systems

### Logging Requirements
- Log every routing decision with context
- Include decision factors in logs
- Structured logging for analysis
- Correlation IDs for request tracing