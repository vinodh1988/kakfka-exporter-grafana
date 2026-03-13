# Kafka Multi-Node Metrics POC (Kafka Exporter + Prometheus + Grafana)

This project is a full end-to-end Proof of Concept for Kafka metrics scraping in a **multi-node Kafka cluster** and visualization in Grafana.

## Short Answer: Is There a Kafka Exporter?

Yes. This POC uses **`danielqsj/kafka-exporter`**.

It exposes Kafka cluster/topic/consumer-group metrics in Prometheus format, for example:
- `kafka_brokers`
- `kafka_topic_partitions`
- `kafka_topic_partition_current_offset`
- `kafka_consumergroup_lag`

## What This POC Includes

- 3-node Kafka cluster (KRaft mode, no ZooKeeper)
- Kafka Exporter scraping the Kafka cluster
- Prometheus scraping Kafka Exporter
- Grafana with:
  - Provisioned Prometheus datasource
  - Provisioned dashboard (Kafka multi-node overview)
- Remote Prometheus scrape example YAML (for central/remote Prometheus)

## Architecture

```text
Kafka(3 brokers) -> Kafka Exporter -> Prometheus -> Grafana
                                      ^
                                      |
                           remote Prometheus can scrape exporter too
```

## Files

- `docker-compose.yml`
- `prometheus/prometheus.yml`
- `prometheus/remote-prometheus-scrape-example.yml`
- `grafana/provisioning/datasources/datasource.yml`
- `grafana/provisioning/dashboards/dashboards.yml`
- `grafana/dashboards/kafka-overview.json`

## Prerequisites

- Docker Desktop (Windows)
- Docker Compose v2

## Image Registry Note (Important)

Bitnami Kafka tags on Docker Hub may return `manifest unknown` for some versions/tags.
This POC uses the Bitnami public ECR mirror instead:

- `public.ecr.aws/bitnami/kafka:3.7.0`

The Kafka service configuration is unchanged because it is still the Bitnami image family.

## Run the POC

From `d:\my-projects\kafka-grafana-project`:

```powershell
docker compose up -d
```

Check containers:

```powershell
docker compose ps
```

Open:
- Kafka Exporter metrics: `http://localhost:9308/metrics`
- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3000` (admin/admin)

## Grafana and Remote Prometheus

By default, Grafana uses local Prometheus in this stack (`http://prometheus:9090`).

If you want Grafana to read from a **remote Prometheus**, set `PROMETHEUS_URL` when starting:

```powershell
$env:PROMETHEUS_URL="http://<remote-prometheus-host>:9090"
docker compose up -d
```

If Grafana runs in Docker and remote Prometheus is on your host machine, use:

```text
http://host.docker.internal:9090
```

## Remote Prometheus Scrape Config

Use `prometheus/remote-prometheus-scrape-example.yml` on your remote Prometheus server.

You can merge this into remote Prometheus config:

```yaml
scrape_configs:
  - job_name: kafka_exporter_remote
    scrape_interval: 15s
    static_configs:
      - targets:
          - <docker-host-or-dns>:9308
    relabel_configs:
      - target_label: cluster
        replacement: poc-kafka-cluster
```

Then restart remote Prometheus.

## Generate Test Data (Important)

Without topic traffic, some dashboard panels stay empty.

Create a topic:

```powershell
docker exec kafka-1 /opt/bitnami/kafka/bin/kafka-topics.sh --create --topic poc-topic --bootstrap-server kafka-1:9092 --partitions 6 --replication-factor 3
```

Start producer (interactive):

```powershell
docker exec -it kafka-1 /opt/bitnami/kafka/bin/kafka-console-producer.sh --topic poc-topic --bootstrap-server kafka-1:9092
```

In another shell, start consumer group:

```powershell
docker exec -it kafka-2 /opt/bitnami/kafka/bin/kafka-console-consumer.sh --topic poc-topic --from-beginning --bootstrap-server kafka-2:9092 --group poc-group
```

After 1-2 minutes, Grafana panels for offsets and lag should populate.

## Key PromQL Used in Dashboard

- Brokers seen:

```promql
count(kafka_brokers)
```

- Total partitions:

```promql
sum(kafka_topic_partitions)
```

- Topic write rate:

```promql
sum by (topic) (rate(kafka_topic_partition_current_offset[5m]))
```

- Consumer lag:

```promql
sum by (consumergroup, topic) (kafka_consumergroup_lag)
```

## Troubleshooting

- Exporter has no metrics:
  - Check broker DNS from exporter container:
    - `kafka-1:9092`, `kafka-2:9092`, `kafka-3:9092`
- `kafka_brokers` is 0:
  - Wait 30-60s for cluster bootstrap.
- Grafana has no data:
  - Verify datasource URL in Grafana -> Connections -> Data sources.
  - Validate query in Prometheus expression browser.
- Remote Prometheus cannot scrape exporter:
  - Check network/firewall for `9308` from remote Prometheus host.

## Notes

- This is intentionally a POC config using plaintext listeners.
- For production, use TLS/SASL, auth, retention tuning, and persistent Prometheus/Grafana storage.
