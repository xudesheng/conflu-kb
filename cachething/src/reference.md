# Reference

## API Documentation

This section details the available services and methods for interacting with Cache Thing, including how to add, retrieve, update, and remove cache entries. It also covers utility services for estimating entry size, purging the cache, and managing cache configuration, ensuring developers can efficiently leverage in-memory caching in their applications.

### Core Services

These are the primary operations for interacting with Cache Thing, allowing you to add, retrieve, and remove cache entries using InfoTables that match your configured DataShape.

#### GetEntry

- **Description**: Retrieves a cache entry by key(s)
- **Input**: InfoTable with Primary Key fields
- **Output**: Cached data or _null_ if not found

**Example**:

```javascript
// CreateInfoTableFromDataShape(infoTableName:STRING("InfoTable"), dataShapeName:STRING):INFOTABLE(FibonacciDataShape)
let values = Resources["InfoTableFunctions"].CreateInfoTableFromDataShape({
    infoTableName: "InfoTable",
    dataShapeName: me.GetDataShape()
});
// FibonacciDataShape entry object
let newEntry = {
    position: position, // LONG [Primary Key]
    fibonacci_sum: undefined // LONG
};
values.AddRow(newEntry);

// result: INFOTABLE dataShape: ""
let cache_result = me.GetEntry({
	values: values /* INFOTABLE */
});
```

#### PutEntry

- **Description**: Adds or updates a cache entry
- **Input**: InfoTable matching DataShape
- **Output**: Success status

**Example**:

```javascript
let result;
if(cache_result.rows.length==0){
    result = me.SumNaive({
        position: position /* LONG [Required] */
    });
    let cacheInput = Resources["InfoTableFunctions"].CreateInfoTableFromDataShape({
        infoTableName: "InfoTable",
        dataShapeName: me.GetDataShape()
    });
    let newEntry = {
        position: position, // LONG [Primary Key]
        fibonacci_sum: result // LONG
    };
    cacheInput.AddRow(newEntry);
    me.PutEntry({ // store the computed result into the cache
        values: cacheInput /* INFOTABLE [Required] */
    });
}else{
	result = cache_result.rows[0].fibonacci_sum;
}
```

#### DeleteEntry

- **Description**: Removes a cache entry
- **Input**: InfoTable with Primary Key fields
- **Output**: Success status

**Example**:

```javascript
let values = Resources["InfoTableFunctions"].CreateInfoTableFromDataShape({
    infoTableName: "InfoTable",
    dataShapeName: me.GetDataShape()
});
// FibonacciDataShape entry object
let newEntry = {
    position: position, // LONG [Primary Key]
    fibonacci_sum: undefined // LONG
};
values.AddRow(newEntry);

// result: INFOTABLE dataShape: ""
let cache_result = me.DeleteEntry({
	values: values /* INFOTABLE */
});

```

### Utility Services

These services provide additional functionality for managing and analyzing your cache, such as estimating entry size and purging all cache entries.

#### EstimateEntrySize

- **Description**: Estimates the memory size of a cache entry
- **Input**: InfoTable matching DataShape
- **Output**: Size in bytes

**Example**:

```javascript
// result: INFOTABLE dataShape: ""
let cache_result = me.GetEntryByKey({
	key: key /* STRING */
});

// CreateInfoTableFromDataShape(infoTableName:STRING("InfoTable"), dataShapeName:STRING):INFOTABLE(NameIdentifierCacheDatashape)
let values = Resources["InfoTableFunctions"].CreateInfoTableFromDataShape({
    infoTableName: "InfoTable",
    dataShapeName: me.GetDataShape()
});

// NameIdentifierCacheDatashape entry object
let newEntry = {
    modelNumber: key, // STRING [Primary Key]
    modelResult: cache_result // INFOTABLE {"dataShape":"ModelResultInfotableDatashape"}
};

values.AddRow(newEntry);

// result: LONG
let result = me.EstimateEntrySize({
	values: values /* INFOTABLE [Required] */
});
```

#### PurgeCache

- **Description**: Removes all entries from the cache
- **Input**: None
- **Output**: Success status

**Example**:

```javascript
var result = Things["MyCacheThing"].PurgeCache();
```

## Configuration Reference

This section explains the key configuration options for Cache Thing, such as setting the DataShape, defining expiration policies, and configuring cache size limits. It provides guidance on how to tailor cache behavior to your application's needs, including eviction and expiration strategies, to optimize performance and resource usage.

### Expiration Policies

Defines how and when cache entries are removed based on time or access patterns, helping you control data freshness and memory usage.

| Policy                   | Description |
|--------------------------|-------------|
| _NEVER_                | Entries are never automatically expired by the cache. Data remains available until explicitly removed or evicted due to size constraints. This maximizes cache longevity but may reduce data freshness, as stale data can persist indefinitely unless manually managed. |
| _TIME_SINCE_LAST_ACCESSED_ | Entries expire after a specified duration has elapsed since their last access (read or write). This policy maximizes cache longevity for frequently accessed data, as such entry reuse is prioritized which could lead to staleness if not updated elsewhere. Infrequently accessed data is removed, which can help maintain some level of freshness for less-used entries. |
| _TIME_SINCE_CREATED_   | Entries expire after a fixed duration from the time they were created or added to the cache, regardless of subsequent access. This ensures a predictable maximum lifetime for cached data, balancing cache longevity and freshness by enforcing regular refreshes. |
| _TIME_SINCE_LAST_MODIFIED_ | Entries expire after a specified duration has passed since their last modification (write or update). This policy helps ensure that only recently updated data remains in the cache, promoting freshness, but may reduce longevity for data that is rarely modified. |
| _TIME_SINCE_LAST_TOUCHED_  | Entries expire after a set duration since their last access or modification. This combines both access and update events, providing a balance between keeping frequently used or updated data available and removing stale data to maintain freshness. |

## Metrics Reference

The following table lists the available cache-related metrics for Cache Thing, along with their descriptions. In high-availability (HA) environments, these metrics are tracked per node.

| Metric Name                       | Description                                                                                 |
|-----------------------------------|---------------------------------------------------------------------------------------------|
| _thingworx_cache_hit_rate_          | Ratio of cache requests that resulted in an entry being found in the cache                  |
| _thingworx_cache_hits_              | Number of times cache lookup methods have returned a cached value                           |
| _thingworx_cache_request_count_     | Number of times cache lookup methods have returned either a cached or uncached value        |
| _thingworx_cache_miss_rate_         | Ratio of cache lookup methods that have returned an uncached (newly loaded) value or null   |
| _thingworx_cache_misses_            | Number of times cache lookup methods have returned an uncached (newly loaded) value or null |
| _thingworx_cache_eviction_count_    | Number of times an entry has been evicted                                                   |
| _thingworx_cache_average_load_penalty_ | Average time spent loading new values                                                    |
| _thingworx_cache_weighted_size_     | Approximate current size of the cache in bytes                                              |
| _thingworx_cache_max_weight_        | Maximum size of the cache in bytes before eviction                                          |
| _thingworx_cache_estimated_entry_count_ | Estimated number of entries in the cache                                                |
| _thingworx_cache_global_max_size_   | Global maximum configured size for all caches in bytes                                      |

## Troubleshooting Guide

This section highlights frequent challenges encountered when using Cache Thing, such as import errors, high miss or eviction rates, and performance bottlenecks. It also covers typical in-memory cache issues like memory limits, data expiration, and cache consistency, providing practical steps for diagnosis and resolution.

### Common Issues

#### Unable to Import or Save Cache Thing

The Global Maximum Cache Size is purposely very small to begin, so a fresh/default environment may need this increased before importing existing Cache Thing entities (especially many simultaneuously).  Remeber to check the Application Log for the associated ERROR log message.

A variant on the same issue could be temporarily setting a Cache Thing size very large and forgetting to reduce it, as this could limit the ability to import, create, or increase other Cache Things.

#### High Miss Rate

Higher than expected cache misses either occur because the cache entry has yet to be loaded, or it has expired or been evicted.

1. Check cache size configuration versus entry size & count
2. Verify expiration policy (fit to use case without expiring too soon)
3. Review key distribution (analyze input parameters)
4. Monitor memory usage and cache size

Review cache size and expiration settings allow for maintaining entries long enough to be re-used.  Evicted entries can be monitored using the metric thingworx_cache_eviction_count.

Log and analyze input parameters mapped to cache Primary Keys and ensure relevance to actual behavior as higher variability means less chance of re-use.

Temporarily increasing the TTL and monitoring the impact on hit ratio to determine if cached entries are being expired, or just  not present.

#### High Eviction Rate

Evicted cache entries are not good as they are otherwise still valid, but are being thrown out as there is not enough space for new entries.  Frequent evictions thus indicates that the cache is not performaning as it is designed and intended to.

Eviction occurs due to lack of space, so either the incoming entry is too large, the loaded entries are not expiring, or the cache size is not large enough.

1. Check cache maximum size
2. Review entry sizes (log or validate size)
3. Monitor global maximum size
4. Adjust expiration policy and settings (clean up unused/stale entries)

#### Performance Issues

Despite the huge advantages of caching, the additional complexity can make debugging performance issues more challenging.  Think about validating that the cache is behaving and being leveraged as expected and triaging the area which is manifesting the performance issue: accessing cached data, populating cache data, retrieving source data, or using application services which leverage the cache.  Understanding the problem space is paramount to finding an adapted solution.

1. Verify cache hit rate
2. Check memory usage
3. Review expiration settings
4. Analyze cache loading compared to misses (cache put logged when entry not present)
5. Monitor system resources
6. Identify performance problem area
7. Research and understand the problem
8. Establish resolution plan

#### Slow Cache Loading

Inherently slow or reliable data services could cause the illusion of performance issues when cached data is expired and needs to be reloaded.  Users expecting 100ms responses will rightly complain if loading takes >30 seconds.  Caching can avoid many performance issues, but developers should still expect to deliver a performant and reliable loading experience The importance of which will vary depending on frequency and latency of the reload.

1. Monitor cache loading service performance stability (access logs, utilization statistics)
2. Performance analysis & optimization of data service
3. Add WARN logger indicating slow data loading
4. Monitor and alert on metric ``thingworx_cache_average_load_penalty``

#### Large Cache Size with Few Cache Hits (limited re-use)

As with many things, size only gets you so far.  Efficient caching requires adapted and re-usable data with the relevant cache keys and this may require refactoring your application services (requests, queries, APIs).

If many entries are cached but re-use is low, then you may need to make your cached data more generic and broadly re-usable.  Think about reducing expensive or external system calls, which may mean caching larger chunks of data which are leveraged to find the more specific data when needed (see included reference example ``NameIdentifierCache``).

1. Analyze sepecifcs of GetEntryByKey (log cache get keys and result)
2. Identify which parameters/keys are and are not being re-used
3. Consider using a superset of the data and dropping some key(s)

#### Cache Access/Loading Is Not Occuring on the Expected Node

The cache loading and access services will happen on the node upon which they are executed.  This can lead to unexpected behaviour if cache loading is executed on one node and cache reads on another.

The solution design determines where and how tasks are executed across the cluster.  Use the Application and Script log fields for platform ID [P:] and thread [T:] to determine which parts of the process are running on which node, and which execution processing mechanism is the reason that it is running there.

| Executor | Description |
|----------|-------------|
| Tomcat Executor | Tomcat executor thread pool, executing mainly REST API calls |
| WSExecution Processor | WebSocket execution processor handling WS traffic and remote service executions |
| TWEvent Processor | ThingWorx event processing handles executing subscriptions |
| Akka Actor Ordered Event Processor | Distributes, orders, and maintains subscription state for ordered subscriptions, data ordering, multi-event subscriptions. |
| Async Service Threads | Seperate threads created for unbounded async service execution (contains the Service name). |

## Known Issues

### ThingWorx 10.0.0

- Read Through mode not available
- Limited monitoring capabilities
- No automatic loading support
- Metrics label name is 'platform' instead of the standard 'platformid'

### ThingWorx 10.0.1

## Release Notes

### ThingWorx 10.0.0

- Initial release of Cache Thing
- Basic caching capabilities
- Manual cache management
- Core API support

### ThingWorx 10.0.1

- Read Through mode support
- Enhanced monitoring
- Automatic loading support
- Improved concurrent access handling
- Better memory management
