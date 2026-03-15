# URL Shortener System Design (TinyURL)

## 1. Problem Statement

Design a URL shortening service similar to **TinyURL** or **Bitly**.

Users should be able to submit a long URL and receive a shorter version. When the short URL is accessed, it should redirect to the original long URL.

Example:

Input:

```
https://www.google.com/search/system-design-interview
```

Output:

```
https://short.ly/ab12cd
```

When a user visits `https://short.ly/ab12cd`, they should be redirected to the original URL.

---

# 2. Functional Requirements

1. Users should be able to generate short URLs.
2. Short URLs should redirect to original URLs.
3. Users should be able to access shortened links quickly.
4. System should track number of clicks (optional).
5. Custom aliases (optional).

---

# 3. Non Functional Requirements

1. High availability (links must always work).
2. Low latency redirects.
3. Highly scalable.
4. Durable storage of URLs.
5. System should handle millions of URLs.

---

# 4. Capacity Estimation

Assume:

* 100M URLs created per month
* 1B redirects per month

Storage:

Average URL length = 500 bytes

Storage per month:

```
100M * 500 bytes = 50 GB
```

Yearly:

```
600 GB per year
```

This is manageable using distributed databases.

---

# 5. High Level Architecture

Components:

1. Client (Browser / Mobile)
2. Load Balancer
3. Application Server
4. Cache (Redis)
5. Database
6. URL Generator Service

Flow:

Create Short URL

Client → Load Balancer → Application Server → URL Generator → Database → Return Short URL

Redirect

Client → Load Balancer → Application Server → Cache → Database → Redirect

---

# 6. API Design

### Create Short URL

Endpoint

```
POST /shorten
```

Request

```
{
  "url": "https://example.com/very/long/url"
}
```

Response

```
{
  "shortUrl": "https://short.ly/abc123"
}
```

---

### Redirect URL

Endpoint

```
GET /{shortCode}
```

Example

```
GET /abc123
```

Response

HTTP 301 Redirect → Original URL

---

# 7. Database Design

Table: URLS

```
id (primary key)
short_code
long_url
created_at
expiry_date
click_count
```

Example:

| id | short_code | long_url   | created_at |
| -- | ---------- | ---------- | ---------- |
| 1  | abc123     | google.com | timestamp  |

Indexes:

```
INDEX(short_code)
```

This ensures fast lookups.

---

# 8. Short URL Generation Strategy

Possible approaches:

### Option 1 — Hashing

Use hash of URL.

Example:

```
hash(url) → first 6 characters
```

Problem:

Collisions possible.

---

### Option 2 — Base62 Encoding (Best)

Use ID and encode into Base62.

Example:

```
ID = 125
Base62 = cb
```

Base62 characters:

```
[a-z A-Z 0-9]
```

Example:

```
125 → cb
```

Advantages:

* Short URLs
* No collisions
* Deterministic

---

# 9. Caching

Use **Redis** to reduce database load.

Flow:

```
Request → Redis
       → Hit → Return URL
       → Miss → Query DB
```

Cache Example:

```
key: abc123
value: https://google.com
```

---

# 10. Scaling Strategy

### Horizontal Scaling

Multiple application servers behind load balancer.

```
          Load Balancer
        /      |      \
     Server1 Server2 Server3
```

---

### Database Scaling

Techniques:

1. Replication (Read replicas)
2. Sharding
3. Partitioning

Example:

Shard based on:

```
short_code hash
```

---

# 11. Bottlenecks

### Database overload

Solution:

Use caching (Redis).

---

### Hot URLs

Some URLs may receive huge traffic.

Solution:

Use CDN caching.

---

### Hash collisions

Solution:

Use Base62 encoding instead of hash.

---

# 12. Improvements

Future features:

1. Custom short URLs

Example:

```
short.ly/kushal
```

2. Expiry URLs

```
Expire after 30 days
```

3. Analytics dashboard

Track:

* Click count
* Location
* Device

4. QR code generation

---

# 13. Simple Java Example (Reference)

This is a simplified version of URL shortener logic.

```java
import java.util.HashMap;
import java.util.Map;

public class UrlShortener {

    private Map<String, String> map = new HashMap<>();
    private int counter = 1;

    private static final String BASE62 = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";

    public String encode(String longUrl) {
        String shortCode = toBase62(counter++);
        map.put(shortCode, longUrl);
        return shortCode;
    }

    public String decode(String shortCode) {
        return map.get(shortCode);
    }

    private String toBase62(int num) {
        StringBuilder sb = new StringBuilder();

        while (num > 0) {
            sb.append(BASE62.charAt(num % 62));
            num /= 62;
        }

        return sb.reverse().toString();
    }

    public static void main(String[] args) {
        UrlShortener service = new UrlShortener();

        String shortUrl = service.encode("https://google.com");
        System.out.println("Short URL: " + shortUrl);

        System.out.println("Original URL: " + service.decode(shortUrl));
    }
}
```

---

# 14. Final Architecture Summary

Technologies:

```
Load Balancer → Nginx
Backend → Java Spring Boot
Cache → Redis
Database → MySQL / DynamoDB
Message Queue → Kafka (analytics)
Storage → Distributed DB
```

---

# 15. Interview Answer Summary

Steps to explain in interview:

1. Clarify requirements
2. Estimate scale
3. Design APIs
4. Design database
5. Create high level architecture
6. Discuss scaling
7. Discuss bottlenecks
8. Suggest improvements
