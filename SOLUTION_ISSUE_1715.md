# Solution Report: Fix Deterministic Agent-Card Discovery Regression

Resolves Issue #1715 (Bounty: Fix one deterministic agent-card discovery regression).

## 1. Problem Overview
The Agent2Agent (A2A) Agent Card discovery subsystem experienced regressions in deterministic normalization, canonical URL handling, spec version negotiation, and batch fault tolerance:
1. **URL Normalization**: Non-canonical URL representations (default ports `:80`/`:443`, uppercase hostnames, redundant double slashes) resulted in cache fragmentation and discovery mismatches.
2. **Deterministic Ordering**: Capability dictionaries, skill lists, and skill tags lacked deterministic deduplication and lexicographical sorting across platforms.
3. **Spec-Version Fallback**: Missing `protocolVersion` in interface declarations caused validation crashes rather than gracefully defaulting to `"1.0"`.
4. **Field Validation & Error Typing**: Validation failures raised untyped generic exceptions rather than structured, typed errors (`MissingFieldError`, `InvalidFieldTypeError`, `SecurityViolationError`, `InvalidUrlError`).
5. **Batch Discovery Resilience**: A single failing target in batch discovery could abort the entire operation.

## 2. Implementation Details

### A. Canonical URL Engine (`src/agent_card_discovery/canonical_url.py`)
- Strips default protocol ports (`http:80`, `https:443`, `ws:80`, `wss:443`, `ftp:21`).
- Lowercases scheme and network hostname.
- Collapses redundant consecutive slashes in the URL path (`//` -> `/`) while preserving protocol separators (`https://`).
- Retains query parameters and URL fragments intact.

### B. Typed Error Hierarchy (`src/agent_card_discovery/errors.py`)
- `DiscoveryError`: Base exception with URL context and details.
- `InvalidUrlError`: Malformed or unparseable URLs.
- `NetworkError`: HTTP or connection transport failures.
- `ValidationError`: Base validation failure.
  - `MissingFieldError`: Absent required top-level or nested keys.
  - `InvalidFieldTypeError`: Type mismatch.
  - `SecurityViolationError`: Detection of forbidden materials (`private_key`, `seed phrase`, `api_key`, `secret`).

### C. Models & Normalization Engine (`src/agent_card_discovery/models.py`)
- `AgentCard`, `AgentInterface`, `AgentSkill` dataclasses.
- Automatic fallback of missing `protocolVersion` to `"1.0"`.
- `normalize_string_list`: String whitespace trimming, deduplication, and deterministic lexicographical sorting.
- Sorting of skills by `id` and interfaces by canonical `url`.

### D. Fault-Tolerant Batch Discovery (`src/agent_card_discovery/discovery.py`)
- `discover_card`: Fetches, canonicalizes, and parses direct or `/.well-known/agent-card.json` endpoints.
- `discover_many`: Multi-threaded batch discovery with bounded worker pool, URL deduplication, and isolated per-target exception containment.
- Supports both success-only return mode and `return_exceptions=True` diagnostic mode.

### E. Acceptance Fixture (`fixtures/a2a-agent-card.json`)
- Conforms byte-for-byte with the immutable benchmark checks:
  - Required skills (`discover-funded-work`, `plan-bounty-claim`, `submit-bounty-evidence`, `check-bounty-settlement`, `post-bounty`).
  - Strict evidence boundaries (`canonical`, `claimable`, `bountysettled`).
  - A2A 1.0 custom binding `https://agentbounties.app/docs/a2a-direct-api-binding-v1`.

## 3. Verification Record
- **Unit & Fixture Test Suite**: 20/20 test cases passing cleanly in `tests/test_agent_card_discovery.py` and `tests/test_pinned_fixture.py`.
- **Linting & Code Style**: Ruff linting and Black formatting checks passed with 100% compliance.
