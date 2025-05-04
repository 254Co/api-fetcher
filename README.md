# API Fetcher

A powerful and flexible Python package for fetching data from various APIs with advanced features like middleware, request queuing, progress tracking, and more.

## Features

- Asynchronous API requests with automatic retries
- Support for REST, GraphQL, and SOAP APIs
- Automatic pagination detection and handling
- Rate limiting and circuit breaker patterns
- Request queuing with prioritization and cancellation
- Progress tracking and visualization
- Middleware system for request/response processing
- Comprehensive error handling
- Response caching and connection pooling
- Metrics collection and monitoring

## Installation

```bash
pip install api-fetcher
```

## Basic Usage

```python
from api_fetcher import AsyncAPIDataFetcher

async def main():
    # Initialize the fetcher
    fetcher = AsyncAPIDataFetcher(
        base_url="https://api.example.com",
        show_progress=True,
        concurrency=5
    )
    
    # Fetch data
    df, metadata = await fetcher.fetch(
        query="data",
        priority=1  # Higher priority request
    )
    
    print(f"Fetched {len(df)} rows")
    print(f"Processing time: {metadata['processing_time']:.2f}s")

# Run the async function
import asyncio
asyncio.run(main())
```

## Advanced Features

### Middleware System

The middleware system allows you to process requests and responses through a chain of middleware components:

```python
from api_fetcher import (
    AsyncAPIDataFetcher,
    LoggingMiddleware,
    AuthenticationMiddleware,
    RateLimitMiddleware,
    RetryMiddleware,
    MetricsMiddleware,
    ValidationMiddleware,
    CircuitBreakerMiddleware
)

# Initialize fetcher with middleware
fetcher = AsyncAPIDataFetcher(
    base_url="https://api.example.com",
    auth=("user", "pass")
)

# Add middleware components
fetcher.add_middleware(LoggingMiddleware(log_level=logging.INFO))
fetcher.add_middleware(AuthenticationMiddleware(auth_token="your-token"))
fetcher.add_middleware(RateLimitMiddleware(requests_per_second=10))
fetcher.add_middleware(RetryMiddleware(max_retries=3))
fetcher.add_middleware(MetricsMiddleware())
fetcher.add_middleware(ValidationMiddleware(
    request_schema={"page": int, "limit": int},
    response_schema={"data": list, "total": int}
))
fetcher.add_middleware(CircuitBreakerMiddleware(
    failure_threshold=5,
    reset_timeout=60
))
```

### Request Queue with Prioritization

The request queue system supports prioritization, cancellation, and tagging:

```python
# Add requests with different priorities
request_id1 = await fetcher.add_request(
    context=request_context1,
    priority=1,  # High priority
    tags={"important", "user-data"},
    metadata={"user_id": 123}
)

request_id2 = await fetcher.add_request(
    context=request_context2,
    priority=2,  # Normal priority
    tags={"background", "analytics"}
)

# Wait for specific tags
result = await fetcher.get_result(
    request_id1,
    wait_for_tags={"important", "user-data"}
)

# Cancel requests by tags
cancelled = fetcher.cancel_by_tags({"background"})

# Get queue statistics
stats = fetcher.get_queue_stats()
print(f"Completed requests: {stats['completed_requests']}")
print(f"Average wait time: {stats['average_wait_time']:.2f}s")
```

### Progress Tracking

Track and visualize progress of API requests:

```python
from api_fetcher import ProgressTracker, ProgressBar

# Initialize progress tracking
tracker = ProgressTracker()
tracker.add_callback("progress_bar", ProgressBar().update)

# Track batch requests
async with AsyncAPIDataFetcher(
    base_url="https://api.example.com",
    show_progress=True
) as fetcher:
    # Fetch multiple requests with progress tracking
    results = await fetcher.fetch_batch([
        {"query": "data1", "priority": 1},
        {"query": "data2", "priority": 2},
        {"query": "data3", "priority": 1}
    ])
```

### Custom Middleware

Create custom middleware for specific needs:

```python
from api_fetcher import Middleware

class CustomMiddleware(Middleware):
    async def process_request(self, context):
        # Add custom headers
        context.headers["X-Custom-Header"] = "value"
        return context

    async def process_response(self, context):
        # Process response
        if context.status_code == 429:
            # Handle rate limiting
            await asyncio.sleep(1)
        return context

# Add custom middleware
fetcher.add_middleware(CustomMiddleware())
```

### Error Handling

Comprehensive error handling with custom exceptions:

```python
from api_fetcher import APIFetcherError, RateLimitError, ValidationError

try:
    df, metadata = await fetcher.fetch(query="data")
except RateLimitError as e:
    print(f"Rate limit exceeded: {e}")
except ValidationError as e:
    print(f"Validation failed: {e}")
except APIFetcherError as e:
    print(f"API error: {e}")
```

## Practical Examples

### GitHub API Integration

```python
from api_fetcher import AsyncAPIDataFetcher, RateLimitMiddleware

async def fetch_github_data():
    # Initialize fetcher for GitHub API
    fetcher = AsyncAPIDataFetcher(
        base_url="https://api.github.com",
        headers={"Accept": "application/vnd.github.v3+json"},
        auth=("username", "token")
    )
    
    # Add GitHub-specific middleware
    fetcher.add_middleware(RateLimitMiddleware(requests_per_second=30))  # GitHub's rate limit
    
    # Fetch repository data
    repos_df, metadata = await fetcher.fetch(
        query="/user/repos",
        priority=1
    )
    
    # Fetch issues for each repository
    issues = []
    for repo in repos_df["name"]:
        issues_df, _ = await fetcher.fetch(
            query=f"/repos/username/{repo}/issues",
            priority=2
        )
        issues.append(issues_df)
    
    return repos_df, issues

### Twitter API Integration

```python
from api_fetcher import (
    AsyncAPIDataFetcher,
    RateLimitMiddleware,
    RetryMiddleware,
    CircuitBreakerMiddleware
)

async def fetch_twitter_data():
    fetcher = AsyncAPIDataFetcher(
        base_url="https://api.twitter.com/2",
        headers={"Authorization": f"Bearer {TWITTER_TOKEN}"}
    )
    
    # Add Twitter-specific middleware
    fetcher.add_middleware(RateLimitMiddleware(requests_per_second=50))
    fetcher.add_middleware(RetryMiddleware(max_retries=3))
    fetcher.add_middleware(CircuitBreakerMiddleware(
        failure_threshold=3,
        reset_timeout=300  # 5 minutes
    ))
    
    # Fetch tweets with pagination
    tweets = []
    next_token = None
    
    while True:
        try:
            df, metadata = await fetcher.fetch(
                query="/tweets/search/recent",
                params={
                    "query": "python",
                    "next_token": next_token
                }
            )
            tweets.append(df)
            next_token = metadata.get("next_token")
            if not next_token:
                break
        except Exception as e:
            print(f"Error fetching tweets: {e}")
            break
    
    return pd.concat(tweets, ignore_index=True)

### E-commerce API Integration

```python
from api_fetcher import (
    AsyncAPIDataFetcher,
    ValidationMiddleware,
    MetricsMiddleware
)

async def fetch_product_data():
    fetcher = AsyncAPIDataFetcher(
        base_url="https://api.ecommerce.com/v1",
        headers={"X-API-Key": API_KEY}
    )
    
    # Add validation and metrics middleware
    fetcher.add_middleware(ValidationMiddleware(
        request_schema={
            "category": str,
            "page": int,
            "limit": int
        },
        response_schema={
            "products": list,
            "total": int,
            "page": int
        }
    ))
    fetcher.add_middleware(MetricsMiddleware())
    
    # Fetch products with progress tracking
    async with fetcher as f:
        categories = ["electronics", "clothing", "books"]
        results = await f.fetch_batch([
            {
                "query": "/products",
                "params": {"category": cat, "page": 1, "limit": 100},
                "priority": 1
            }
            for cat in categories
        ])
    
    # Process results
    products_df = pd.concat([df for df, _ in results])
    metrics = fetcher.get_queue_stats()
    
    return products_df, metrics

### Real-time Data Streaming

```python
from api_fetcher import (
    AsyncAPIDataFetcher,
    LoggingMiddleware,
    CircuitBreakerMiddleware
)

async def stream_market_data():
    fetcher = AsyncAPIDataFetcher(
        base_url="https://api.market.com/stream",
        timeout=30.0
    )
    
    # Add streaming-specific middleware
    fetcher.add_middleware(LoggingMiddleware(log_level=logging.INFO))
    fetcher.add_middleware(CircuitBreakerMiddleware(
        failure_threshold=2,
        reset_timeout=60
    ))
    
    # Stream market data
    while True:
        try:
            df, metadata = await fetcher.fetch(
                query="/market-data",
                params={"symbols": "AAPL,GOOGL,MSFT"}
            )
            
            # Process real-time data
            process_market_data(df)
            
            # Check for circuit breaker
            if metadata.get("circuit_breaker_state") == "open":
                print("Circuit breaker open, waiting...")
                await asyncio.sleep(60)
                
        except Exception as e:
            print(f"Streaming error: {e}")
            await asyncio.sleep(5)

### Batch Processing with Progress

```python
from api_fetcher import (
    AsyncAPIDataFetcher,
    ProgressTracker,
    ProgressBar
)

async def process_large_dataset():
    fetcher = AsyncAPIDataFetcher(
        base_url="https://api.data.com",
        show_progress=True,
        concurrency=10
    )
    
    # Initialize progress tracking
    tracker = ProgressTracker()
    tracker.add_callback("progress_bar", ProgressBar().update)
    
    # Prepare batch requests
    batch_size = 1000
    total_items = 10000
    requests = []
    
    for i in range(0, total_items, batch_size):
        requests.append({
            "query": "/data",
            "params": {
                "offset": i,
                "limit": batch_size
            },
            "priority": 1 if i < 5000 else 2  # Prioritize first half
        })
    
    # Process in batches with progress tracking
    results = []
    async with fetcher as f:
        for batch in chunks(requests, 10):  # Process 10 requests at a time
            batch_results = await f.fetch_batch(batch)
            results.extend(batch_results)
            
            # Update progress
            tracker.update(increment=len(batch))
    
    return pd.concat([df for df, _ in results])

## Autonomous Optimization

The package now includes autonomous optimization capabilities that automatically adapt to your system resources and API characteristics. The `AutonomousFetcher` class handles all optimization automatically:

```python
from api_fetcher import AutonomousFetcher

async def fetch_data():
    # Initialize with autonomous optimization
    fetcher = AutonomousFetcher(
        base_url="https://api.example.com",
        auth=("user", "pass")
    )
    
    # Fetch data - optimization is handled automatically
    df, metadata = await fetcher.fetch(query="data")
    
    # Get optimization report
    report = fetcher.get_optimization_report()
    print(f"Current concurrency: {report['api_characteristics']['optimal_concurrency']}")
    print(f"Success rate: {report['api_characteristics']['success_rate']:.2%}")
```

### Autonomous Features

1. **System Resource Optimization**
   - Automatically detects CPU cores, memory, and network speed
   - Adjusts concurrency based on available resources
   - Optimizes batch sizes for your system

2. **API Pattern Recognition**
   - Learns from API response patterns
   - Adapts to rate limits and error patterns
   - Optimizes retry strategies and circuit breaker settings

3. **Dynamic Configuration**
   - Automatically adjusts timeouts based on response times
   - Optimizes middleware chain based on API characteristics
   - Adapts to changing network conditions

4. **Performance Monitoring**
   - Tracks success rates and response times
   - Monitors rate limit frequency
   - Provides detailed optimization reports

### Example: Autonomous Batch Processing

```python
async def process_large_dataset():
    fetcher = AutonomousFetcher(
        base_url="https://api.example.com",
        show_progress=True
    )
    
    # Prepare requests
    requests = [
        {"query": f"/data/{i}", "priority": 1 if i < 1000 else 2}
        for i in range(2000)
    ]
    
    # Process with autonomous optimization
    results = await fetcher.fetch_batch(requests)
    
    # Get optimization report
    report = fetcher.get_optimization_report()
    print("Optimization Report:")
    print(f"- System: {report['system_metrics']['cpu_count']} CPUs, "
          f"{report['system_metrics']['network_speed_mbps']:.1f} Mbps")
    print(f"- API: {report['api_characteristics']['success_rate']:.1%} success rate, "
          f"{report['api_characteristics']['avg_response_time']:.2f}s avg response")
```

### Example: Real-time Adaptation

```python
async def stream_with_adaptation():
    fetcher = AutonomousFetcher(
        base_url="https://api.example.com/stream",
        timeout=30.0
    )
    
    while True:
        try:
            # Fetch with autonomous optimization
            df, metadata = await fetcher.fetch(
                query="/market-data",
                params={"symbols": "AAPL,GOOGL,MSFT"}
            )
            
            # Process data
            process_market_data(df)
            
            # Get current optimization state
            report = fetcher.get_optimization_report()
            if report['api_characteristics']['success_rate'] < 0.9:
                print("Warning: API performance degrading")
                
        except Exception as e:
            print(f"Error: {e}")
            await asyncio.sleep(5)
```

The autonomous optimization system continuously learns and adapts to:
- System resource availability
- Network conditions
- API response patterns
- Error rates and types
- Rate limit behavior

This ensures optimal performance without manual configuration.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the LICENSE file for details.