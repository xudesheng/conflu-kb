# Introduction

## What is the Cache Thing

Cache Thing is a new feature in ThingWorx 10 that improves application performance through in-memory caching. It provides a simple interface for storing and retrieving key-value data, making it an essential tool for optimizing ThingWorx applications.

Cache Thing reduces load on external systems and improves response times by caching frequently accessed data. It works well with:

- Database queries from Database/SQL Things, DataTables, Streams, and ValueStreams
- Results from network calls and external API requests
- Computationally intensive calculations
- ContentLoaderFunction results
- Third-party service responses

Cache Things help reduce costs from external services while improving application performance. Each Cache Thing is created from the Cache Thing Template with its own configuration and usage.

## Challenges and Solutions

### Current Challenges

In modern ThingWorx applications, several performance challenges commonly arise:

1. **Database Overload**: Applications make expensive database queries repeatedly, especially for data that rarely changes. This increases database load and slows response times.

2. **Resource Inefficiency**: The same data gets queried multiple times, wasting system resources.

3. **User Experience Impact**: Slow response times cause session timeouts and disrupt operations, affecting user experience.

4. **Cascading Failures**: When external systems become slow or unavailable, it can cause outages in your application.

5. **Limited Control**: Without caching, developers cannot easily control data access patterns or optimize performance.

### Solution Overview

Cache Thing solves these problems by providing:

1. **Fast Data Access**: Stores and retrieves data quickly, reducing access times for frequently used data.

2. **Better Performance**: Speeds up slow processes by caching results from database queries, streams, and external APIs.

3. **Less Resource Usage**: Reduces system load by reusing results from expensive operations instead of repeating them.

4. **Better Scalability**: Using cached data reduces pressure on external systems and makes your application more reliable.

5. **Lower Costs**: Reduces costs from third-party services by caching responses and making fewer API calls.

## Key Terminologies

To use Cache Thing effectively, you should understand these concepts:

These key concepts are detailed in the [Basic Concepts](./basic_concepts.md) section.

## Version Availability

Cache Thing is available in two versions with different features:

### ThingWorx 10.0.0

The first version provides basic caching features:

- Basic Cache Thing functionality with manual cache management
- Cache Aside mode implementation
- Core API support for basic operations
- Essential monitoring features

### ThingWorx 10.0.1

The newer version adds advanced features:

- Read Through mode for automatic cache management
- Improved concurrent access handling
- Enhanced monitoring and observability

### Feature Comparison

| Feature | 10.0.0 | 10.0.1 |
|---------|--------|--------|
| Cache Aside | ✓ | ✓ |
| Read Through/Auto-loading | ✗ | ✓ |
| Manual Loading | ✓ | ✓ |
| Monitoring | ✓ | ✓ |

Choose the version that fits your needs - basic caching or advanced features for complex scenarios.
