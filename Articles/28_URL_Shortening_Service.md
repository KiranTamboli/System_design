# System Design Briefing: URL Shortening Service

## Executive Summary

The following document outlines the technical requirements, high-level design, and deep-dive considerations for building a URL shortening service (similar to TinyURL). The system must handle massive scale—specifically 100 million new URLs per day—and provide high availability, scalability, and fault tolerance.

The core technical challenge involves mapping long URLs to unique, short aliases using a hash function. Two primary methods for this are identified: **Hash with Collision Resolution** and **Base 62 Conversion**. While the former relies on standard hashing algorithms (like CRC32) and iterative collision checks, the latter utilizes a unique ID generator to create a non-colliding Base 62 representation. For high-performance redirection, the system prioritizes caching and offers a trade-off between server load (301 redirects) and data analytics (302 redirects). Over a 10-year span, the system is projected to store 365 billion records, requiring approximately 36.5 TB of storage.

## 1\. Design Scope and Requirements

To ensure a well-crafted system, the design is established based on specific traffic volumes and functional constraints.

### Functional Requirements

* **URL Shortening:** Transform a long URL into a much shorter alias.  
* **URL Redirecting:** Forward users from the short alias to the original long URL.  
* **Immutability:** For simplicity, shortened URLs cannot be deleted or updated.  
* **Attributes:** The shortened URL must be as short as possible and consist of alphanumeric characters (0-9, a-z, A-Z).

### Capacity Estimation (Back-of-the-Envelope)

* **Traffic Volume:** 100 million URLs generated per day.  
* **Write Throughput:** \~1,160 writes per second.  
* **Read Throughput:** Assuming a 10:1 read-to-write ratio, the system must handle \~11,600 reads per second.  
* **Long-term Storage:** Supporting 365 billion records over 10 years.  
* **Storage Size:** Assuming an average URL length of 100 bytes, the total storage requirement is approximately 36.5 TB.

## 2\. API Design and Redirection Mechanics

The service utilizes a REST-style API to facilitate communication between clients and servers.

### API Endpoints

* **POST /v1/data/shorten**: Receives a longUrl parameter and returns a shortened URL.  
* **GET /v1/{shortUrl}**: Receives a short URL and initiates a redirection to the original long URL.

### Redirection Types: 301 vs. 302

| Redirect Type | Description | Browser Behavior | Use Case |
| :---- | :---- | :---- | :---- |
| **301 Redirect** | "Permanently" moved. | Caches the response; subsequent requests bypass the shortener. | Best for reducing server load. |
| **302 Redirect** | "Temporarily" moved. | Does not cache; every request is sent to the shortener first. | Best for tracking click rates and sources. |

## 3\. Data Modeling and Storage

While hash tables are intuitive for storing pairs, they are impractical for real-world systems due to limited and expensive memory.

* **Relational Database:** A more feasible approach is storing mappings in a relational database.  
* **Schema:** At a minimum, the table requires three columns: id (primary key), shortURL, and longURL.

## 4\. Hash Function Deep Dive

The system must map a long URL to a hashValue of a specific length. Based on the requirement to support 365 billion URLs, the hashValue must be **7 characters long** (62^7 ≈ 3.5 trillion, which exceeds the required 365 billion).

### Strategy A: Hash \+ Collision Resolution

This method uses standard hash functions (CRC32, MD5, or SHA-1).

* **Process:** The long URL is hashed, and the first 7 characters are used.  
* **Collision Handling:** If the 7-character hash already exists in the database, a predefined string is appended to the long URL and re-hashed until a unique value is found.  
* **Optimization:** A **Bloom filter** can be used to efficiently check if a shortURL exists without querying the database every time.

### Strategy B: Base 62 Conversion

This method converts a unique numeric ID (base 10\) into its base 62 representation.

* **Logic:** Base 62 uses 62 characters: 0-9, a-z, A-Z.  
* **Example:** A numeric ID of 11157 converts to 2TX in base 62\.  
* **Prerequisite:** This requires a distributed unique ID generator to ensure no two URLs share the same ID.

## 5\. System Workflows

### The URL Shortening Flow

1. **1\. Input:** Client provides a longURL.  
2. **2\. Lookup:** System checks if the longURL already exists in the database.  
3. **3\. Generation:** If not found, a unique ID is generated.  
4. **4\. Conversion:** The ID is converted to a shortURL using Base 62\.  
5. **5\. Persistence:** The ID, shortURL, and longURL are saved to the database.

### The URL Redirecting Flow

1. **1\. Request:** User clicks a short URL.  
2. **2\. Load Balancing:** The load balancer forwards the request to a web server.  
3. **3\. Cache Check:** The server checks if the shortURL is in the cache.  
4. **4\. Database Fetch:** If not in cache, the server fetches the mapping from the database.  
5. **5\. Redirect:** The longURL is returned to the user via a 301 or 302 redirect.

## 6\. Advanced Scalability and Security

To maintain a robust production system, several additional layers are required:

* **Rate Limiter:** Protects the system from malicious users by filtering based on IP addresses.  
* **Web Tier Scaling:** Web servers are stateless and can be scaled horizontally.  
* **Database Scaling:** Replication and sharding manage the 36.5 TB of data.  
* **Analytics Integration:** Tracking click frequency, timing, and user demographics.  
* **Reliability:** Ensuring high availability and consistency at scale.

