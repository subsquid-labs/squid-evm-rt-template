# Custom Prometheus server example

This squid indexes `Transfer(address,address,uint256)` events emitted by the
[USDC token contract](https://etherscan.io/address/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48)
and uses them to demonstrate how to attach a **custom Prometheus server**
to the SQD batch processor. The wiring is at [`src/main.ts`](./src/main.ts).

## What is customized

The batch processor exposes a built-in set of `sqd_processor_*` metrics
(chain height, last processed block, mapping speed, sync ETA, sync ratio)
whenever the `PROCESSOR_PROMETHEUS_PORT` or `PROMETHEUS_PORT` environment
variable is set. To add your own metrics, or to control the server
in code, you construct a `PrometheusServer` yourself and pass it to
`run()` via the `prometheus` option.

This example does three things on top of the defaults:

1. **Declares two custom metrics with `prom-client`** —
   `usdc_transfers_processed_total` (a `Counter`) and
   `usdc_transfers_last_batch` (a `Gauge`). Both are constructed with
   `registers: []` so that prom-client does not attach them to its
   global default registry.

2. **Registers them via a `MetricsSink`** so they end up on the
   *same* private registry as the built-in `sqd_processor_*` metrics
   — i.e. they are served from the same `/metrics` endpoint.

3. **Pins the listening port in code** with `prometheus.setPort(3000)`.
   `setPort()` takes precedence over both env vars; without it the port
   comes from `PROCESSOR_PROMETHEUS_PORT` / `PROMETHEUS_PORT`, falling
   back to `0` (an OS-assigned ephemeral port) if neither is set.

The framework still calls `addRunnerMetrics()` and `serve()` on the
server we hand it, so we don't need to do either ourselves.

## Trying it out

```bash
# 0. Install @subsquid/cli a.k.a. the sqd command globally
npm i -g @subsquid/cli

# 1. Install dependencies
npm ci

# 2. Start a Postgres database container and detach
sqd up

# 3. Build and start the processor
sqd process
```

In a separate terminal, scrape the metrics endpoint:

```bash
curl localhost:3000/metrics
```

You should see the built-in runner metrics:

```
sqd_processor_chain_height ...
sqd_processor_last_block ...
sqd_processor_mapping_blocks_per_second ...
sqd_processor_sync_eta_seconds ...
sqd_processor_sync_ratio ...
```

alongside the two custom ones added by this example:

```
usdc_transfers_processed_total ...
usdc_transfers_last_batch ...
```