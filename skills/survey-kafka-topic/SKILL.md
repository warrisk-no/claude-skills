---
name: survey-kafka-topic
description: Inventory what is actually on a Kafka topic — payload shapes, type discriminators, and volumes — by reading it end to end without joining a consumer group, then cross-check against what Elasticsearch indexed to find silently dropped messages. Use when asked what is on a topic, which message types a decoder must support, why data is missing downstream, or to replay retained messages after a decoder fix.
---

# survey-kafka-topic

Find out what a Kafka topic really carries, without disturbing production consumers. This is the diagnostic that answers "which message types do we need to support?" and "are we dropping data?" with evidence rather than inference.

## Trigger

User asks what is on a topic, which payload types a decoder handles or misses, why documents are missing in Elasticsearch, or to replay messages after fixing a decoder.

## What to ask if not provided

1. **Topic** — e.g. `svartbak-raw-groundcontrol`. Required.
2. **Repo** — the survey runs from the repo whose `.env` and `kafka_helper` reach the cluster (`ais` for the AIS/svartbak topics).

## Rule 1 — never join the production consumer group

Read with **no group at all**: `group_id=None` plus manual `assign()` and `seek_to_beginning()`. Offsets are then never committed and no rebalance is triggered, so the running decoder does not notice you.

Never pass the service's group ID (`groundcontrol-decoder`, `ais-transformer`, …) to "have a look". That commits offsets and makes production skip messages.

This is Kafka-specific. A vendor feed can have the opposite property — PNTGuard/SkyRouter MO latest-delivery is at-most-once server side, so *any* manual poll consumes messages production would otherwise receive. Check which kind of feed you are on before reading anything.

## Step 1 — Connect

```python
import sys
sys.path.insert(0, "/Users/kiowa/warrisk/ais")
from dotenv import load_dotenv
load_dotenv("/Users/kiowa/warrisk/ais/.env")   # explicit path: see trap below
from kafka import KafkaConsumer, TopicPartition
from warrisk import kafka_helper

c = KafkaConsumer(
    bootstrap_servers=kafka_helper.get_kafka_brokers(),
    security_protocol="SSL",
    ssl_context=kafka_helper.get_kafka_ssl_context(),
    value_deserializer=lambda v: v,
    enable_auto_commit=False,
    group_id=None,
    consumer_timeout_ms=15000,
)
```

**Trap:** `load_dotenv()` with no argument resolves from the *calling file's* directory, not the cwd. A scratch script in a temp directory silently gets no `KAFKA_URL` and fails with "The KAFKA_URL config variable is not set" even though `.env` is correct. Always pass the absolute path.

## Step 2 — Bound the read

```python
tps = [TopicPartition(topic, p) for p in sorted(c.partitions_for_topic(topic))]
c.assign(tps)
c.seek_to_beginning(*tps)
begins, ends = c.beginning_offsets(tps), c.end_offsets(tps)
total = sum(ends[tp] - begins[tp] for tp in tps)
```

Report the retained range and the window it covers in wall-clock time (`m.timestamp`) — on a low-volume topic the whole history may be a few hours, which bounds how long any recovery stays possible. Stop after `total` messages; do not rely on the consumer timeout alone.

## Step 3 — Tally shapes, not contents

Count the *structure* of each message: the top-level key set, the payload-type key (for a protobuf `oneof`, the single key beside the id), any `type` enum. Print a table of shape → count, and keep one sample per distinct shape for inspection.

Identify the real discriminator empirically. A numeric field that looks like a message type often is not: on the Ground Control feed `imt.messageId` runs 1,2,3,4,5,6 within one device across six *different* payload types — it is a per-device sequence counter. Verify a candidate discriminator by printing it alongside the payload key for every message before dispatching on it.

## Step 4 — Cross-check against what was indexed

The tally alone does not prove loss. Aggregate the same discriminator in Elasticsearch:

```python
es.search(index="<index>", size=0,
          aggs={"kinds": {"terms": {"field": "<discriminator field>"}}})
```

A type present on the topic and absent from the index is being dropped — that is the finding, and it is worth quoting both numbers in the issue ("6 of 16 retained messages are GRAIN; zero GRAIN documents have ever been indexed").

## Step 5 — Replay, if the fix landed after the loss

Retained messages that a broken decoder already consumed can be re-decoded. Do it **outside** the production group:

- Read offsets `[first, last-before-the-fix]` with `group_id=None`.
- **Filter to only the payloads that were dropped.** Everything else in that range was indexed successfully and would duplicate.
- Feed each through the fixed transform function and produce to the decoded topic.

Check how the writer assigns document IDs first. If it lets Elasticsearch auto-generate `_id` (as `orbcomm/elasticwriter.py` does), a replayed message creates a *new* document — replaying anything already indexed duplicates it. Filtering by payload type is what keeps the replay exactly-once in practice.

## Traps

- **Key ordering.** The svartbak raw topics are single-partition; the `ais-raw-*` topics use 32 partitions with a constant `b"group"` key so multi-part NMEA stays in order. Never make a raw-topic key more specific — it splits related messages across partitions.
- **Logging unknown payloads.** When printing shapes from unfamiliar data, log key *names*, never values, and guard `sorted(x)` with an `isinstance(x, dict)` check — sorting a list dumps its contents into the log.
- **Retention is the clock.** Decide on a replay the same day. State the retained window explicitly in any report so the deadline is visible.
