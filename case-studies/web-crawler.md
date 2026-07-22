# Web Crawler

*Design a web crawler like Google Bot. Crawl billions of web pages, extract content, handle politeness/robots.txt, detect duplicates.*

## Problem Statement

Build a crawler to index the entire web. Start with 1M seed URLs, follow links, extract content, store in index. 10B+ pages crawled. Must respect robots.txt, avoid duplicates, handle spam.

## Scale Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Pages to crawl** | 50B pages on web | Massive |
| **Crawl rate** | 1M pages/sec | Feasible with distributed crawlers |
| **Storage** | 50B × 5KB avg | 250 PB |
| **Bandwidth** | 1M pages × 50KB = 50GB/s | Huge |

## Architecture

```
URL Frontier (Kafka):
  Priority queue of URLs to crawl
  
Crawler Bots (distributed):
  Fetch URL
  Parse content
  Extract links
  Detect duplicates
  Re-queue new links
  
Content Store (distributed file system):
  Store raw HTML/content
  
Inverted Index:
  Index content for search
```

## Challenges & Solutions

### Duplicate Detection

**Problem**: Crawl same URL twice, waste bandwidth.

**Solution**: Bloom filter on seen URLs.
- Bloom filter: 50B URLs × 10 bits = 50GB memory
- False positive rate: ~0.1% (acceptable)

### Politeness

**Problem**: Crawl too aggressively → block crawler.

**Solution**:
- Respect robots.txt
- 1 request per domain per second
- User-Agent: Identify as crawler
- Honor Crawl-Delay, Retry-After headers

### Spam Detection

**Problem**: Spam pages, doorway pages, duplicates.

**Solution**:
- Page rank (links from high-authority pages matter)
- Content similarity (hashing to detect near-duplicates)
- Manual review of top results

### Link Extraction

**Problem**: Extract URLs from HTML, follow them.

**Solution**:
- Parse HTML (BeautifulSoup-style parser)
- Extract href attributes
- Normalize URLs (canonical URLs to avoid duplicates)
- Re-queue to URL Frontier

## Bottlenecks & Scaling

**Bottleneck**: Network bandwidth (50GB/s hard to sustain).

**Solution**:
- Distributed crawling across multiple data centers
- Compress pages (gzip during transfer)
- Prioritize high-value sites (PageRank-based)

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Bloom filter** | Set | Bloom filter uses 1/100th memory |
| **Politeness** | Aggressive crawling | Respect domain's server |
| **Distributed crawling** | Single crawler | Parallelism required for scale |

---

**Status**: ✅ Complete. Shows distributed crawling, deduplication, politeness.
