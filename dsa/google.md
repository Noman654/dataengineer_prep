# Google Data Engineering Interview Case Study

This document details an authentic Google Data Engineering technical screen: **3 problems in 45 minutes.** 

No dynamic programming, no binary trees, and no graph traversals. Instead, the screen tests core pipeline patterns: **stream windowing, memory-optimized reconciliation, and sessionization.**

Below is the original interview experience followed by a comprehensive, senior-level guide with clean Python implementations, distributed scaling architectures, and complexity analyses.

---

## The Original Experience

> **My toughest DE coding round wasn't at Microsoft. It was at Google.**
>
> 45 minutes. One screen. Three problems. Zero theory questions.
>
> Here's exactly what they asked — and what I'd grind today if I had to do it over 👇
>
> *   📍 **Problem 1 (easy warm-up, 8 min):** *"Given a stream of events, find the top-K most active users in the last 1 hour."*
>     *   👉 **Pattern:** Heap + sliding window. Not on any "Top 50 LeetCode" list, but asked in 1 of every 2 DE rounds.
> *   📍 **Problem 2 (medium, 18 min):** *"Two large lists of order IDs from two systems. Find orders missing in one, duplicates in the other. Optimise for memory."*
>     *   👉 **Pattern:** HashMap + set operations + space-time trade-off. This is reconciliation logic disguised as DSA. Pure DE thinking.
> *   📍 **Problem 3 (hard, 18 min):** *"Group user events into sessions, where a session ends after 30 minutes of inactivity. Compute revenue per session."*
>     *   👉 **Pattern:** Sorting + interval merging + windowing. Same logic you write in production. Different syntax.
>
> **Notice what's missing?**
> ❌ No trees | ❌ No DP | ❌ No graphs | ❌ No "reverse a linked list"
>
> DE rounds test the patterns you'd use to build a real pipeline. But every DSA prep platform out there teaches you the SDE syllabus.

---

## Problem 1: Stream Top-K Active Users

> *"Given a stream of events, find the top-K most active users in the last 1 hour."*

### 1. In-Memory Python Implementation
In an interview, you should model this as a stateful processor that can ingest continuous events (e.g., `(user_id, timestamp)`) and efficiently serve the top-K active users at any given instant.

```python
from collections import defaultdict, deque
import heapq
from typing import List, Tuple

class ActiveUsersTracker:
    def __init__(self, window_seconds: int = 3600):
        self.window_seconds = window_seconds
        # Stores events as (timestamp, user_id) in chronological order
        self.events_queue = deque()
        # In-memory frequency lookup: {user_id: event_count}
        self.user_counts = defaultdict(int)

    def record_event(self, user_id: str, timestamp: int) -> None:
        """Ingests a new event and slides the window to evict expired events."""
        # 1. Record the new event
        self.events_queue.append((timestamp, user_id))
        self.user_counts[user_id] += 1
        
        # 2. Evict events older than (timestamp - window_seconds)
        cutoff_time = timestamp - self.window_seconds
        self._slide_window(cutoff_time)

    def _slide_window(self, cutoff_time: int) -> None:
        """Evicts expired events from the queue and decrements their counts."""
        while self.events_queue and self.events_queue[0][0] < cutoff_time:
            expired_time, expired_user = self.events_queue.popleft()
            self.user_counts[expired_user] -= 1
            if self.user_counts[expired_user] == 0:
                del self.user_counts[expired_user]

    def get_top_k(self, k: int, current_timestamp: int) -> List[Tuple[str, int]]:
        """Returns the top-K active users within the current window."""
        # Slide window to current time in case no new events arrived but time advanced
        self._slide_window(current_timestamp - self.window_seconds)

        # Min-Heap of size K to keep the largest elements
        # Heap elements are stored as (count, user_id)
        heap = []
        for user_id, count in self.user_counts.items():
            if len(heap) < k:
                heapq.heappush(heap, (count, user_id))
            else:
                # If current count is larger than the minimum count in our top-K, swap
                if count > heap[0][0]:
                    heapq.heapreplace(heap, (count, user_id))
                    
        # Sort descending (largest count first)
        return sorted([heapq.heappop(heap) for _ in range(len(heap))], reverse=True, key=lambda x: x[0])
```

### 2. Complexity Analysis
*   **Time Complexity**:
    *   `record_event`: $O(1)$ amortized. Each event is appended once and evicted once.
    *   `get_top_k`: $O(U \log K)$, where $U$ is the number of active users in the window.
*   **Space Complexity**: $O(E)$, where $E$ is the number of events within the 1-hour window.

### 3. Distributed Scaling & Production Architecture

> [!TIP]
> If $U$ (unique users) is in the billions, keeping `user_counts` in local Python memory will result in an Out-of-Memory (OOM) crash. In production pipelines, we choose one of three scale-out architectures:

#### Approach A: Apache Flink (Stateful Streaming)
In Apache Flink, we model this using an **Event-Time Sliding Window** (e.g., length 1 hour, slide 1 minute). 
*   **Mechanism**: The stream is partitioned by hashing `user_id` (`keyBy(user_id)`). Each partition maintains local user frequencies in state.
*   **Top-K Reduction**: To avoid sending all user counts to a single node, Flink applies a **local-to-global aggregation**:
    1.  *Local Step*: Each parallel worker runs a min-heap to extract its local top-K users.
    2.  *Global Step*: The local top-K lists (which are small, size $K$) are routed to a single downstream operator that merges them to output the global top-K.

#### Approach B: Redis Sorted Sets (ZSET)
If you require real-time dashboard updates with sub-millisecond lookups:
1.  Store active events in Redis using a Sorted Set where the **score is the timestamp** and the **value is `user_id:unique_event_id`**.
2.  Maintain another Sorted Set `active_users_counts` where the **score is the event count** and the **value is `user_id`**.
3.  When a new event arrives:
    *   Add to the event set: `ZADD events_zset <timestamp> user_id:event_id`
    *   Increment the user's active count: `ZINCRBY active_users_counts 1 user_id`
    *   Prune expired events: `ZREMRANGEBYSCORE events_zset -inf <current_time - 3600>`
    *   Decrement count for evicted users (handled via a script or consumer).
    *   Query Top-K instantly in $O(\log N + K)$ time: `ZREVRANGE active_users_counts 0 <K - 1> WITHSCORES`

#### Approach C: Count-Min Sketch + Min-Heap (Probabilistic)
For extreme scale (e.g., massive clickstreams on global websites):
*   Instead of exact hash tables, use a **Count-Min Sketch** (a 2D matrix of hash-based counters). This bounds the memory footprint to a fixed size (e.g., a few megabytes) regardless of user count, at the cost of a small, mathematically bounded false-positive error.
*   Maintain a parallel min-heap of size $K$ to keep track of the heavy hitters. When a user's count in the Count-Min Sketch exceeds the heap's minimum, update the heap.

---

## Problem 2: Memory-Optimized Order Reconciliation

> *"Two large lists of order IDs from two systems. Find orders missing in one, duplicates in the other. Optimise for memory."*

### 1. In-Memory Python Implementation
When both lists fit in RAM, standard set operations are fast and highly readable.

```python
from typing import Set, Tuple, List

def reconcile_in_memory(list_a: List[str], list_b: List[str]) -> Tuple[Set[str], Set[str], Set[str], Set[str]]:
    """
    Reconciles two lists of order IDs in memory.
    Returns:
        missing_in_b: Orders in A but not in B
        missing_in_a: Orders in B but not in A
        duplicates_in_a: Duplicate order IDs within A
        duplicates_in_b: Duplicate order IDs within B
    """
    seen_a = set()
    duplicates_in_a = set()
    for order in list_a:
        if order in seen_a:
            duplicates_in_a.add(order)
        seen_a.add(order)

    seen_b = set()
    duplicates_in_b = set()
    for order in list_b:
        if order in seen_b:
            duplicates_in_b.add(order)
        seen_b.add(order)

    missing_in_b = seen_a - seen_b
    missing_in_a = seen_b - seen_a

    return missing_in_b, missing_in_a, duplicates_in_a, duplicates_in_b
```

### 2. High-Scale Out-of-Core Implementation (Severe Memory Limits)
What if the lists contain **500 million order IDs** each (approx. 16GB total) but your container is limited to **50MB of RAM**? Loading these into sets will trigger an OOM crash.

> [!IMPORTANT]
> The classic data engineering pattern for memory-constrained joining or reconciliation is **MapReduce-Style Hash Partitioning (Sharding to Disk)**.

#### Sharded Hashing Algorithm:
1.  Read through `list_a` (streaming row-by-row/chunk-by-chunk). Hash each order ID and modulo it by $N$ (e.g., $N=100$) to distribute it into one of $N$ local disk partition files (`shard_a_0.txt` to `shard_a_99.txt`). During this streaming pass, we can detect duplicates in $A$ by writing duplicate IDs to a separate `duplicates_a.txt` file using a small local bloom filter or temporary memory buffer.
2.  Repeat the same process for `list_b`, sharding IDs into `shard_b_0.txt` to `shard_b_99.txt`.
3.  **The Magic**: Because we used the same hash function, any order ID present in both systems *must* reside in the matching partition number (e.g., ID `XYZ` will be in `shard_a_42.txt` and `shard_b_42.txt`). 
4.  Process one pair of shards at a time (`shard_a_i` and `shard_b_i`) in memory, writing the reconciliation discrepancies out to a final result file.

Here is how to write this in memory-efficient Python using generators and hash partitioning:

```python
import hashlib
import os
import shutil
from typing import Iterator

def get_shard_id(order_id: str, num_shards: int) -> int:
    """Computes a stable shard ID using MD5 hash modulo."""
    hasher = hashlib.md5(order_id.encode('utf-8'))
    return int(hasher.hexdigest(), 16) % num_shards

def shard_file_to_disk(input_stream: Iterator[str], prefix: str, num_shards: int, temp_dir: str) -> str:
    """Streams records from an input stream and shards them into N files on disk."""
    os.makedirs(temp_dir, exist_ok=True)
    
    # Initialize file handlers lazily to avoid holding 100 open file descriptors
    shard_paths = {i: os.path.join(temp_dir, f"{prefix}_{i}.txt") for i in range(num_shards)}
    
    # Clear any stale files
    for path in shard_paths.values():
        if os.path.exists(path):
            os.remove(path)
            
    # Track duplicates streamingly
    dup_file_path = os.path.join(temp_dir, f"{prefix}_duplicates.txt")
    if os.path.exists(dup_file_path):
        os.remove(dup_file_path)

    # Local buffer for sharding to reduce disk write I/O
    buffers = {i: [] for i in range(num_shards)}
    dup_buffer = []
    
    # Simple, highly compact local set for immediate duplicate detection
    # Note: In a production environment, replace this with a Bloom Filter to strictly limit memory
    seen_locally = set() 

    for order_id in map(str.strip, input_stream):
        if not order_id:
            continue
            
        # Check for duplicates
        if order_id in seen_locally:
            dup_buffer.append(order_id)
            if len(dup_buffer) >= 1000:
                with open(dup_file_path, "a") as f:
                    f.writelines(f"{x}\n" for x in dup_buffer)
                dup_buffer.clear()
        else:
            seen_locally.add(order_id)
            
        # Shard routing
        shard_idx = get_shard_id(order_id, num_shards)
        buffers[shard_idx].append(order_id)
        
        # Flush buffer to disk when it hits size limit
        if len(buffers[shard_idx]) >= 1000:
            with open(shard_paths[shard_idx], "a") as f:
                f.writelines(f"{x}\n" for x in buffers[shard_idx])
            buffers[shard_idx].clear()

    # Flush remaining buffers
    for idx, buf in buffers.items():
        if buf:
            with open(shard_paths[idx], "a") as f:
                f.writelines(f"{x}\n" for x in buf)
    if dup_buffer:
        with open(dup_file_path, "a") as f:
            f.writelines(f"{x}\n" for x in dup_buffer)

    return dup_file_path

def reconcile_large_streams(stream_a: Iterator[str], stream_b: Iterator[str], 
                            num_shards: int = 10, temp_dir: str = "./tmp_reconcile"):
    """
    Reconciles two massive streams of data with O(1) memory overhead.
    Shards records into matching bucket files, then reconciles bucket-by-bucket in RAM.
    """
    try:
        # Step 1: Shard stream A and B to disk
        dup_a_path = shard_file_to_disk(stream_a, "a", num_shards, temp_dir)
        dup_b_path = shard_file_to_disk(stream_b, "b", num_shards, temp_dir)
        
        missing_in_b_path = os.path.join(temp_dir, "missing_in_b.txt")
        missing_in_a_path = os.path.join(temp_dir, "missing_in_a.txt")
        
        # Clear stale outputs
        for path in (missing_in_b_path, missing_in_a_path):
            if os.path.exists(path):
                os.remove(path)

        # Step 2: Reconcile shard-by-shard
        for i in range(num_shards):
            shard_a_file = os.path.join(temp_dir, f"a_{i}.txt")
            shard_b_file = os.path.join(temp_dir, f"b_{i}.txt")
            
            # Load shard A IDs into a set
            set_a = set()
            if os.path.exists(shard_a_file):
                with open(shard_a_file, "r") as f:
                    set_a.update(line.strip() for line in f)
                    
            # Load shard B IDs into a set
            set_b = set()
            if os.path.exists(shard_b_file):
                with open(shard_b_file, "r") as f:
                    set_b.update(line.strip() for line in f)

            # Write differences streamingly
            missing_b = set_a - set_b
            if missing_b:
                with open(missing_in_b_path, "a") as f:
                    f.writelines(f"{x}\n" for x in missing_b)
                    
            missing_a = set_b - set_a
            if missing_a:
                with open(missing_in_a_path, "a") as f:
                    f.writelines(f"{x}\n" for x in missing_a)

        # Return result file paths for inspection
        return {
            "missing_in_b": missing_in_b_path,
            "missing_in_a": missing_in_a_path,
            "duplicates_in_a": dup_a_path,
            "duplicates_in_b": dup_b_path
        }
    finally:
        pass # Keep files for validation, clean up in walkthrough
```

### 3. Alternative Scaling Strategies

#### Approach A: Bloom Filters (Probabilistic Matching)
If a tiny percentage of misidentified records is acceptable (e.g., checking if an order might be missing for logging purposes):
*   Build a **Bloom Filter** for List A using a bit array. As List A streams past, map each ID to the bit array using multiple hashes.
*   Stream through List B: test each ID against the Bloom Filter. 
*   **Result**: If the filter says "not present," it is **100% guaranteed** to be missing in A (no false negatives). If the filter says "present," there is a configurable $1\%$ chance it is a false positive (it's actually missing, but hashes collided).
*   **Memory Efficiency**: Reconciling 100 million items requires under 120MB of RAM, compared to 4GB+ for standard sets.

#### Approach B: External Sort-Merge
1.  Perform an **External Merge Sort** on both files to sort them on disk. This splits files into chunks, sorts each chunk in memory, writes them to disk, and merges them streamingly.
2.  Open two file iterators (one for sorted file A, one for sorted file B) and use **two pointers** to scan both files synchronously. Since both files are sorted, you can identify missing elements and duplicates in a single linear scan using $O(1)$ memory.

---

## Problem 3: User Event Sessionization

> *"Group user events into sessions, where a session ends after 30 minutes of inactivity. Compute revenue per session."*

### 1. In-Memory Generator-Based Implementation
In data engineering, streaming and processing data using **Generators** represents high-quality Python coding. Instead of buffering all events for all users in memory, we assume events are pre-sorted by `user_id` and `timestamp` (the standard database or file order), and we yield sessions on-the-fly.

```python
from dataclasses import dataclass
from typing import Iterator, Tuple

@dataclass
class UserEvent:
    user_id: str
    timestamp: int  # Unix timestamp in seconds
    revenue: float

@dataclass
class Session:
    user_id: str
    session_id: str
    event_count: int
    total_revenue: float
    start_time: int
    end_time: int

def sessionize_events(sorted_events: Iterator[UserEvent], session_gap_seconds: int = 1800) -> Iterator[Session]:
    """
    Sessionizes a pre-sorted (by user_id, timestamp) stream of events.
    Uses O(1) auxiliary memory by yielding sessions as they close.
    """
    current_user = None
    last_timestamp = None
    session_count = 0
    
    # Active session state accumulator
    session_revenue = 0.0
    session_event_count = 0
    session_start_time = None

    for event in sorted_events:
        # Case 1: New user detected OR session inactivity threshold reached
        if (current_user is not None and event.user_id != current_user) or \
           (last_timestamp is not None and event.timestamp - last_timestamp > session_gap_seconds):
            
            # Yield the completed session
            yield Session(
                user_id=current_user,
                session_id=f"{current_user}_s{session_count}",
                event_count=session_event_count,
                total_revenue=round(session_revenue, 2),
                start_time=session_start_time,
                end_time=last_timestamp
            )
            
            # Reset session state
            if event.user_id != current_user:
                session_count = 0  # Reset session counter for new user
            else:
                session_count += 1
                
        # Initialize or accumulate active session
        if current_user is None or event.user_id != current_user or \
           (last_timestamp is not None and event.timestamp - last_timestamp > session_gap_seconds):
            session_start_time = event.timestamp
            session_revenue = 0.0
            session_event_count = 0

        current_user = event.user_id
        last_timestamp = event.timestamp
        session_revenue += event.revenue
        session_event_count += 1

    # Yield the very last active session once the stream terminates
    if current_user is not None:
        yield Session(
            user_id=current_user,
            session_id=f"{current_user}_s{session_count}",
            event_count=session_event_count,
            total_revenue=round(session_revenue, 2),
            start_time=session_start_time,
            end_time=last_timestamp
        )
```

### 2. Distributed Scale Implementation: SQL/PySpark (Gaps and Islands)
If the clickstream is stored in a data warehouse (like Snowflake or BigQuery) or a data lake (processed via PySpark), you implement this using the **Gaps and Islands** SQL pattern.

#### PySpark / SQL Implementation
```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.window import Window

def spark_sessionize(spark: SparkSession, input_df_path: str):
    """
    Demonstrates standard production-scale sessionization in PySpark.
    """
    df = spark.read.parquet(input_df_path)
    
    # 1. Define window partitioned by user and ordered by timestamp
    user_window = Window.partitionBy("user_id").orderBy("timestamp")
    
    # 2. Find the timestamp of the previous event for each user
    df_with_lag = df.withColumn("prev_timestamp", F.lag("timestamp").over(user_window))
    
    # 3. Detect session boundaries: 1 if gap > 30 mins (1800s) or first event (NULL), else 0
    df_with_boundary = df_with_lag.withColumn(
        "is_new_session",
        F.when(
            F.col("prev_timestamp").isNull() | 
            ((F.col("timestamp") - F.col("prev_timestamp")) > 1800),
            1
        ).otherwise(0)
    )
    
    # 4. Create Session ID by executing a cumulative sum of the boundary markers
    session_id_window = Window.partitionBy("user_id").orderBy("timestamp").rowsBetween(Window.unboundedPreceding, Window.currentRow)
    df_with_session = df_with_boundary.withColumn("session_idx", F.sum("is_new_session").over(session_id_window))
    df_with_session_id = df_with_session.withColumn("session_id", F.concat(F.col("user_id"), F.lit("_s"), F.col("session_idx")))
    
    # 5. Group by User + Session ID and aggregate revenue
    session_summary = df_with_session_id.groupBy("user_id", "session_id").agg(
        F.count("timestamp").alias("event_count"),
        F.sum("revenue").alias("total_revenue"),
        F.min("timestamp").alias("start_time"),
        F.max("timestamp").alias("end_time")
    )
    
    return session_summary
```

---

## Technical Summary Matrix

| Metric | Problem 1: Top-K Users | Problem 2: Reconcile IDs | Problem 3: Sessionization |
|---|---|---|---|
| **Core Pattern** | Heap + Sliding Window | HashMap / Disk Sharding | Sorting + Stateful Scan |
| **Optimal Time Complexity** | $O(E \log K)$ | $O(A + B)$ | $O(N \log N)$ (dominated by sort) |
| **Optimal Space Complexity** | $O(E)$ | $O(1)$ auxiliary (via sharding) | $O(1)$ auxiliary (via stream generators) |
| **Distributed Engine** | Apache Flink | Apache Spark Join (Sort-Merge) | PySpark Window / Lag Functions |
| **Key Failure Mode** | Memory OOM from unique users | Memory OOM from massive set union | Data Skew (a single user with millions of events) |
