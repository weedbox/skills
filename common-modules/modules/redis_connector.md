# redis_connector

Redis client connector with connection pooling.

## Installation

```bash
go get github.com/weedbox/common-modules/redis_connector
```

## Usage in Fx

```go
import "github.com/weedbox/common-modules/redis_connector"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        redis_connector.Module("redis"),
    }, nil
}
```

## Configuration

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.host` | `0.0.0.0` | Redis host |
| `{scope}.port` | `6379` | Redis port |
| `{scope}.password` | (empty) | Redis password |
| `{scope}.db` | `0` | Database number (0-15) |

### TOML Example

```toml
[redis]
host = "localhost"
port = 6379
password = "secret"
db = 0
```

### Environment Variables

```bash
export REDIS_HOST=localhost
export REDIS_PORT=6379
export REDIS_PASSWORD=secret
export REDIS_DB=0
```

## Basic Usage

```go
import "github.com/weedbox/common-modules/redis_connector"

type Params struct {
    fx.In
    Redis *redis_connector.RedisConnector
}

type MyModule struct {
    params Params
}

func (m *MyModule) GetClient() *redis.Client {
    return m.params.Redis.GetClient()
}
```

## String Operations

```go
ctx := context.Background()
client := m.params.Redis.GetClient()

// Set value
err := client.Set(ctx, "key", "value", 0).Err()

// Set with expiration
err := client.Set(ctx, "key", "value", 24*time.Hour).Err()

// Set only if not exists
ok, err := client.SetNX(ctx, "key", "value", 24*time.Hour).Result()

// Get value
val, err := client.Get(ctx, "key").Result()
if err == redis.Nil {
    // Key does not exist
}

// Get multiple keys
vals, err := client.MGet(ctx, "key1", "key2", "key3").Result()

// Increment
newVal, err := client.Incr(ctx, "counter").Result()
newVal, err := client.IncrBy(ctx, "counter", 10).Result()
```

## Hash Operations

```go
ctx := context.Background()
client := m.params.Redis.GetClient()

// Set hash field
err := client.HSet(ctx, "user:123", "name", "John", "email", "john@example.com").Err()

// Get hash field
name, err := client.HGet(ctx, "user:123", "name").Result()

// Get all fields
fields, err := client.HGetAll(ctx, "user:123").Result()
// fields = map[string]string{"name": "John", "email": "john@example.com"}

// Check field exists
exists, err := client.HExists(ctx, "user:123", "name").Result()

// Delete field
err := client.HDel(ctx, "user:123", "email").Err()
```

## List Operations

```go
ctx := context.Background()
client := m.params.Redis.GetClient()

// Push to list
err := client.LPush(ctx, "queue", "item1", "item2").Err()
err := client.RPush(ctx, "queue", "item3").Err()

// Pop from list
val, err := client.LPop(ctx, "queue").Result()
val, err := client.RPop(ctx, "queue").Result()

// Blocking pop (for job queues)
val, err := client.BLPop(ctx, 5*time.Second, "queue").Result()

// Get range
items, err := client.LRange(ctx, "queue", 0, -1).Result()

// Get length
length, err := client.LLen(ctx, "queue").Result()
```

## Set Operations

```go
ctx := context.Background()
client := m.params.Redis.GetClient()

// Add to set
err := client.SAdd(ctx, "tags", "golang", "redis", "weedbox").Err()

// Check membership
isMember, err := client.SIsMember(ctx, "tags", "golang").Result()

// Get all members
members, err := client.SMembers(ctx, "tags").Result()

// Remove from set
err := client.SRem(ctx, "tags", "redis").Err()

// Set operations
intersection, _ := client.SInter(ctx, "set1", "set2").Result()
union, _ := client.SUnion(ctx, "set1", "set2").Result()
diff, _ := client.SDiff(ctx, "set1", "set2").Result()
```

## Sorted Set Operations

```go
ctx := context.Background()
client := m.params.Redis.GetClient()

// Add with scores
err := client.ZAdd(ctx, "leaderboard",
    redis.Z{Score: 100, Member: "player1"},
    redis.Z{Score: 200, Member: "player2"},
).Err()

// Get rank (0-based)
rank, err := client.ZRank(ctx, "leaderboard", "player1").Result()

// Get range by rank
players, err := client.ZRange(ctx, "leaderboard", 0, 9).Result()

// Get range by score
players, err := client.ZRangeByScore(ctx, "leaderboard", &redis.ZRangeBy{
    Min: "100",
    Max: "500",
}).Result()

// Increment score
newScore, err := client.ZIncrBy(ctx, "leaderboard", 50, "player1").Result()
```

## Key Management

```go
ctx := context.Background()
client := m.params.Redis.GetClient()

// Check existence
exists, err := client.Exists(ctx, "key").Result()

// Delete keys
deleted, err := client.Del(ctx, "key1", "key2").Result()

// Set expiration
err := client.Expire(ctx, "key", 1*time.Hour).Err()

// Get TTL
ttl, err := client.TTL(ctx, "key").Result()

// Remove expiration
err := client.Persist(ctx, "key").Err()

// Find keys by pattern
keys, err := client.Keys(ctx, "user:*").Result()
```

## Pipelines

Execute multiple commands in a single round-trip:

```go
ctx := context.Background()
client := m.params.Redis.GetClient()

pipe := client.Pipeline()

incr := pipe.Incr(ctx, "counter")
pipe.Expire(ctx, "counter", 1*time.Hour)
get := pipe.Get(ctx, "other_key")

_, err := pipe.Exec(ctx)
if err != nil {
    return err
}

// Access results
counterVal := incr.Val()
otherVal := get.Val()
```

## Transactions

```go
ctx := context.Background()
client := m.params.Redis.GetClient()

// Watch key for changes
err := client.Watch(ctx, func(tx *redis.Tx) error {
    val, err := tx.Get(ctx, "key").Int()
    if err != nil && err != redis.Nil {
        return err
    }

    _, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
        pipe.Set(ctx, "key", val+1, 0)
        return nil
    })
    return err
}, "key")
```

## Caching Pattern

```go
func (m *MyModule) GetUser(ctx context.Context, userID string) (*User, error) {
    client := m.params.Redis.GetClient()
    cacheKey := "user:" + userID

    // Try cache first
    data, err := client.Get(ctx, cacheKey).Bytes()
    if err == nil {
        var user User
        json.Unmarshal(data, &user)
        return &user, nil
    }

    // Cache miss - fetch from database
    user, err := m.fetchUserFromDB(ctx, userID)
    if err != nil {
        return nil, err
    }

    // Store in cache
    data, _ := json.Marshal(user)
    client.Set(ctx, cacheKey, data, 1*time.Hour)

    return user, nil
}
```

## Session Storage

```go
func (m *MyModule) CreateSession(ctx context.Context, userID string) (string, error) {
    client := m.params.Redis.GetClient()

    sessionID := uuid.New().String()
    sessionKey := "session:" + sessionID

    err := client.HSet(ctx, sessionKey,
        "user_id", userID,
        "created_at", time.Now().Unix(),
    ).Err()
    if err != nil {
        return "", err
    }

    client.Expire(ctx, sessionKey, 24*time.Hour)

    return sessionID, nil
}

func (m *MyModule) GetSession(ctx context.Context, sessionID string) (map[string]string, error) {
    client := m.params.Redis.GetClient()
    return client.HGetAll(ctx, "session:"+sessionID).Result()
}
```
