# Redis replication and automatic Sentinel failover

Redis and Sentinel use stable IP addresses on the Compose network (172.21.0.0/16).
Sentinel monitors numeric addresses to avoid blocking Docker DNS lookups when a
container stops. Slow DNS previously caused repeated TILT mode and prevented
automatic promotion.

Address mapping:
- redis-0: 172.21.0.3
- redis-1: 172.21.0.5
- redis-2: 172.21.0.4

The configuration files in redis/ and sentinel/ are initial templates. On first
startup they are copied into the existing named data volumes. Subsequent starts
preserve /data/redis.conf and /sentinel-data/sentinel.conf so Redis role changes,
Sentinel IDs, elected primaries, and election epochs survive restarts. Template
changes do not overwrite existing runtime configuration. Do not delete volumes
to apply configuration changes: use Redis/Sentinel configuration commands and
persist them with CONFIG REWRITE or SENTINEL FLUSHCONFIG as appropriate.

## Verify before testing

Run from this directory:

```sh
docker compose exec sentinel-01 redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
docker compose exec sentinel-01 redis-cli -p 26379 SENTINEL replicas mymaster
docker compose exec sentinel-01 redis-cli -p 26379 SENTINEL ckquorum mymaster
docker compose exec sentinel-01 redis-cli -p 26379 INFO sentinel
```

Before stopping a primary, confirm two healthy replicas, three Sentinels,
sentinel_tilt:0, and successful quorum. Discovery and replica synchronization
can take a few seconds after startup. Avoid SENTINEL RESET immediately before
a test: it clears discovered replicas until they are rediscovered.

Stop the CURRENT primary service identified by the address mapping above,
not necessarily redis-0. For example, only if redis-0 is currently primary:

```sh
docker compose stop redis-0
docker compose exec sentinel-01 redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
docker compose logs --since 2m sentinel-01 sentinel-02 sentinel-03
docker compose start redis-0
```

The primary address should change automatically after failure detection and
election. The returned node should become a replica of the elected primary.
Check INFO replication on all three Redis services: exactly one master should
have connected_slaves:2, and the other two should have master_link_status:up.
Redis does not automatically move the primary role back to redis-0.

Clients needing automatic recovery must discover the primary through Sentinel
using the service name mymaster and Sentinel port 26379. A client pinned to
redis-0 will not follow a promotion. This configuration is for local development:
Redis permits unauthenticated peer-container access and has no published host
port. Keep the configured subnet free of conflicts with other Docker networks.
