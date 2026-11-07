---
title: 'Kafka 4.4 Is Out: Share-Group DLQs, Transactional State Stores, and One Upgrade Gotcha'
date: 2026-11-05T00:00:00-04:00
draft: false
ShowToc: true
description: >-
  Kafka 4.4.0 landed October 22 with 25 KIPs: dead-letter queues for share
  groups (KIP-1191), transactional state stores for Kafka Streams (KIP-892),
  and a storage-formatting change (KIP-1170) that can bite you mid-upgrade.
  The exact configs, the poison-message flow, and the upgrade order that
  keeps you safe.
tags:
  - kafka
  - java
  - backend
  - system-design
categories: article
keywords:
  - kafka 4.4
  - kafka share group dead letter queue
  - KIP-1191
  - kafka streams transactional state stores
  - KIP-892
  - KIP-1170 kafka-storage
---

Apache Kafka 4.4.0 landed on October 22, 2026 — 25 KIPs and 963 commits since 4.3.0. Three things in it are worth your attention: share groups finally get dead-letter queues, Kafka Streams gets transactional state stores, and one storage change can surprise you during an upgrade. There is also a safety note that comes *before* all of it.

## First: if you run 4.3.x, patch to 4.3.2 first

Kafka 4.3.2 shipped the very next day, October 23, and it is not a routine bugfix release. It fixes around 60 issues, including two potential data-loss bugs:

- **KAFKA-20732** (tiered storage only) — size-based retention could delete data that was still within retention from *both* the local and remote tiers after a leader election.
- **KAFKA-20979** (not specific to tiered storage) — the time index could be truncated across a broker restart, causing segments still within retention to be deleted by time-based retention.

If you run 4.3.0 or 4.3.1, go to 4.3.2 (or straight to 4.4.0, which includes the fixes) before you do anything else in this post. KAFKA-20979 can hit any topic, tiered or not, and KAFKA-20732 matters if you use tiered storage. The rest of this post can wait; data loss cannot.

## Share-group DLQs: poison messages finally have somewhere to go

Share groups — Kafka's queue semantics from KIP-932 — let multiple consumers cooperatively work through a topic, past the one-consumer-per-partition limit of classic consumer groups. (If you read my earlier post on [dead letter queues and poison messages](/posts/dead-letter-queues-poison-messages/) and its "archive, don't DLQ" gotcha: this is the fix for it.) Each record carries a delivery count, and a record that keeps failing eventually hits its delivery limit. Before 4.4, that record just went to the **Archived** state. Archived and invisible: no way to find it, inspect it, or replay it. For a payments pipeline, that is not error handling, it is a black hole.

KIP-1191 fixes that. A share group can now name a dead-letter queue topic. When a record is explicitly rejected by the consumer, or it exhausts its delivery attempts, it moves to a new **Archiving** state, a record is written to the DLQ topic, and only then does it become **Archived**.

### The setup: three layers, all opt-in

DLQs are off by default, and every layer has to agree:

```bash
# 1. Turn the feature on, once every broker runs 4.4
bin/kafka-features.sh --bootstrap-server localhost:9092 \
  upgrade --feature share.version=2

# 2. The topic must be allowed to serve as a DLQ topic (default: false)
bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --alter --topic dlq.payments \
  --config errors.deadletterqueue.group.enable=true

# 3. The group names its DLQ topic (default: blank = no DLQ)
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --entity-type groups --entity-name payments \
  --add-config errors.deadletterqueue.topic.name=dlq.payments
```

Two more defaults worth knowing. The broker only permits DLQ topic names starting with `dlq.` (`errors.deadletterqueue.topic.name.prefix`, default `"dlq."`) — this is a security control, not cosmetics: without it, a share group could be pointed at any topic and the broker would write into it. And the broker does *not* auto-create DLQ topics by default (`errors.deadletterqueue.auto.create.topics.enable`, default `false`); flip it on if you want the topics created for you.

Step 1 is the gate that matters most: DLQs require the `share.version` feature to be at **2**, which only switches on when every broker in the cluster supports it. Rolling upgrade to 4.4 first, feature upgrade second. Without it, DLQs are not enabled, whatever the group and topic configs say.

### What lands on the DLQ topic

Every DLQ record carries context headers, all prefixed `__dlq.errors.`:

| Header | Contents |
| --- | --- |
| `__dlq.errors.topic` | Topic that held the original record |
| `__dlq.errors.partition` | Partition (UTF-8 string) |
| `__dlq.errors.offset` | Offset (UTF-8 string) |
| `__dlq.errors.group` | Group ID, if known |
| `__dlq.errors.delivery.count` | Delivery count, if known |
| `__dlq.errors.message` | Reason for the delivery failure |

By default that is *all* you get — headers, no payload. Copying the original record's key and value into the DLQ record is controlled by `errors.deadletterqueue.copy.record.enable`, and it defaults to `false` for good reason: duplicating payloads costs performance and can leak sensitive data into a topic with different access controls. Turn it on only when your DLQ consumers genuinely need the body to diagnose or replay.

### The consumer side: reject, don't just fail

The whole feature hinges on one consumer behavior: when a record is unprocessable, reject it explicitly instead of crashing and letting redelivery burn through the delivery count:

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "payments");
props.put("share.acknowledgement.mode", "explicit");
props.put("key.deserializer", StringDeserializer.class.getName());
props.put("value.deserializer", StringDeserializer.class.getName());

try (KafkaShareConsumer<String, String> consumer = new KafkaShareConsumer<>(props)) {
    consumer.subscribe(List.of("payments"));
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        for (ConsumerRecord<String, String> record : records) {
            try {
                process(record);
                consumer.acknowledge(record, AcknowledgeType.ACCEPT);
            } catch (UnprocessableException e) {
                // Explicit reject: ends delivery attempts, record goes
                // Archiving -> DLQ (once the group has a DLQ topic).
                consumer.acknowledge(record, AcknowledgeType.REJECT);
            }
        }
    }
}
```

Explicit acknowledgement mode is required for per-record `acknowledge()` calls: in the default implicit mode, the next `poll()` acknowledges everything delivered as accepted. `ACCEPT` and `RELEASE` behave as before; `REJECT` is the new fast path to the DLQ. (Snippet compile-checked against kafka-clients 4.4.0; not run against a live cluster.)

### Three honest caveats

- **Duplicates are possible.** The broker does not write the DLQ record atomically with the share-group state update, so in rare cases a record can produce more than one DLQ record. Your DLQ consumer should be idempotent.
- **A failed DLQ write does not block the record.** If the broker cannot write to the DLQ topic for a non-retryable reason, it logs an error and the record goes to **Archived** anyway. Monitor the `DeadLetterQueueFailedProduceRequestsPerSec` metric — a DLQ that silently stops receiving is worse than no DLQ.
- **One DLQ topic per group is the sane convention.** `dlq.payments` for the `payments` group keeps schemas predictable. A single cluster-wide DLQ topic works, but its records will have every schema in your system.

## Transactional state stores: no more wipe-and-rebuild on error

This one is for Kafka Streams, and it fixes a pain every exactly-once Streams developer knows. Before 4.4, when a Streams app using exactly-once processing hit an error, its local state stores were often wiped and rebuilt from scratch — replaying the entire changelog. For a large store, that is minutes of recovery for a transient failure.

KIP-892 adds transactional state stores, enabled with one config:

```properties
processing.guarantee=exactly_once_v2
enable.transactional.statestores=true
```

With it on, writes are buffered in a transaction buffer and committed with the changelog commit. On error, only writes since the last successful commit are lost and replayed — the store itself survives. Two tuning knobs come with it: `statestore.uncommitted.max.bytes` caps how much uncommitted data is buffered, and `default.interactive.query.isolation.level` controls whether interactive queries can see uncommitted data.

Be honest about the maturity, though. The feature is **off by default**, and there is already a known sharp edge: KAFKA-21230 reports that long rebalances can let transaction buffers grow past the configured limit and risk out-of-memory errors in native memory, because commits are skipped mid-rebalance. Test it on your own workloads in staging before production — the recovery win is real, but this is a first-release feature, not a battle-tested default.

## The upgrade gotcha: KIP-1170 and storage formatting

This is the change most likely to surprise operators, and it has nothing to do with features. Under KIP-1170, storage directories formatted by the `kafka-storage` tool are **no longer forward-compatible**. A broker must be the same version as, or newer than, the `kafka-storage` tool that formatted its directory — regardless of the `--release-version` you chose at format time.

The old mental model was: format with `--release-version 4.3`, run any broker that supports 4.3 metadata. That is gone. The practical rules now:

1. **Format with the version you deploy.** Do not format a new node's storage with a newer `kafka-storage` tool and then run an older broker against it.
2. **During a rolling upgrade, upgrade the tooling with the brokers.** If your provisioning formats disks as a separate step from starting brokers, make sure the formatting step uses the same version as the broker that will read those disks.
3. **Check your automation.** Any script, image, or operator that bundles `kafka-storage` separately from the broker binary is now a version-skew bug waiting to happen.

This one will not show up in staging if staging is built fresh with matching versions — it shows up when a new node joins an old cluster, or when a disk is reformatted during an incident. Worth a line in your upgrade runbook.

## Also in 4.4, briefly

- **Cheaper tiered storage (KIP-1241):** `remote.copy.lag.bytes` and `remote.copy.lag.ms` let you delay uploads to remote storage, cutting the redundancy of copying data that is about to be deleted anyway.
- **DNS resilience (KIP-909):** a new client config, `bootstrap.resolve.timeout.ms`, opts into asynchronous bootstrap DNS resolution: the client constructor returns immediately and resolution failures surface later instead of at startup. The default `0` keeps the old synchronous behavior, the feature is marked as evolving, and Kafka Streams and Connect do not support it yet.
- **Security fix worth upgrading for:** CVE-2026-111115 — a broker out-of-memory via tagged-field allocation amplification in flexible protocol parsing — is fixed in 4.4.0 (and backported to 4.2.2 and 4.3.2). The fix caps tagged-field sections at 10,000 fields before allocation.
- **Deprecation:** `broker.id` is deprecated in favor of `node.id` (KIP-1232) and goes away in Kafka 5.0.

## The upgrade order

1. On 4.3.x? Patch to 4.3.2 first — the data-loss fixes are the priority.
2. Rolling upgrade to 4.4.0. Read the "Upgrading to 4.4" notes first; the KIP-1170 formatting rule affects any node you reformat.
3. Upgrade the `share.version` feature to 2, then configure DLQ topics per group. Start with `copy.record.enable` off.
4. Enable transactional state stores in staging only, watch memory during rebalances, and promote after a clean soak.

Kafka 4.4 is not a flashy release. It makes Kafka a more practical queue, a more resilient stream processor, and a cheaper long-term store — and it closes two data-loss holes on the way there. That is a good month's work.

## Sources

- Apache Kafka 4.4.0 upgrade notes and 4.3.2 release announcement (kafka.apache.org)
- KIP-1191: Dead-letter queues for share groups (cwiki.apache.org)
- KIP-892: Transactional Semantics for StateStores (cwiki.apache.org)
- KIP-1170: Unify cluster metadata bootstrapping (kafka.apache.org upgrade notes)
- KAFKA-20732, KAFKA-20979, KAFKA-21230 (issues.apache.org)
- CVE-2026-111115 (oss-security)
