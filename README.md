# rahyab

**rahyab** (راه‌یاب — "wayfinder") is a high-throughput geospatial place-search
service in Go. Given a location and an optional text query, it returns nearby
points of interest ranked by a blend of text relevance and distance — the core
"what's near me?" problem behind any maps or ride-hailing product.

Built as a hands-on study of production backend engineering: Go services,
Elasticsearch-backed geo + text retrieval, caching, event-driven ingestion,
containerization, Kubernetes orchestration, and observability.

> **Status:** early development. This README describes the target system; the
> roadmap below tracks what's actually built. See [`docs/`](./docs) for design
> decisions as they're made.

## What it does

```
GET /search?lat=35.7219&lng=51.3347&q=coffee&radius=1000
```

Returns the nearest matching places, ranked. Under the hood:

- **Geospatial retrieval** over POI data using Elasticsearch `geo_point` /
  `geo_distance` queries.
- **Text retrieval** with relevance scoring, fuzzy matching (spell correction),
  and normalization (lowercasing + Persian/Arabic character folding).
- **Ranking** combining a normalized text score with distance into one ordering.
- **Caching** of hot queries in Redis.
- **Event-driven ingestion**: place create/update events arrive over Kafka and
  are indexed by a concurrent worker pool.

## Architecture

```
          ┌──────────┐   search    ┌───────────────┐
 client ──▶  HTTP API │────────────▶ Elasticsearch  │  (geo + text index)
          │   (Go)   │◀────────────│               │
          └────┬─────┘   results   └───────▲───────┘
               │                            │ index
          ┌────▼─────┐              ┌────────┴───────┐
          │  Redis   │  cache       │  ingest worker │
          │ (hot q)  │              │  pool (Go)     │
          └──────────┘              └────────▲───────┘
                                              │ consume
                                      ┌───────┴───────┐
                                      │     Kafka     │  place.updates
                                      └───────────────┘

observability: Prometheus (metrics) + Grafana (dashboards) + Kibana (logs)
orchestration: Docker + Kubernetes (kind/minikube) with probes, limits, HPA
```

## Tech stack

| Concern          | Choice                                      |
|------------------|---------------------------------------------|
| Language         | Go                                          |
| Search / geo     | Elasticsearch (`geo_point`, `geo_distance`) |
| Cache            | Redis                                       |
| Messaging        | Kafka                                       |
| Container / orch | Docker, Kubernetes (kind / minikube)        |
| Metrics          | Prometheus + Grafana                        |
| Logs             | Structured logs → Elasticsearch / Kibana    |
| Load testing     | k6 / vegeta                                 |
| Data             | OpenStreetMap (Tehran extract)              |

## Roadmap

- [ ] **Phase 0** — Go fundamentals; hello-world JSON API with a test
- [ ] **Phase 1** — `/search` skeleton; load real POI data; brute-force nearest (Haversine) baseline
- [ ] **Phase 2** — Elasticsearch geo index; `geo_distance` queries; text query; sub-100ms search
- [ ] **Phase 3** — ranking function (text score + distance), fuzzy spell correction, normalization
- [ ] **Phase 4** — Redis cache; Kafka ingestion; concurrent worker pool; graceful degradation
- [ ] **Phase 5** — Dockerfile; Kubernetes deploy (probes, resource limits, HPA); pod-failure recovery
- [ ] **Phase 6** — Prometheus/Grafana metrics, Kibana logs, load test + dashboards

## Running locally

> Work in progress — instructions land as each phase completes.

```bash
# planned
make up        # bring up ES, Redis, Kafka via docker compose
make run       # run the API locally
make load      # run a load test against it
```

## Design notes

Decisions and trade-offs are documented as they're made — why Elasticsearch over
a hand-rolled spatial index, how the ranking function is tuned, how the service
degrades when a dependency is down. See [`docs/decisions.md`](./docs/decisions.md).

## License

MIT
