```markdown
```
# MIL — Memory Interoperability Layer

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Status: Draft](https://img.shields.io/badge/Status-Draft-orange.svg)]()
[![Version: 1.0](https://img.shields.io/badge/Version-1.0-green.svg)]()

**MIL** is an open, encryption‑first protocol for **user‑owned AI memory**.  
It allows AI applications to store, query, and migrate memory across platforms — while ensuring the user, not the platform, controls all sensitive data.

---

## Why MIL?

- **You own your memory** – Export your AI’s memory at any time and take it to another app.
- **Encryption by default** – Personal knowledge is always encrypted client‑side. Platforms never see plaintext.
- **Semantic, not just raw logs** – Memory is organized into entities, topics, decisions, projects, and preferences.
- **Designed for PSDL** – Works seamlessly with the PSDL personality protocol to give characters a persistent, evolving self.

---

## Architecture

MIL defines a **three‑tier** memory model:

| Layer | Name | Purpose | Encryption |
|-------|------|---------|-------------|
| **Layer 1** | Event Log | Immutable, chronological record of all interactions | Optional |
| **Layer 2** | Semantic Graph | Structured knowledge (entities, topics, decisions, projects, preferences) | **Mandatory AES‑256‑GCM** |
| **Layer 3** | Meta‑Memory | User habits, session context, system configuration | Plaintext (non‑PII) |

All memory is stored in a portable `.mil` package, which can be exported and imported between compliant platforms.

---

## Quick Example

A simplified view of a `.mil` memory structure:

```json
{
  "mil_version": "1.0",
  "user_id": "user-abc123-hash",
  "created": "2026-06-15T10:00:00Z",
  "layers": {
    "event_log": [
      {
        "timestamp": "2026-06-15T10:05:00Z",
        "event_type": "user_message",
        "content_hash": "sha256:abcd..."
      }
    ],
    "semantic_graph": {
      "encrypted": true,
      "algorithm": "AES-256-GCM",
      "payload": "<encrypted base64 blob>"
    },
    "meta_memory": {
      "preferred_language": "zh-TW",
      "session_count": 42,
      "last_active": "2026-07-01T22:00:00Z"
    }
  }
}
```

Inside the decrypted **Semantic Graph** (Layer 2), data is organized like this:

```json
{
  "entities": {
    "ent_1": { "name": "Alice", "type": "person", "relationship": "friend" }
  },
  "topics": {
    "topic_1": { "label": "machine learning", "interest_level": "high" }
  },
  "decisions": {
    "dec_1": { "description": "chose React over Vue", "date": "2026-05-01" }
  },
  "projects": {
    "proj_1": { "name": "personal website", "status": "active" }
  },
  "preferences": {
    "pref_1": { "key": "response_style", "value": "concise" }
  }
}
```

---

## Key Features

- **Encryption‑First** – Layer 2 is always encrypted with AES‑256‑GCM. The encryption key never leaves the user’s device.
- **User Ownership** – The `.mil` file belongs to the user. Platforms may cache encrypted blobs but cannot decrypt them.
- **Full Portability** – Migrate your AI’s memory between any MIL‑compatible application.
- **Semantic Retrieval** – Query memory by type (`entity`, `topic`, `decision`, `project`, `preference`) rather than just keyword matching.
- **PSDL Integration** – When used with PSDL, Layer 3 meta‑memory manages the character’s dynamic state (mood, relationship stages, context).

---

## Full Specification

Read the complete technical specification:  
📄 **[MIL v1.0 Specification](./docs/mil-v1.0.md)**

---

## Roadmap

- [x] Core specification (v1.0)
- [ ] Reference client‑side encryption library (JavaScript / Python)
- [ ] `.mil` import/export tooling
- [ ] Integration guide for pairing MIL with PSDL personalities
- [ ] Zero‑knowledge proof for selective memory sharing

---

## License

MIT – see [LICENSE](./LICENSE) for details.

---

## Contributing

MIL is an open community effort. We welcome contributions, implementations, and discussions about portable, user‑owned AI memory.
```
