MIL v1.1 — Memory Interoperability Layer with Path Archiving
Status: Draft
Version: 1.1
Date: 2026-07-26
License: MIT

1. Introduction
1.1 Purpose
MIL (Memory Interoperability Layer) defines a memory and state architecture for AI applications. It enables:

Portable memory: Users can export and import their full AI memory across platforms.
Encrypted storage: Sensitive memory layers are encrypted client-side with AES-256-GCM.
Semantic retrieval: Memories are organized by type and structured path for efficient recall.
Dynamic state management: The character's current emotional and relational state (from PSDL) is stored here, bridging personality and memory.
Standardized path archiving: Every memory is stored in a deterministic path based on PSDL identity, memory type, and scene, enabling context-aware automatic retrieval.

1.2 Design Goals
User Ownership: All memory and state data belongs exclusively to the user. The platform never has access to plaintext sensitive data.
Encryption-First: Layer 2 (Semantic Graph), Dynamic State, and archived memory contents MUST be encrypted with AES-256-GCM when containing sensitive information.
Portability: A complete .mil export MUST be importable by any compatible client, preserving the path structure.
Append-Only Integrity: Layer 1 (Event Log) MUST be append-only and cryptographically verifiable.
Deterministic Pathing: Every memory is stored under a standardized, PSDL-anchored path, allowing automatic retrieval driven by conversation context.

1.3 Scope
This specification defines:

The memory and state taxonomy
The standardized directory and naming structure for memory archiving
JSON schema for each layer and for archived memory files
Encryption requirements
Mandatory client operations, including context-aware path querying
Versioning and migration rules

This specification does NOT define:

How data is stored on disk at the filesystem level (the path structure is a logical organization, implementations may map it to any storage backend)
How encryption keys are managed (user password, biometrics, etc.)
How memory is retrieved or queried beyond the described recall() and path-based operations
The internal implementation of automatic path assembly (e.g., NLP models, intent classifiers)

2. Memory & State Taxonomy
2.1 Layer 1: Event Log (Immutable)
Purpose: Records raw, chronological interactions. This is the source of truth.

Schema:

Field	Type	Required	Description
event_id	string	Yes	UUID v4. Globally unique.
timestamp	string	Yes	ISO 8601 UTC format (e.g., "2026-06-26T10:00:00Z")
actor	string	Yes	MUST be "user" or "assistant"
persona_id	string	Yes	References a PSDL id (e.g., "luffy_v1")
payload_type	string	Yes	MUST be one of: "text", "image", "file", "action", "system"
content	string	Yes	The actual message content (plaintext or base64-encoded)
content_hash	string	Yes	SHA-256 hash of content before any encoding
parent_event_id	string	No	For threading. References another event_id
metadata	object	No	Implementation-specific extensions

Rules:

Layer 1 is append-only. Existing events MUST NOT be modified or deleted.
content_hash MUST be computed BEFORE any encoding.
Implementations SHOULD verify content_hash on read to detect corruption.
Events MUST be stored in chronological order by timestamp.

2.2 Layer 2: Semantic Graph (Indexed)
Purpose: Extracts and organizes knowledge from Layer 1 events into structured, queryable nodes. These nodes may also be referenced by the path archiving system.

Nodes Collection:

Field	Type	Required	Description
node_id	string	Yes	UUID v4
node_type	string	Yes	MUST be one of: "entity", "topic", "decision", "project", "preference"
label	string	Yes	Human-readable name (max 64 characters)
properties	object	No	Key-value store (e.g., {"confidence": 0.92, "priority": "high"})
source_event_ids	array	Yes	List of Layer 1 event_ids that informed this node
last_updated	string	Yes	ISO 8601 UTC timestamp

Edges Collection:

Field	Type	Required	Description
edge_id	string	Yes	UUID v4
source_node_id	string	Yes	References a node_id
target_node_id	string	Yes	References another node_id
relation	string	Yes	MUST be one of: "derived_from", "categorized_as", "related_to", "conflicts_with", "supports"
weight	float	No	0.0–1.0, representing relationship strength

2.3 Layer 3: Meta-Memory (System Control)
Purpose: Stores high-level user habits, session context, and system-level configuration. Plaintext by design, but MUST NOT contain PII.

Schema:

Field	Type	Required	Description
user_habits	object	No	Learned preferences (e.g., {"preferred_verbosity": 7})
session_context	object	No	Current session goals
system_config	object	No	Client-specific configuration

2.4 Dynamic State (PSDL Bridge)
Purpose: Manages the character's current dynamic state — mood, relationship with user, and contextual focus. This is the runtime counterpart to the static PSDL definition. This layer is read/write during a session and persists across sessions.

Schema:

Field	Type	Required	Description
psdl_id	string	Yes	References the active PSDL id (e.g., "luffy_v1")
mood	string	No	Current emotional state (e.g., "excited", "angry", "calm")
mood_intensity	integer	No	0–100. How strongly the mood is felt.
relationship_with_user	string	No	Current relationship (e.g., "nakama", "stranger", "rival")
relationship_strength	integer	No	0–100. How strong the bond is.
current_goal	string	No	What the character is currently focused on (e.g., "Protect the crew")
attention_focus	array	No	Entities, topics, or people the character is paying attention to right now.
last_updated	string	No	ISO 8601 timestamp of last state change.

Example:

{
  "psdl_id": "luffy_v1",
  "mood": "angry",
  "mood_intensity": 95,
  "relationship_with_user": "nakama",
  "relationship_strength": 80,
  "current_goal": "Defeat the enemy threatening his crew",
  "attention_focus": ["enemy commander", "injured crewmate"],
  "last_updated": "2026-07-26T12:30:00Z"
}

Relationship with PSDL:

PSDL Layer 1 (Core Logic) defines who the character is — immutable.
PSDL Layer 2 (Behavior Profile) defines how the character speaks — adjustable.
MIL Dynamic State defines how the character feels right now — fluid, context-driven.
When a PSDL personality is loaded, its initial Dynamic State should be set to defaults. The state evolves through interaction.

3. Memory Path Archiving System
3.1 Design Rationale
To enable deterministic storage and automatic context-aware retrieval, every persisted memory object (a structured summary or insight derived from interactions) MUST be stored under a logical path anchored to the PSDL identity. This pathing system acts as a standardized index on top of the logical layers.

3.2 Core Naming Rule: PSDL ID Anchoring
All memory paths begin with the PSDL personality id, ensuring that memories belonging to different AI personas are isolated and portable.

Path syntax:
{psdl_id}/{memory_type}/{scene_type}/{event_id}.json

Example path:
luffy_v1/emotional/personal_growth/evt_20260717_002.json

The event_id used in the filename MUST correspond to a Layer 1 event_id that triggered or anchors this memory entry. If a memory is a composite insight from multiple events, the client may choose the most representative event_id or generate a dedicated memory_id (which should be recorded in the file's metadata).

3.3 Standardized Directory Tree
The following logical directory tree organizes memories by memory type, scene category, and subcategory. Implementations MUST support this structure; the exact mapping to physical storage is implementation-defined.

/MIL/{psdl_id}/
├── factual/                       # Factual memories
│   ├── personal/                  # Personal facts
│   │   ├── identity/              # Name, birthday, relationships
│   │   ├── milestone/             # Graduation, promotion, marriage
│   │   └── preference/            # Dietary, music, brand preferences
│   └── world/                     # World knowledge
│       ├── current_events/        # News, policies, trends
│       └── common_knowledge/      # General facts
├── emotional/                     # Emotional memories
│   ├── personal_growth/           # Personal development
│   │   ├── breakthrough/          # Overcoming fear, achieving goals
│   │   └── vulnerability/         # Hurt, failure, rejection
│   ├── relationship/              # Interpersonal relationships
│   │   ├── conflict/              # Arguments, misunderstandings
│   │   └── connection/            # Dates, deep talks, confessions
│   └── existential/               # Existential experiences
│       ├── peak_experience/       # Extreme joy, accomplishment
│       └── crisis/                # Loss, confusion, loneliness
├── experiential/                  # Experiential memories
│   ├── project/                   # Projects
│   │   └── {project_id}/          # Project name or ID
│   │       ├── research/          # Research activities
│   │       ├── creation/          # Creation activities
│   │       └── collaboration/     # Collaboration activities
│   └── learning/                  # Learning
│       └── {subject}/             # Subject or topic
│           ├── theory/            # Theoretical learning
│           └── practice/          # Practical application
├── relational/                    # Relational memories
│   ├── with_user/                 # Relationship state with the user
│   │   ├── trust_level/           # Trust level snapshots
│   │   └── attachment_style/      # Attachment style records
│   └── with_others/               # Relationships with other entities
├── cultural/                      # Cultural memories
│   ├── norms/                     # Etiquette, taboos, holidays
│   └── language/                  # Idioms, speech patterns
└── dynamic_state/                 # Current dynamic state (live, not archived)
    ├── mood.json                  # Current mood, intensity, trigger cause
    └── current_context.json       # Active conversation context, attention focus

Archived memory files under these directories are JSON objects following a common schema:

{
  "memory_id": "string (UUID v4)",
  "event_id": "string (originating or anchor event_id)",
  "timestamp": "ISO 8601 UTC",
  "memory_type": "string (one of the top-level types: factual, emotional, experiential, relational, cultural)",
  "scene_type": "string (second-level directory name)",
  "subcategory": "string (third-level or deeper directory name, optional)",
  "summary": "string (human-readable summary of the memory, max 500 characters)",
  "details": "object (key-value data relevant to the memory, e.g., emotions, decisions, lessons learned)",
  "related_nodes": ["array of Layer 2 node_ids"],
  "metadata": {}
}

3.4 Automatic Context-Driven Retrieval
A MIL-compatible client SHOULD implement an automatic retrieval mechanism that maps the current conversation context to one or more memory paths. This enables the AI to proactively load relevant memories into the prompt without explicit user commands.

The typical workflow:

Intent and entity extraction: From the user's message and the ongoing context, the client identifies key topics, emotions, and references (e.g., "argument with friend", "interview nervousness").
Path assembly: The extracted clues are mapped to the standardized directory structure. For example:
"argument with friend" → emotional/relationship/conflict/
"job interview" + "anxiety" → factual/personal/milestone/ (check for past interview/job experiences) AND emotional/personal_growth/breakthrough/ (check for past success overcoming fear)
Contextual ranking: Within the target path(s), memory files are sorted by timestamp (most recent first) or by relevance score.
Memory injection: The summaries (and optionally details) of the top N relevant memories are injected into the system prompt or conversation context as a "historical context" block, clearly marked as retrieved memories.

Example of automatic context injection:

[System] Relevant memories retrieved:
- 2026-07-14: You had a conflict with a close friend about future plans. Core issue: different life goals. (Path: luffy_v1/emotional/relationship/conflict/evt_20260714_001.json)
- 2026-06-01: You successfully resolved a communication breakdown by writing a letter. This strategy was highly effective. (Path: luffy_v1/experiential/learning/communication/practice/evt_20260601_005.json)

Implementations must ensure that this retrieval respects encryption boundaries; decryption must happen only client-side after user authentication.

3.5 Integration with Project/Deep Discussion Workflows
When a structured discussion or project session concludes, the client SHOULD automatically generate and archive a memory summary in the appropriate path:

Project end: Save under experiential/project/{project_id}/ with a summary containing final proposal, key decisions, roles, and outcomes.
Emotional/personal growth session: Save under emotional/personal_growth/ or emotional/relationship/ with core insights, emotion curve, and user's self-reflection conclusions.

This automatic archival ensures the memory library stays up to date without user intervention.

4. Encryption Boundary
Layer	Encryption Required?	Algorithm	Key Management
Layer 1 (Event Log)	Optional	Any	User-defined
Layer 2 (Semantic Graph)	Mandatory	AES-256-GCM	User-defined password or PKI
Layer 3 (Meta-Memory)	No (plaintext)	N/A	N/A
Dynamic State	Recommended	AES-256-GCM	User-defined password or PKI
Archived Memory Files (Section 3)	Mandatory for emotional, relational, and other sensitive categories	AES-256-GCM	User-defined; may share key with Layer 2

Encryption Rules:
- All encrypted data in Layer 2, Dynamic State, and archived memory files MUST use unique nonces per encryption operation.
- Archived memory files that contain only non-sensitive factual or cultural data MAY be stored in plaintext, but the default SHOULD be encryption.
- When exporting, encrypted objects are stored as base64-encoded ciphertext blobs.

5. File Format (Serialization)
5.1 Single-File Export
All layers and the path-archived memories MAY be exported as a single .mil file:

{
  "mil_version": "1.1",
  "user_id": "anon_f8e7d3a1",
  "exported_at": "2026-07-26T12:00:00Z",
  "dynamic_state": { ... },
  "path_archives": {
    "directory_tree": "a JSON structure representing the logical tree with file paths as keys and memory metadata as values (actual content may be embedded or referenced as encrypted blobs)"
  },
  "layers": {
    "layer1_event_log": [ ... ],
    "layer2_semantic_graph": { "nodes": [...], "edges": [...] },
    "layer3_meta_memory": { ... }
  }
}

5.2 Encryption of Export File
Layer 2, Dynamic State, and sensitive archived memory files MUST be stored as base64-encoded encrypted blobs within the export. The path_archives section MUST preserve the logical path and the encrypted content (or a reference to it) so that a full restore is possible.

6. Mandatory Client Operations
Any MIL-compatible client MUST implement:

Operation	Description
append_event(event)	Add event to Layer 1.
recall(type, limit)	Retrieve nodes from Layer 2 by type.
recall_related(node_id, relation)	Retrieve related nodes.
rebuild_graph()	Re-index Layer 1 into Layer 2.
export(password)	Export full .mil file including path archives.
import(data, password)	Import a .mil file and reconstruct the logical path tree.
get_dynamic_state()	Retrieve current Dynamic State.
set_dynamic_state(state)	Update Dynamic State. MUST validate psdl_id matches active personality.
archive_memory(memory_object)	Persist a memory summary to the appropriate path under the standardized tree. The client MUST determine the correct path based on memory_type, scene_type, etc. If the path does not exist, it MUST be created logically.
query_path(psdl_id, memory_type, scene_type, filters)	Return memory file entries (metadata and content, after decryption) from the specified path, optionally filtered by time range or keywords. This operation is the basis for context retrieval.
context_retrieve(psdl_id, current_message, context)	Analyze the current interaction and return a set of relevant memory summaries from the path archive. Implementation details are client-specific, but the output format MUST include the path and memory_id of each retrieved entry.

7. Versioning and Compatibility
7.1 Version Format
mil_version follows Semantic Versioning: MAJOR.MINOR.

7.2 Compatibility Rules
Change Type	Backward Compatible?	Client Action
Adding optional fields	Yes	Ignore unknown fields
Adding required fields	No	MUST reject older versions
Changing encryption algorithm	No	MUST reject
Changing node_type enum	No	MUST reject
Deprecating a field	Yes (with warning)	Support both old and new
Adding new directory branches in the path tree	Yes	Older clients may not write to new paths but can still read
Removing or renaming a standard path branch	No	MUST reject import; provide migration utility

8. Error Handling
Error Type	Code	Client Action
Invalid mil_version	400	Reject with error: "Unsupported version"
Missing required field	400	Reject with error: "Missing field: [field_name]"
Invalid node_type	400	Reject with error: "Invalid node_type."
Invalid memory_type or scene_type for archive	400	Reject with error: "Unknown memory path category."
Decryption failure	401	Reject with error: "Invalid password or corrupted data"
Malformed JSON	400	Reject with error: "Invalid JSON syntax"
psdl_id mismatch in Dynamic State or path archive	409	Reject with error: "PSDL identity mismatch"
Missing path archive during import that was referenced	400	Reject or warn; must not lose data

9. Security Considerations
Encryption keys MUST NOT be hardcoded or stored in plaintext.
content_hash in Layer 1 provides integrity verification.
Layer 3 MUST NOT contain PII.
Dynamic State and emotional/relational memory files MUST be encrypted due to sensitivity.
When deleting any .mil file or memory archive, implementations SHOULD overwrite with random data before deletion.
Path-based retrieval SHOULD ensure that only memories belonging to the currently active psdl_id are accessible during a session.

10. Full Example
Below is a minimal but complete example of a .mil export containing event log, semantic graph, meta-memory, dynamic state, and one archived memory file (encrypted content represented as a base64 blob).

{
  "mil_version": "1.1",
  "user_id": "anon_f8e7d3a1",
  "exported_at": "2026-07-26T12:00:00Z",
  "dynamic_state": {
    "psdl_id": "luffy_v1",
    "mood": "angry",
    "mood_intensity": 95,
    "relationship_with_user": "nakama",
    "relationship_strength": 80,
    "current_goal": "Defeat the enemy threatening his crew",
    "attention_focus": ["enemy commander", "injured crewmate"],
    "last_updated": "2026-07-26T12:00:00Z"
  },
  "path_archives": {
    "luffy_v1": {
      "emotional": {
        "relationship": {
          "conflict": {
            "evt_20260714_001.json": {
              "memory_id": "mem_9a3b2c1d",
              "event_id": "evt_20260714_001",
              "timestamp": "2026-07-14T18:30:00Z",
              "memory_type": "emotional",
              "scene_type": "relationship",
              "subcategory": "conflict",
              "summary": "Argument with friend about future plans. Core disagreement on life goals.",
              "encrypted_content": "base64encryptedblob..."
            }
          }
        }
      }
    }
  },
  "layers": {
    "layer1_event_log": [
      {
        "event_id": "evt_20260714_001",
        "timestamp": "2026-07-14T18:25:00Z",
        "actor": "user",
        "persona_id": "luffy_v1",
        "payload_type": "text",
        "content": "I had a huge fight with my best friend about where to live in the future.",
        "content_hash": "sha256:abc123...",
        "parent_event_id": null
      },
      {
        "event_id": "evt_20260714_002",
        "timestamp": "2026-07-14T18:25:15Z",
        "actor": "assistant",
        "persona_id": "luffy_v1",
        "payload_type": "text",
        "content": "That sounds painful. Do you want to talk about what was said?",
        "content_hash": "sha256:def456...",
        "parent_event_id": "evt_20260714_001"
      }
    ],
    "layer2_semantic_graph": {
      "nodes": [
        {
          "node_id": "node_001",
          "node_type": "topic",
          "label": "Friendship Conflict",
          "properties": { "confidence": 0.95, "priority": "high" },
          "source_event_ids": ["evt_20260714_001", "evt_20260714_002"],
          "last_updated": "2026-07-14T18:30:00Z"
        }
      ],
      "edges": []
    },
    "layer3_meta_memory": {
      "user_habits": { "preferred_verbosity": 7 },
      "session_context": { "current_goal": "Resolve friendship tension" },
      "last_activity": "2026-07-26T11:55:00Z"
    }
  }
}

11. License
MIT License.

12. References
AES-GCM: https://csrc.nist.gov/pubs/sp/800/38d/final
PSDL Specification: See /docs/psdl/psdl-v1.0.md in this repository