```markdown
```
# MIL v1.0 — Memory Interoperability Layer

**Status**: Draft  
**Version**: 1.0  
**Date**: 2026-06-26  
**License**: MIT

---

## 1. Introduction

### 1.1 Purpose
MIL (Memory Interoperability Layer) defines a **memory and state architecture** for AI applications. It enables:
- **Portable memory**: Users can export and import their full AI memory across platforms.
- **Encrypted storage**: Sensitive memory layers are encrypted client-side with AES-256-GCM.
- **Semantic retrieval**: Memories are organized by type for efficient recall.
- **Dynamic state management**: The character's current emotional and relational state (from PSDL) is stored here, bridging personality and memory.

### 1.2 Design Goals
- **User Ownership**: All memory and state data belongs exclusively to the user. The platform never has access to plaintext sensitive data.
- **Encryption-First**: Layer 2 (Semantic Graph) and Dynamic State MUST be encrypted with AES-256-GCM.
- **Portability**: A complete `.mil` export MUST be importable by any compatible client.
- **Append-Only Integrity**: Layer 1 (Event Log) MUST be append-only and cryptographically verifiable.

### 1.3 Scope
This specification defines:
- The memory and state taxonomy
- JSON schema for each layer
- Encryption requirements
- Mandatory client operations
- Versioning and migration rules

This specification does NOT define:
- How data is stored on disk (file system, database, etc.)
- How encryption keys are managed (user password, biometrics, etc.)
- How memory is retrieved or queried beyond basic `recall()` operations

---

## 2. Memory & State Taxonomy

### 2.1 Layer 1: Event Log (Immutable)

**Purpose**: Records raw, chronological interactions. This is the source of truth.

**Schema**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | string | **Yes** | UUID v4. Globally unique. |
| `timestamp` | string | **Yes** | ISO 8601 UTC format (e.g., `"2026-06-26T10:00:00Z"`) |
| `actor` | string | **Yes** | MUST be `"user"` or `"assistant"` |
| `persona_id` | string | **Yes** | References a PSDL `id` (e.g., `"luffy_v1"`) |
| `payload_type` | string | **Yes** | MUST be one of: `"text"`, `"image"`, `"file"`, `"action"`, `"system"` |
| `content` | string | **Yes** | The actual message content (plaintext or base64-encoded) |
| `content_hash` | string | **Yes** | SHA-256 hash of `content` before any encoding |
| `parent_event_id` | string | No | For threading. References another `event_id` |
| `metadata` | object | No | Implementation-specific extensions |

**Rules**:
- Layer 1 is **append-only**. Existing events MUST NOT be modified or deleted.
- `content_hash` MUST be computed BEFORE any encoding.
- Implementations SHOULD verify `content_hash` on read to detect corruption.
- Events MUST be stored in chronological order by `timestamp`.

### 2.2 Layer 2: Semantic Graph (Indexed)

**Purpose**: Extracts and organizes knowledge from Layer 1 events into structured, queryable nodes.

**Nodes Collection**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `node_id` | string | **Yes** | UUID v4 |
| `node_type` | string | **Yes** | MUST be one of: `"entity"`, `"topic"`, `"decision"`, `"project"`, `"preference"` |
| `label` | string | **Yes** | Human-readable name (max 64 characters) |
| `properties` | object | No | Key-value store (e.g., `{"confidence": 0.92, "priority": "high"}`) |
| `source_event_ids` | array | **Yes** | List of Layer 1 `event_id`s that informed this node |
| `last_updated` | string | **Yes** | ISO 8601 UTC timestamp |

**Edges Collection**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `edge_id` | string | **Yes** | UUID v4 |
| `source_node_id` | string | **Yes** | References a `node_id` |
| `target_node_id` | string | **Yes** | References another `node_id` |
| `relation` | string | **Yes** | MUST be one of: `"derived_from"`, `"categorized_as"`, `"related_to"`, `"conflicts_with"`, `"supports"` |
| `weight` | float | No | 0.0–1.0, representing relationship strength |

### 2.3 Layer 3: Meta-Memory (System Control)

**Purpose**: Stores high-level user habits, session context, and system-level configuration. Plaintext by design, but MUST NOT contain PII.

**Schema**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `user_habits` | object | No | Learned preferences (e.g., `{"preferred_verbosity": 7}`) |
| `session_context` | object | No | Current session goals |
| `system_config` | object | No | Client-specific configuration |

### 2.4 Dynamic State (PSDL Bridge)

**Purpose**: Manages the character's **current dynamic state** — mood, relationship with user, and contextual focus. This is the runtime counterpart to the static PSDL definition.

This layer is **read/write** during a session and persists across sessions.

**Schema**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `psdl_id` | string | **Yes** | References the active PSDL `id` (e.g., `"luffy_v1"`) |
| `mood` | string | No | Current emotional state (e.g., `"excited"`, `"angry"`, `"calm"`) |
| `mood_intensity` | integer | No | 0–100. How strongly the mood is felt. |
| `relationship_with_user` | string | No | Current relationship (e.g., `"nakama"`, `"stranger"`, `"rival"`) |
| `relationship_strength` | integer | No | 0–100. How strong the bond is. |
| `current_goal` | string | No | What the character is currently focused on (e.g., `"Protect the crew"`) |
| `attention_focus` | array | No | Entities, topics, or people the character is paying attention to right now. |
| `last_updated` | string | No | ISO 8601 timestamp of last state change. |

**Example**:

```json
{
  "psdl_id": "luffy_v1",
  "mood": "angry",
  "mood_intensity": 95,
  "relationship_with_user": "nakama",
  "relationship_strength": 80,
  "current_goal": "Defeat the enemy threatening his crew",
  "attention_focus": ["enemy commander", "injured crewmate"],
  "last_updated": "2026-06-27T12:30:00Z"
}
```

**Relationship with PSDL**:
- PSDL Layer 1 (Core Logic) defines **who the character is** — immutable.
- PSDL Layer 2 (Behavior Profile) defines **how the character speaks** — adjustable.
- MIL Dynamic State defines **how the character feels right now** — fluid, context-driven.
- When a PSDL personality is loaded, its initial Dynamic State should be set to defaults. The state evolves through interaction.

---

## 3. Encryption Boundary

| Layer | Encryption Required? | Algorithm | Key Management |
|-------|---------------------|-----------|----------------|
| Layer 1 (Event Log) | Optional | Any | User-defined |
| Layer 2 (Semantic Graph) | **Mandatory** | AES-256-GCM | User-defined password or PKI |
| Layer 3 (Meta-Memory) | No (plaintext) | N/A | N/A |
| Dynamic State | **Recommended** | AES-256-GCM | User-defined password or PKI |

**Encryption Rules for Dynamic State**:
- Dynamic State SHOULD be encrypted because it may contain emotionally sensitive information.
- If encrypted, use the same key as Layer 2 or a separate key at the user's discretion.
- Nonces MUST be unique per encryption operation.

---

## 4. File Format (Serialization)

### 4.1 Single-File Export
All layers MAY be exported as a single `.mil` file:

```json
{
  "mil_version": "1.0",
  "user_id": "anon_f8e7d3a1",
  "exported_at": "2026-06-26T12:00:00Z",
  "dynamic_state": { ... },
  "layers": {
    "layer1_event_log": [ ... ],
    "layer2_semantic_graph": { "nodes": [...], "edges": [...] },
    "layer3_meta_memory": { ... }
  }
}
```

### 4.2 Encryption of Export File
- Layer 2 and Dynamic State MUST be stored as **base64-encoded encrypted blobs** within the export.
- Layer 1 and Layer 3 MAY remain in plaintext.

---

## 5. Mandatory Client Operations

Any MIL-compatible client MUST implement:

| Operation | Description |
|-----------|-------------|
| `append_event(event)` | Add event to Layer 1. |
| `recall(type, limit)` | Retrieve nodes from Layer 2 by type. |
| `recall_related(node_id, relation)` | Retrieve related nodes. |
| `rebuild_graph()` | Re-index Layer 1 into Layer 2. |
| `export(password)` | Export full `.mil` file. |
| `import(data, password)` | Import a `.mil` file. |
| `get_dynamic_state()` | Retrieve current Dynamic State. |
| `set_dynamic_state(state)` | Update Dynamic State. MUST validate `psdl_id` matches active personality. |

---

## 6. Versioning and Compatibility

### 6.1 Version Format
`mil_version` follows Semantic Versioning: `MAJOR.MINOR`.

### 6.2 Compatibility Rules

| Change Type | Backward Compatible? | Client Action |
|-------------|----------------------|---------------|
| Adding optional fields | Yes | Ignore unknown fields |
| Adding required fields | No | MUST reject older versions |
| Changing encryption algorithm | No | MUST reject |
| Changing node_type enum | No | MUST reject |
| Deprecating a field | Yes (with warning) | Support both old and new |

---

## 7. Error Handling

| Error Type | Code | Client Action |
|------------|------|---------------|
| Invalid `mil_version` | 400 | Reject with error: "Unsupported version" |
| Missing required field | 400 | Reject with error: "Missing field: [field_name]" |
| Invalid `node_type` | 400 | Reject with error: "Invalid node_type." |
| Decryption failure | 401 | Reject with error: "Invalid password or corrupted data" |
| Malformed JSON | 400 | Reject with error: "Invalid JSON syntax" |
| `psdl_id` mismatch in Dynamic State | 409 | Reject with error: "Dynamic state does not match active personality" |

---

## 8. Security Considerations

- Encryption keys MUST NOT be hardcoded or stored in plaintext.
- `content_hash` in Layer 1 provides integrity verification.
- Layer 3 MUST NOT contain PII.
- Dynamic State SHOULD be encrypted due to potential emotional sensitivity.
- When deleting a `.mil` file, implementations SHOULD overwrite with random data before deletion.

---

## 9. Full Example

```json
{
  "mil_version": "1.0",
  "user_id": "anon_f8e7d3a1",
  "exported_at": "2026-06-26T12:00:00Z",
  "dynamic_state": {
    "psdl_id": "luffy_v1",
    "mood": "angry",
    "mood_intensity": 95,
    "relationship_with_user": "nakama",
    "relationship_strength": 80,
    "current_goal": "Defeat the enemy threatening his crew",
    "attention_focus": ["enemy commander", "injured crewmate"],
    "last_updated": "2026-06-26T12:00:00Z"
  },
  "layers": {
    "layer1_event_log": [
      {
        "event_id": "evt_001",
        "timestamp": "2026-06-26T10:00:00Z",
        "actor": "user",
        "persona_id": "luffy_v1",
        "payload_type": "text",
        "content": "What are the key risks in our market expansion plan?",
        "content_hash": "sha256:abc123def456...",
        "parent_event_id": null
      },
      {
        "event_id": "evt_002",
        "timestamp": "2026-06-26T10:00:15Z",
        "actor": "assistant",
        "persona_id": "luffy_v1",
        "payload_type": "text",
        "content": "The key risks include regulatory hurdles in new markets, currency fluctuation, and supply chain disruption.",
        "content_hash": "sha256:def456ghi789...",
        "parent_event_id": "evt_001"
      }
    ],
    "layer2_semantic_graph": {
      "nodes": [
        {
          "node_id": "node_001",
          "node_type": "topic",
          "label": "Market Expansion Risks",
          "properties": { "confidence": 0.92, "priority": "high" },
          "source_event_ids": ["evt_001", "evt_002"],
          "last_updated": "2026-06-26T10:00:20Z"
        }
      ],
      "edges": []
    },
    "layer3_meta_memory": {
      "user_habits": { "preferred_verbosity": 7, "preferred_tone": "formal" },
      "session_context": { "current_goal": "Market expansion risk assessment" },
      "last_activity": "2026-06-26T10:00:20Z"
    }
  }
}
```

---

## 10. License
MIT License.

## 11. References
- AES-GCM: https://csrc.nist.gov/pubs/sp/800/38d/final
- PSDL Specification: See `/docs/psdl/psdl-v1.0.md` in this repository
```
