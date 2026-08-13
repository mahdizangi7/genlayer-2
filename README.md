# MultiSourceEventResolver – GenLayer Intelligent Contract

A reusable multi-source event resolver for prediction markets, oracles, and trustless adjudication on GenLayer.

## Purpose

This Intelligent Contract lets you create an event with a natural-language question and a list of independent web sources.  
Validators fetch each source, validate the content, extract the outcome using LLMs, aggregate the results, and reach consensus through GenLayer’s Equivalence Principle.

Use cases:
- Prediction market resolution
- Real-world event verification
- Agent dispute resolution
- Trustless fact checking

## Why this is not a thin wrapper

- Independent multi-source fetching and evaluation
- Source validation (HTTP status, content length, error handling)
- Explicit failure states (`fetch_failed`, `insufficient_content`, `error`)
- Substantive aggregation logic (majority + confidence)
- Full lifecycle: `open → resolving → resolved / failed`
- Challenge mechanism that moves status to `disputed`
- Thoughtful Equivalence Principle with clear tolerances

## How Consensus Works

- Non-deterministic operations: `gl.nondet.web.get` + `gl.nondet.exec_prompt`
- Equivalence Principle: `gl.eq_principle.prompt_comparative`
  - `final_outcome` must be identical
  - `valid_count` may differ by at most 1
  - `final_confidence` tolerance of 0.15
  - Number of source results must be the same

## State Design

- `events`: TreeMap[str, str] (event_id → JSON)
- Clean structure that stores question, sources, per-source results, final decision, and metadata
- Clear status machine with invariants

## Main Functions

### create_event(question, source_urls)
Creates a new event.  
`source_urls` is a comma-separated list of URLs (recommended 2–5 independent sources).  
Returns `event_id`.

### resolve_event(event_id)
Fetches all sources, validates them, runs LLM extraction, aggregates results, and finalizes the event.

### challenge_event(event_id, reason)
Allows challenging a finalized event and sets status to `disputed`.

### get_event(event_id)
Returns the full event data including source results and final outcome.

### get_event_count()
Returns the total number of events created.

## Example Flow

1. Create event with a clear question and several reliable sources  
2. Call `resolve_event`  
3. Read the result with `get_event`  
4. (Optional) Challenge if new evidence appears

## Deployment

- No constructor arguments needed
- Works on GenLayer Studio and Testnet Bradbury
- Best tested with real news, sports, or official sources

## Why this is useful for other builders

This contract demonstrates a production-style pattern for multi-source adjudication that can be extended for insurance claims, delivery verification, content moderation, or any subjective/objective real-world decision that needs consensus.