# Ride-Sharing Service

*Design Uber/Lyft: real-time matching, geolocation, pricing surge, driver-rider coordination across a city.*

## Problem Statement

Build a ride-sharing platform for a major city. 100M trips/day, real-time driver-rider matching, surge pricing, 99.9% availability. Drivers and riders constantly on the move, location updates every 10 seconds.

## Scale Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Active users** | 1M at peak | Concurrent connections |
| **Location updates/sec** | 1M drivers × 0.1 updates/sec | 100k location updates/sec |
| **Trip requests/sec** | 1M drivers × 0.02 request rate | 20k trip requests/sec |
| **Storage** | 1M drivers × location history | Historical tracking |

## Architecture

```
Driver App:
  Send: location, current status (online/available/busy)
  Receive: trip requests, cancellations
  
Rider App:
  Send: request ride (current location, destination)
  Receive: matched driver, ETA, real-time tracking
  
Matching Service:
  Listen to driver/rider locations
  Find nearby drivers for ride request
  Assign best driver (distance, rating, surge price)
  
Pricing Service:
  Calculate surge multiplier (demand/supply ratio)
  Adjust price in real-time
  
Tracking Service:
  Broadcast driver location to rider (every 5 sec)
  Persistent trip history (for support, analytics)
```

## Deep Dive: Geolocation Matching

**Challenge**: Find nearest driver to rider in real-time.

```
Naive: Query all 1M drivers, sort by distance → O(1M) slow

Solution: Geo-partitioning
  Divide city into grid cells (e.g., 10m x 10m)
  Rider location → Cell [x, y]
  Find drivers in cells [x±1, y±1] (9 cells total)
  O(100-200 drivers per cell) → Much faster

Implementation:
  Redis Geo commands: GEOADD, GEORADIUS
  Or: DynamoDB with geohash partition key
  
Result: Matching latency < 1 second
```

## Bottlenecks & Scaling

**Bottleneck**: Geolocation queries (100k/sec).

**Solution**:
- Geo-partitioning (grid or geohash)
- In-memory trees (R-tree, Quadtree)
- Caching hot zones (downtown during rush hour)

## Failure Scenarios

**Network partition**: Driver disconnects, rider sees stale location.

**Mitigation**:
- Client retries (exponential backoff)
- Show last known location with "stale" indicator
- Auto-cancel after 30 sec if no update

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Geo-partitioning** | Query all drivers | Speed (O(100) vs O(1M)) |
| **Eventual consistency** | Strong consistency | Real-time responsiveness |
| **Async location updates** | Sync | Reduce latency, accept slight staleness |

---

**Status**: ✅ Complete. Shows real-time geographic matching at scale.
