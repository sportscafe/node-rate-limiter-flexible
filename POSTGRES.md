## RateLimiterPostgres

### Usage

PostgreSQL >= 9.5

Note: It isn't recommended to use it with more than 200-300 limited actions per second.

It is recommended to use pool of connections.

By default, RateLimiterPostgres creates separate table by `keyPrefix` for every limiter.

All limits are stored in one table if `tableName` option is set.

Limits data, which expired more than an hour ago, are removed every 5 minutes by `setTimeout`.

[See detailed options description here](https://github.com/animir/node-rate-limiter-flexible#options)

```javascript
const { Pool } = require('pg');

const client = new Pool({
  host: '127.0.0.1',
  port: 5432,
  database: 'root',
  user: 'root',
  password: 'secret',
});

const opts = {
  // Basic options
  storeClient: client,
  points: 5, // Number of points
  duration: 1, // Per second(s)

  // Custom
  tableName: 'mytable', // if not provided, keyPrefix used as table name
  keyPrefix: 'myprefix', // must be unique for limiters with different purpose
  
  execEvenly: true, // Do not delay actions evenly
  blockDuration: 10, // Block for 10 seconds if consumed more than `points`
  
  inmemoryBlockOnConsumed: 20, // If 20 points consumed in current duration
  inmemoryBlockDuration: 30, // block for 30 seconds in current process memory
  insuranceLimiter: new RateLimiterMemory(
    // It will be used only database error as insurance
    // Can be any implemented limiter like RateLimiterMemory or RateLimiterRedis extended from RateLimiterAbstract
    {
      points: 1, // 1 is fair if you have 5 workers and 1 cluster
      duration: 1,
    }),
};

const rateLimiter = new RateLimiterPostgres(opts);

rateLimiter.consume(userIdOrIp)
  .then((rateLimiterRes) => {
    // There were enough points to consume
    // ... Some app logic here ...
  })
  .catch((rejRes) => {
    if (rejRes instanceof Error) {
      // Some Postgres error
      // Never happen if `insuranceLimiter` set up
      // Decide what to do with it in other case
    } else {
      // Can't consume
      // If there is no error, rateLimiterRedis promise rejected with number of ms before next request allowed
      // consumed and remaining points
      const secs = Math.round(rejRes.msBeforeNext / 1000) || 1;
      res.set('Retry-After', String(secs));
      res.status(429).send('Too Many Requests');
    }
  });
```

### Benchmark

Endpoint is pure NodeJS endpoint launched in `node:latest` and `mysql:5.7` Docker containers with 4 workers

User key is random number from 0 to 300.

Endpoint is limited by `RateLimiterMySQL` with config:

```javascript
new RateLimiterPostgres({
  storeClient: mysql,
  points: 5, // Number of points
  duration: 1, // Per second(s)
});
```

```text
Statistics        Avg      Stdev        Max
  Reqs/sec       997.60     449.81    4472.92
  Latency        8.44ms    12.64ms   161.73ms
  Latency Distribution
     50%     5.05ms
     75%     5.98ms
     90%     9.75ms
     95%    30.72ms
     99%    77.80ms
  HTTP codes:
    1xx - 0, 2xx - 26893, 3xx - 0, 4xx - 3108, 5xx - 0
```