# Video Streaming Platform

*Design a video streaming service like Netflix, YouTube, or Twitch. Billions of videos, adaptive bitrate streaming, global CDN, massive scale.*

## Problem Statement

Build a video platform for 500M users. 100M DAU watching videos, 50M hours watched/day, new content constantly uploaded. Videos range from 1 min to 4 hours. Users in 150+ countries. Must stream smoothly (buffer-free) at scale.

## Clarifying Questions & Requirements

### Functional Requirements
- Upload video (convert to multiple bitrates)
- Stream video (adaptive bitrate)
- Watch history, recommendations
- Search/browse content
- User ratings/comments

### Non-Functional Requirements
- **Scale**: 100M DAU, 50M hours/day
- **Latency**: Start playback < 2 seconds, smooth streaming
- **Quality**: Adaptive bitrate (1080p down to 360p based on bandwidth)
- **Availability**: 99.9% uptime globally
- **Consistency**: Eventual consistency OK

## Scale Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Concurrent viewers** | 100M DAU × 20% = 20M | 20M simultaneous streams |
| **Bandwidth** | 20M × 5 Mbps avg | 100 Tbps (!!) |
| **Storage** | 50M hours × 5 Mbps = 2.5PB/day | 900 PB/year |
| **Transcoding** | 50M hours × 10 bitrates = 500M hours | Massive compute |

**Key insight**: Bandwidth is the hard constraint. CDN is mandatory. Transcoding is offline batch job.

## Architecture

### Tier 1: Upload & Transcoding

```
User uploads video.mp4 (4GB)
  → S3 object storage
  → Kafka event: "video.uploaded"
  
Transcoding service subscribes:
  → Download from S3
  → Run ffmpeg: mp4 → {1080p, 720p, 480p, 360p, 144p}
  → Upload each bitrate to S3
  → Update metadata: "transcoding complete"
  
Total time: 1-4 hours (batch offline)
```

### Tier 2: Metadata & Indexing

```
Video metadata:
  - Title, description, thumbnails
  - Duration, bitrates available
  - Upload date, creator, views
  
Stored in:
  - Database (for queries)
  - Search index (Elasticsearch for full-text search)
  - Redis cache (hot videos)
```

### Tier 3: Streaming

```
Client requests: video_id=123, quality=720p
  → Look up metadata (cache hit for hot videos)
  → Look up video segments (HLS playlists)
  → Serve from CDN (99% of traffic)
  
Playback protocol: HLS or DASH
  - Split video into 10-second segments
  - Playlist: "segment1.m3u8, segment2.m3u8, ..."
  - Client requests segments as user watches
  - Client adapts bitrate based on bandwidth
```

### Tier 4: Analytics

```
Every 10 seconds:
  Send: {videoId, userId, quality, buffering, bandwidth}
  → Kafka
  → Analytics warehouse
  
Metrics:
  - Drop-off rate (at what timestamp do users leave?)
  - Quality distribution (how many 1080p vs 360p?)
  - Geographic latency (which regions are slow?)
```

## CDN Strategy

### Hierarchy

```
Tier 1: Origin (S3 + transcoded segments)
Tier 2: Origin Shield (intermediate cache, regional)
Tier 3: CDN PoPs (100+ global locations)
Tier 4: Client-side player cache

Request path:
  Client → Nearest PoP (cache hit 80%)
    → Miss: Origin Shield (cache hit 95%)
      → Miss: Origin S3 (99% cached in CDN anyway)

Result: 80% P99 latency < 10ms, 20% hits origin (100ms)
```

### Adaptive Bitrate (ABR)

Client-side algorithm:

```
Measure: bandwidth = bits_downloaded / time

If bandwidth > 8 Mbps: Request 1080p (4.5 Mbps)
If bandwidth 4-8 Mbps: Request 720p (2.5 Mbps)
If bandwidth < 4 Mbps: Request 480p (1.5 Mbps)

Buffer management:
  If buffer < 5s: Lower quality (avoid buffering)
  If buffer > 20s: Increase quality (good bandwidth available)
```

## Bottlenecks & Scaling

### Transcoding

**Problem**: 500M hours/day to transcode, ffmpeg is slow.

**Solution**:
- Distributed transcoding fleet (GPU acceleration)
- Priority queue: Live streams prioritized over archives
- Auto-scale (spot instances for batch jobs)

### Storage

**Problem**: 900PB/year, exabyte scale.

**Solution**:
- Tiered storage: Hot (recent), Warm (1 year), Cold (archive)
- Erasure coding (reduce replicas from 3x to 1.5x)
- Geo-distribution (replicate to multiple regions)

### Bandwidth

**Problem**: 100 Tbps is hard to achieve.

**Solution**:
- P2P between users in same region (peer caching)
- ISP caches (Vimeo, Netflix have these)
- Bitrate limiting for live (cap at 1080p) during peak

## Failure Scenarios

### Transcode Service Down

Videos can't be uploaded until fixed.

**Mitigation**:
- Queue uploads to Kafka (durable)
- Retry with backoff
- Fallback: Serve low-quality until transcoded

### CDN PoP Failure

Users in that region slow to stream.

**Mitigation**:
- CDN automatically redirects to next nearest PoP
- Users experience 200-500ms latency increase (noticeable but playable)

### Storage Failure

**Mitigation**:
- Erasure coding: Can tolerate 2 node failures
- Geo-replication: Video in 3 regions

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **HLS + DASH** | Single bitrate | Adaptive bitrate uses bandwidth efficiently |
| **Offline transcoding** | On-demand | Batch is cheaper (spot instances), on-demand is fast |
| **CDN + origin shield** | Direct origin | Reduces origin load 100x |
| **Segment-based streaming** | Progressive download | Segments allow client-side bitrate adaptation |

## Related Fundamentals

- [Content Delivery](../fundamentals/content-delivery-and-edge/) – CDN architecture
- [Storage](../fundamentals/storage-systems/) – Object storage, geo-replication
- [Batch Processing](../fundamentals/batch-and-stream-processing/) – Transcoding pipeline
- [Observability](../fundamentals/observability/) – Analytics on playback

---

**Status**: ✅ Complete. Shows scale architecture with CDN, transcoding, adaptive bitrate.
