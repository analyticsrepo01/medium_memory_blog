# Reference Implementation Blueprints (SQL, CDC & Google ADK)

This companion document holds the full DDL schemas, streaming pipeline flags, and webhook router boilerplate omitted from the main Medium article for readability.

---

## 1. Column-Selective Change Stream & Streaming Connector

```sql
CREATE CHANGE STREAM companion_memory_cs
FOR workspace_profiles(plan_tier, deployment_region, compliance_mode),
    incident_cases(status, mitigation_summary, resolution_code),
    access_controls(sso_enforced, stepup_policy)
OPTIONS (
  value_capture_type = 'NEW_ROW_AND_OLD_VALUES',
  retention_period = '7d'
);
```

```bash
gcloud dataflow flex-template run cdc-to-agent-memory-pubsub \
  --template-file-gcs-location=\
    gs://dataflow-templates/2026-09-01-00_RC00/flex/Spanner_Change_Streams_to_PubSub \
  --region=us-central1 \
  --parameters=\
    spannerInstanceId=saas-core-prod,\
    spannerDatabase=workspace-state-db,\
    spannerMetadataInstanceId=saas-core-meta,\
    spannerMetadataDatabase=cdc-connector-meta,\
    spannerChangeStreamName=companion_memory_cs,\
    pubsubTopic=projects/saas-ai-prod/topics/user-memory-mutations,\
    outputDataFormat=JSON,\
    rpcPriority=MEDIUM
```

---

## 2. Stateless Memory Router (Watermark + PII Tokenization + Multi-Store Fan-Out)

```python
from fastapi import FastAPI

app = FastAPI()

@app.post("/pubsub/memory-events")
async def ingest_backend_event(envelope: dict):
    ev = decode_cloudevent(envelope)  # {user_id, domain, type, id, commit_ts, delta}

    # 1. Enforce Commit-Timestamp High-Water-Mark & Idempotency
    if not await watermark_db.advance_if_newer(ev.user_id, ev.domain, ev.id, ev.commit_ts):
        return {"status": "IGNORED_STALE_OR_DUPLICATE"}  # HTTP 200 prevents retry storms!

    # 2. Redact / Tokenize PII & Secrets BEFORE touching any LLM or Vector Index
    clean_summary = await dlp_client.deidentify_text(ev.human_summary)

    # 3. Multi-Store Fan-Out by Semantic Event Class
    if ev.type in ("INCIDENT_TRIGGERED", "TICKET_STATUS_CHANGED"):
        await episodic_pgvector.upsert_episode(ev.user_id, ev.case_id, clean_summary)
        await stm_session_bus.push_active_alert(ev.user_id, ev.alert_payload)
    elif ev.type in ("PLAN_TIER_UPGRADED", "REGION_PREFERENCE_UPDATED"):
        await memory_bank.consolidate_direct_fact(ev.user_id, clean_summary)
        await semantic_kg.upsert_bitemporal_edge(ev.user_id, ev.delta, ev.commit_ts)

    return {"status": "SYNCED", "event_id": ev.id}
```

---

## 3. Dual-Path Long-Term Memory Consolidation (`MemoryRevisions`)

```python
# Path A: Post-chat non-blocking session extraction
client.memory_banks.memories.generate(
    name=memory_bank_name,
    vertex_session_source={"session": session_resource_name},
    scope={"user_id": verified_user_id},
    config={"wait_for_completion": False},
)

# Path B: Out-of-Band backend state change (pre-extracted deterministic fact)
client.memory_banks.memories.generate(
    name=memory_bank_name,
    direct_memories_source={
        "direct_memories": [{
            "fact": "Workspace compliance mode upgraded to HIPAA-Strict; "
                    "restrict workloads to single-tenant regional nodes."
        }]
    },
    scope={"user_id": verified_user_id},
)
```

---

## 4. Keyed Episodic Upsert (`pgvector`) & ACID Bitemporal Semantic Swap

```sql
-- Keyed Episodic Upsert (1 consolidated vector row per incident/ticket)
INSERT INTO agent_episodes
  (episode_key, user_id, event_type, status, occurred_at, updated_at, narrative, embedding)
VALUES ($1, $2, 'INCIDENT', $3, $4, $5, $6, $7::vector)
ON CONFLICT (episode_key) DO UPDATE
  SET status     = EXCLUDED.status,
      updated_at = EXCLUDED.updated_at,
      narrative  = EXCLUDED.narrative,
      embedding  = EXCLUDED.embedding
WHERE agent_episodes.updated_at < EXCLUDED.updated_at;

-- Atomic Bitemporal Semantic Swap (zero window with two active rates!)
BEGIN TRANSACTION;
  UPDATE semantic_facts SET valid_to = $4
   WHERE scope = 'app:catalog' AND subject = $1 AND predicate = $2 AND valid_to IS NULL;

  INSERT INTO semantic_facts (scope, subject, predicate, obj_val, valid_from, valid_to, embedding)
  VALUES ('app:catalog', $1, $2, $3, $4, NULL, $5::vector);
COMMIT;
```
