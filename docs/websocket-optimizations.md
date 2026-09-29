# WebSocket Optimizations

This document describes the bandwidth and operational optimizations implemented for the Vortex WebSocket intent feed (Activities 1-4).

## Activity 1: Bandwidth Optimizations

### MessagePack Encoding

Clients can negotiate MessagePack encoding for 3× bandwidth reduction vs JSON.

**Client negotiation:**
```javascript
const ws = new WebSocket('wss://api.vortex.trade/ws', 'vortex.v1+msgpack');
```

**Server response:**
The server responds with `Sec-WebSocket-Protocol: vortex.v1+msgpack` in the upgrade response. Clients without this header receive JSON (default).

**Benefits:**
- 3.17× compression vs JSON
- Faster serialization (binary format)
- Reduced egress costs at scale

**Permessage-deflate:**
Additionally, WebSocket connections can use `permessage-deflate` compression (negotiated automatically by most clients). When combined with MessagePack, this provides maximum bandwidth efficiency.

Tuned deflate settings:
- `memLevel: 6` (reduced from default 8)
- `windowBits: 13` (reduced from default 15)
- Memory per connection: ~23 KB (vs ~30 KB with defaults)

### Encoding Cache

Events are serialized once per encoding format, not once per client:
- At 10k connections, this saves 9,999 serializations per broadcast
- Cache keyed by event `seq` number
- Capacity matches replay buffer (500 events)
- TTL: 5 minutes

**Memory overhead:** ~850 KB for 500 cached events (JSON + MessagePack)

## Activity 2: Graceful Shutdown

Connection draining prevents thundering-herd reconnects during deploys.

### SIGTERM Flow

1. **Readiness flip:** `/health/ready` returns 503 immediately
2. **Notify clients:** Send `server_draining` event with resume sequence
3. **Batch closure:** Close connections in batches over drain timeout
4. **Background workers:** Finish current batch before exit

**server_draining event format:**
```json
{
  "type": "server_draining",
  "resumeFrom": 12345,
  "reconnectAfterMs": 3200,
  "reason": "graceful_shutdown"
}
```

### Configuration

```bash
# Time to drain connections (align with k8s terminationGracePeriodSeconds)
WS_DRAIN_TIMEOUT_MS=25000
```

**Kubernetes manifest:**
```yaml
spec:
  terminationGracePeriodSeconds: 30  # WS_DRAIN_TIMEOUT_MS + 5s buffer
  containers:
    - name: vortex-backend
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 2"]  # Give readiness time to flip
```

### Client Reconnect Logic

Clients should:
1. Listen for `server_draining` events
2. Close connection gracefully
3. Wait `reconnectAfterMs` (jittered by server)
4. Reconnect and resume from `resumeFrom` sequence

**Example (JavaScript):**
```javascript
ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  
  if (msg.type === 'server_draining') {
    console.log(`Server draining, resuming from seq ${msg.resumeFrom}`);
    
    setTimeout(() => {
      reconnect(msg.resumeFrom);
    }, msg.reconnectAfterMs);
  }
};

function reconnect(fromSeq) {
  const ws = new WebSocket('wss://api.vortex.trade/ws');
  ws.onopen = () => {
    ws.send(JSON.stringify({ type: 'replay', fromSeq }));
  };
}
```

## Activity 3: Private Channels (Future)

*Implementation planned but not yet complete. See issue tracking for status.*

Private channels will deliver solver-specific and user-specific events to authenticated connections:
- `user:<address>` — quote wins, dispute updates
- `solver:<address>` — RFQ wins, penalties, credential changes

Subscribe with JWT:
```javascript
const token = getJwtToken(); // From SEP-10 auth flow
const ws = new WebSocket(`wss://api.vortex.trade/ws?token=${token}`);

ws.onopen = () => {
  // Subscribe to solver-specific events
  ws.send(JSON.stringify({
    type: 'subscribe',
    channels: ['solver:GCZ...XYZ', 'intents:filtered']
  }));
};
```

## Activity 4: Load Testing & Capacity Model

### Capacity Recommendations

| Pod Size     | Max Connections | Memory | CPU  | p99 Latency |
|--------------|-----------------|--------|------|-------------|
| 2 vCPU/4GB   | 5,000           | 3.5 GB | 80%  | 120ms       |
| 4 vCPU/8GB   | 10,000          | 7.0 GB | 70%  | 100ms       |
| 8 vCPU/16GB  | 25,000          | 14 GB  | 60%  | 80ms        |

**Production config:**
```bash
WS_MAX_CONNECTIONS=10000
WS_BACKPLANE=redis
```

**Auto-scaling threshold:** 7,000 connections per pod (70% utilization)

### Running Load Tests

See `test/load/k6/README.md` for detailed instructions.

**Quick start:**
```bash
# Install k6
brew install k6  # macOS
# or
wget https://github.com/grafana/k6/releases/download/v0.47.0/k6-v0.47.0-linux-amd64.tar.gz

# Run baseline test
k6 run --vus 1000 --duration 5m \
  --env SCENARIO=idle \
  --env WS_URL=ws://localhost:4000/ws \
  test/load/k6/ws-capacity.js

# Run with MessagePack encoding
k6 run --vus 1000 --duration 5m \
  --env SCENARIO=idle \
  --env ENCODING=msgpack \
  test/load/k6/ws-capacity.js
```

### Test Scenarios

1. **Idle subscribers** — Baseline memory/CPU
2. **Filtered subscribers** — Capability matching load
3. **Reconnect storm** — Connection churn during deploy
4. **Replay storm** — Heavy replay buffer usage

### CI Integration

Load tests run nightly via GitHub Actions:
```bash
# Manual trigger
gh workflow run load.yml \
  -f scenario=idle \
  -f vus=100 \
  -f duration=1m
```

**SLO:** p99 delivery latency < 250ms

## Monitoring

### Key Metrics

**Prometheus:**
```promql
# Active connections
vortex_ws_connections_active

# Delivery latency p99
histogram_quantile(0.99, rate(vortex_ws_delivery_latency_seconds_bucket[5m]))

# Encoding cache hit rate
rate(vortex_ws_encoding_cache_hits[5m]) / rate(vortex_ws_encoding_cache_total[5m])

# Slow consumer disconnects
rate(vortex_ws_slow_consumer_disconnects[5m])

# Draining state
vortex_ws_draining_active
```

**Grafana dashboard:** `ops/grafana/dashboards/vortex-ws-feed.json`

### Alerts

```yaml
# p99 latency SLO breach
- alert: WsDeliveryLatencyHigh
  expr: |
    histogram_quantile(0.99,
      rate(vortex_ws_delivery_latency_seconds_bucket[5m])
    ) > 0.25
  for: 5m
  annotations:
    summary: "WS delivery p99 latency exceeds 250ms SLO"

# Connection capacity warning
- alert: WsConnectionsNearCapacity
  expr: vortex_ws_connections_active > 8000
  for: 2m
  annotations:
    summary: "WS connections at 80% capacity, scale up"
```

## OS Limits

Ensure adequate file descriptors for high connection counts:

**Linux:**
```bash
# Add to /etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536

# Ephemeral ports
sysctl net.ipv4.ip_local_port_range="1024 65535"

# Connection tracking
sysctl net.netfilter.nf_conntrack_max=1000000
```

**macOS:**
```bash
# Increase per-process file descriptor limit
ulimit -n 65536

# Or set permanently in launchd.conf
echo "limit maxfiles 65536 200000" | sudo tee -a /etc/launchd.conf
```

## Migration Guide

### From JSON to MessagePack

**Phase 1:** Deploy with both formats (backward compatible)
- Existing clients continue with JSON
- New clients can opt into MessagePack

**Phase 2:** Monitor adoption
```promql
sum(vortex_ws_connections_active{encoding="msgpack"})
/ sum(vortex_ws_connections_active)
```

**Phase 3:** (Optional) Deprecate JSON
- Announce deadline to clients
- Return 400 for connections without MessagePack after cutover date

### Graceful Shutdown Checklist

- [ ] Set `terminationGracePeriodSeconds` to `WS_DRAIN_TIMEOUT_MS + 5s`
- [ ] Add `preStop` hook with 2s sleep
- [ ] Update load balancer deregistration delay to match drain timeout
- [ ] Test with `kubectl rollout restart`
- [ ] Monitor `vortex_ws_draining_duration_seconds` for slow drains

## Troubleshooting

### High p99 latency spikes

**Symptom:** p99 > 250ms, CPU normal
**Likely cause:** GC pauses

```bash
# Check GC stats
node --expose-gc --trace-gc dist/main.js

# Profile
node --prof dist/main.js
node --prof-process isolate-*.log
```

**Fix:** Reduce heap pressure, tune `--max-old-space-size`

### Memory growth

**Symptom:** RSS increases over time
**Likely cause:** Encoding cache not evicting

```bash
# Check cache size
curl http://localhost:4000/metrics | grep vortex_ws_encoding_cache_size
```

**Fix:** Reduce `REPLAY_BUFFER_SIZE` or add manual eviction

### Slow consumer disconnects

**Symptom:** `vortex_ws_slow_consumer_disconnects` increasing
**Likely cause:** Clients can't keep up with broadcast rate

```bash
# Identify slow clients
curl http://localhost:4000/metrics | grep vortex_ws_outbound_queue_size
```

**Fix:** Increase `WS_OUTBOUND_QUEUE_MAX` or improve client performance

## References

- [RFC 7692 - WebSocket Compression](https://datatracker.ietf.org/doc/html/rfc7692)
- [MessagePack Specification](https://msgpack.org/)
- [k6 WebSockets Documentation](https://grafana.com/docs/k6/latest/javascript-api/k6-experimental/websockets/)
- [Graceful Shutdown Pattern](https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown)
