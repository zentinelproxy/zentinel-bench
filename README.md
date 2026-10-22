<p align="center">
  <a href="https://zentinelproxy.io">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/zentinelproxy/design/main/marks/lockup/zentinel-lockup-dark.svg">
      <img alt="Zentinel" src="https://raw.githubusercontent.com/zentinelproxy/design/main/marks/lockup/zentinel-lockup-light.svg" height="52">
    </picture>
  </a>
</p>

<h3 align="center">Zentinel benchmarks</h3>

<p align="center">A reproducible harness for benchmarking Zentinel against Envoy, HAProxy and Nginx with equivalent configurations.</p>

<p align="center">
  <a href="docs/CONCEPT.md">Methodology</a> &middot;
  <a href="https://docs.zentinelproxy.io">Docs</a> &middot;
  <a href="https://github.com/zentinelproxy">All repositories</a>
</p>

The harness starts each proxy in Docker in front of the same backend, drives
it with a load generator at a fixed request rate, and stores the raw output,
per-run JSON and a summary with machine metadata for every run. All four
proxies get equivalent settings, with access logging off. Today it implements
one scenario, `passthrough`; the others in [`scenarios/tier1.yaml`](scenarios/tier1.yaml)
are defined but have no proxy configurations yet.

## Quick start

Requires [mise](https://mise.jdx.dev) and Docker (with the `docker-compose`
command, which the tasks call).

```bash
mise trust
mise run setup                 # mise tools (hey, k6, wrk), oha via cargo, Docker check
mise run pull                  # Envoy, HAProxy, Nginx and backend images
mise run build-zentinel        # zentinel:optimized from ../zentinel (or set ZENTINEL_IMAGE)

mise run quick:envoy           # start Envoy, 30-second run, stop
mise run bench:zentinel        # full benchmark (warmup + 3 runs), passthrough
mise run bench:all             # zentinel, envoy, haproxy, nginx in turn, 10 s cool-down between
mise run compare --latest 1    # markdown comparison of the latest run per proxy (needs jq)
```

## Tasks

`mise tasks` lists them all.

| Task | What it does |
|---|---|
| `mise run setup` | Install load generators and check the environment |
| `mise run pull` | Pull the Envoy, HAProxy, Nginx and backend images |
| `mise run build-zentinel` | Build `zentinel:optimized` from `$ZENTINEL_DIR` (default `../zentinel`) |
| `mise run up:<proxy>` | Start one proxy on `localhost:8080` (`zentinel`, `envoy`, `haproxy`, `nginx`) |
| `mise run quick:<proxy>` | Start the proxy, run a 30-second benchmark, stop it |
| `mise run bench:<proxy> [scenario] [options]` | Start the proxy, run the full benchmark, stop it |
| `mise run bench:all [scenario]` | Run `bench:<proxy>` for all four proxies |
| `mise run bench:native` | Benchmark a native `zentinel` release binary (`$ZENTINEL_BIN`) against a local `miniserve` backend, no Docker |
| `mise run quick-main [url]` | 30-second benchmark against a proxy that is already running |
| `mise run compare [--latest n] [--scenario s] [--output file] [--json]` | Comparison report from `results/` |
| `mise run results` | List recent results |
| `mise run down` / `logs` / `ps` | Stop, follow logs of, or list the containers |
| `mise run check-tools` | Show which load generators are installed |
| `mise run install-oha` / `install-wrk2` | Install oha (cargo); print wrk2 build steps |
| `mise run clean` | Delete everything in `results/` |

Options for `bench:<proxy>`: `--duration`, `--connections`, `--threads`,
`--rate`, `--warmup`, `--runs`, `--target`, `--tool <oha|wrk2|wrk|hey|k6>`,
`--name`.

## Configuration

Defaults are set in `mise.toml` and can be overridden in the shell:

| Variable | Default | Meaning |
|---|---|---|
| `BENCH_DURATION` | `60` | Seconds per run (`quick` uses 30 unless set) |
| `BENCH_CONNECTIONS` | `100` | Concurrent connections |
| `BENCH_RATE` | `10000` | Target requests per second |
| `BENCH_WARMUP` | `10` | Warmup seconds |
| `BENCH_RUNS` | `3` | Runs per benchmark |
| `BENCH_TARGET` | `http://localhost:8080` | URL under test |
| `ZENTINEL_IMAGE` | `zentinel:optimized` | Zentinel image used by `up:zentinel` |

Without `--tool`, the first installed load generator is used, in this order:
oha (HdrHistogram, recommended), wrk2 (corrects coordinated omission), wrk
(no correction), hey, k6.

## Scenarios

| Scenario | Status |
|---|---|
| `passthrough` | Implemented: plain L7 proxying to one backend, configs for all four proxies |
| `routing`, `loadbalancing`, `tls`, `connection_scaling` | Defined in `scenarios/tier1.yaml`, no configs yet |
| `agent_pipeline`, `rate_limiting`, `circuit_breaker` | Planned (Zentinel-specific), not defined yet |

The scenario argument is recorded with the results, but the containers always
mount the passthrough configurations (`docker/docker-compose.yaml`), so another
scenario name does not change what is measured yet.

## Configuration equivalence

All proxies run with:

- the same worker count (auto-detected), connection limits, keepalive and timeouts
- access logging disabled, which matters for a fair comparison

The equivalence matrix is in [docs/CONCEPT.md](docs/CONCEPT.md).

## Results

Each run writes `results/<timestamp>_<proxy>_<scenario>/`:

```
results/20251230_093411_envoy_passthrough/
├── metadata.json      # proxy and tool versions, image, machine, parameters
├── aggregated.json    # mean requests/s and p50/p99 across runs
├── summary.md         # human-readable summary
└── runs/
    ├── run_1.json     # load generator output (latency percentiles, histogram, errors)
    └── run_1.txt
```

`results/` is tracked in git, with a comparison in
[`results/report.md`](results/report.md). The committed runs are from one
Apple M4 laptop (December 2025 and January 2026) at a target rate of 10,000
requests/s, so they compare latency at a fixed rate, not maximum throughput.

## Methodology

[docs/CONCEPT.md](docs/CONCEPT.md) covers coordinated omission, warmup and
cool-down, repeated runs and median reporting, and common pitfalls. Parts of
it (runners, analysis and Nighthawk) describe the planned design rather than
what is in the repository today.

## Related

- [Zentinel](https://github.com/zentinelproxy/zentinel): the proxy under test
- [zentinelproxy.io/benchmarks](https://zentinelproxy.io/benchmarks/): published benchmark results

## License

Apache-2.0, see [`LICENSE`](LICENSE).

---

<p align="center">
  <sub>Part of <a href="https://zentinelproxy.io">Zentinel</a>, the security-first reverse proxy. The shisa guards the gate.</sub>
</p>
