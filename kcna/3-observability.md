# Observability

## Prometheus

The 4 main Prometheus monitoring metrics are:

- Counter
  A metric type that only increases (or is reset). Good for counting events like requests served, jobs completed, or errors occurred.
- Gauge
  A metric that can go up and down. Used for current values like memory usage, temperature, number of concurrent users, etc.
- Histogram
  Tracks the distribution of observed values in buckets (e.g., request duration). Useful for quantiles, latency ranges, and computing counts/sums.
- Summary
  Also tracks value distribution and quantiles over a sliding window, with client-side quantile calculation in addition to count and sum.