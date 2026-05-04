# B.Tech Project Mid-Term Summary

**Title:** Reducing Cross-Node Network Calls in Distributed Payment Systems via Thread-Level and Server-Local Caching  
**Student:** Raman Kapoor | **Supervisor:** Dr. Rahul Kala | **Institution:** ABV-IIITM Gwalior  

---

## 1. Problem Statement
Stateless API servers process millions of financial transactions daily, with each request triggering **5-15 cross-node network calls** to a Redis cluster or primary database for configuration, routing, and risk data. This redundant data fetching creates significant Redis load amplification at peak times and drives up p95 and p99 tail latencies due to network variability. The core challenge is to safely cache this data to reduce network hops without violating the strict correctness and consistency guarantees required for payment processing.

## 2. Data Classification & Safety Boundaries
Analysis shows that while transactional state changes rapidly, configuration data (e.g., merchant configs, gateway routing tables, feature flags) remains globally scoped and mostly immutable over short time windows. 

To formalize what can be cached without financial risk, I defined **four Cache-Safety Boundary Criteria**:
1. **Immutability within TTL:** Data must not change over the cache TTL window.
2. **Request-Scope Independence:** Data depends only on global/slow-changing state, not request parameters.
3. **Semantic Idempotency of Stale Reads:** Even if briefly stale, the business outcome remains semantically correct.
4. **Deterministic Cache Key:** Keys are deterministically derivable without collision risk.

If a data class meets all 4 criteria, it is safe to cache.

## 3. Two-Tier In-Process Caching Architecture
To reduce network calls with **zero new infrastructure**, I designed an entirely in-process, two-tier caching architecture in Haskell:

* **Tier 1 (Thread-Level Cache):** A request-scoped cache (`IORef HashMap`) that deduplicates identical lookups within the same transaction request. It guarantees zero staleness since it is destroyed when the request completes.
* **Tier 2 (Server-Local Cache):** A node-scoped, concurrent cache (`TVar` with bounded size and LRU eviction) shared across all threads on a server. It uses short TTLs (30-60 seconds) to serve cross-request redundant lookups directly from local memory.

## 4. Key Results and Impact
The caching architecture was deployed in a staged rollout on the production platform handling ~68,000 cross-node operations per second.

* **Network Reduction:** Achieved a **54.8% combined cache hit rate**, eliminating ~2,800 network calls per second and reducing total cross-node calls at peak by **4.1%**.
* **Latency Improvements:** 
  * p95 API latency reduced by **11.2%** (98ms $\rightarrow$ 87ms).
  * p99 API latency reduced by **13.0%** (285ms $\rightarrow$ 248ms).
* **Redis Decompression:** Peak Redis p99 latency dropped by a massive **26.2%** due to the removed load, creating a virtuous cycle of faster API responses.
* **Overhead & Correctness:** The implementation required under 300 lines of code, adds less than 20MB of memory overhead per node, and recorded **zero cache-related correctness incidents or financial discrepancies**.

## 5. Conclusion
A lightweight, in-process caching strategy governed by formal safety guarantees effectively delivers measurable latency and scalability improvements in correctness-critical distributed systems, without the cost or operational burden of deploying external cache infrastructure.
