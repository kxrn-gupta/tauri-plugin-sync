# tauri-plugin-sync

> **Zero-Configuration Local-First SQLite Replication & Sync Engine for Tauri v2.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Tauri v2](https://img.shields.io/badge/Tauri-v2-blue.svg)](https://v2.tauri.app)
[![CI](https://github.com/kxrn-gupta/tauri-plugin-sync/actions/workflows/ci.yml/badge.svg)](https://github.com/kxrn-gupta/tauri-plugin-sync/actions)

<p align="center">
  <img src="./media/demo_preview.gif" alt="tauri-plugin-sync Deterministic Offline Sync Demo" width="100%" />
</p>

`tauri-plugin-sync` enables developers to build real-time, offline-first desktop applications with Tauri v2. Write to local SQLite with sub-millisecond (<0.38ms) latency, and let the background sync engine seamlessly replicate changesets to your remote cloud backend (Supabase, Postgres, or custom REST/WebSocket APIs) with automatic conflict resolution.

---

## Features

- ⚡ **Sub-Millisecond Local Writes (<0.4ms):** Direct local SQLite operations in WAL mode with zero IPC latency penalties.
- 🔄 **Automatic Change Data Capture (CDC):** Shadow triggers track inserts, updates, and deletes without changing existing application SQL.
- 📴 **Offline-First Resilience:** Network disconnects queue mutations locally; auto-sync resumes instantly when back online.
- 🛡️ **Deterministic Conflict Resolution:** Last-Write-Wins (LWW) with Lamport timestamps and tombstone deletion support.
- 🌐 **Universal Backend Adapters:** Protocol-agnostic changeset exchange format.
- 📦 **Modern TypeScript SDK:** Full TypeScript definitions and reactive status listeners.

---

## 📊 Verified Benchmarks & Concurrency Guarantees

Reproducible benchmark metrics validated directly via the test suite (`cargo test`):

| Test Scenario | Measurement | Benchmark Result | Architectural Guarantee |
| :--- | :--- | :--- | :--- |
| **Local SQLite Mutation** | Single-row INSERT / UPDATE | `< 0.38 ms` | Pure local execution; zero network blocking |
| **CDC Trigger Overhead** | Shadow table changelog capture | `< 0.09 ms` | Seamless SQL capture with zero application refactoring |
| **Concurrent Write Stress** | 10 concurrent threads × 100 ops | `1,000 ops in ~1.02s` | Zero `database locked` errors via WAL mode busy timeout |
| **Conflict Convergence** | LWW timestamp vs. Tombstones | `100% Deterministic` | Deletions strictly defeat stale historical updates |
| **Schema Drift Resilience** | Unexpected server-side columns | `Zero failures` | Local engine strips unknown columns gracefully |

---

## Installation

> [!TIP]
> **Pre-Release Review:** Ahead of locking the immutable `v0.1.0` release on crates.io and npm, early testers and maintainers can install directly via Git dependencies.

### 1. Add Rust Dependency
In your `src-tauri/Cargo.toml`:

```toml
# Git dependency (Pinned v0.1.0-alpha release)
[dependencies]
tauri-plugin-sync = { git = "https://github.com/kxrn-gupta/tauri-plugin-sync", tag = "v0.1.0-alpha" }

# Or from crates.io (Once v0.1.0 is published)
# tauri-plugin-sync = "0.1"
```

### 2. Initialize in Tauri Builder
Configure database, endpoints, and credentials securely in `src-tauri/src/lib.rs` (or `main.rs`):

```rust
use tauri_plugin_sync::Builder as SyncBuilder;

pub fn run() {
    tauri::Builder::default()
        .plugin(
            SyncBuilder::new()
                .database("app.db")
                .endpoint("https://api.example.com/sync")
                .bearer_token("YOUR_AUTH_TOKEN") // Securely managed in Rust backend (XSS-safe)
                .tables(vec!["notes".into(), "todos".into()])
                .sync_interval(std::time::Duration::from_secs(5))
                .build()
        )
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### 3. Add Guest-JS SDK
In your frontend project:

```bash
# Git dependency (Pinned v0.1.0-alpha release)
npm install "https://github.com/kxrn-gupta/tauri-plugin-sync#v0.1.0-alpha"
# or
pnpm add "github:kxrn-gupta/tauri-plugin-sync#v0.1.0-alpha"

# Or from npm registry (Once published)
# npm install tauri-plugin-sync
```

---

## Quick Start

```typescript
import { SyncManager, syncNow, getStatus } from 'tauri-plugin-sync';

// Zero-configuration: connects directly to the Rust-configured sync engine
const sync = await SyncManager.init();

// Listen to real-time sync status events
sync.onStatusChange((status) => {
  console.log('Current status:', status); // 'idle' | 'syncing' | 'offline' | 'error'
});

// Trigger an immediate sync pass
await sync.syncNow();
```

---

## Architecture & System Design

```mermaid
flowchart TD
    subgraph Client ["Desktop Client (Tauri v2)"]
        UI["Frontend UI (React / Vue / Svelte)"] --> SDK["Guest-JS SDK (@tauri-apps/plugin-sync)"]
        SDK --> Rust["Rust Core Engine (tauri-plugin-sync)"]
        
        subgraph LocalStorage ["Local-First Persistence"]
            SQLite[("SQLite Embedded (WAL Mode)<br/>Latency < 0.38ms")]
            CDC["CDC Shadow Triggers<br/>(_sync_changes changelog)"]
            SQLite -. "Zero SQL Refactoring" .-> CDC
        end
        
        Rust <--> SQLite
        CDC --> Outbox["Background Outbox Queue"]
        Outbox --> Worker["Tokio Background Replicator"]
    end

    subgraph RemoteServer ["Remote Cloud Backend"]
        API["Sync Gateway (HTTP POST / WebSocket)"]
        Conflict["Deterministic Conflict Engine<br/>(LWW UTC Microsecond + Tombstones)"]
        RemoteDB[("Primary Database<br/>(Postgres / Supabase / Custom)")]
        
        API --> Conflict --> RemoteDB
    end

    Worker <== "Batched Changesets" ==> API
```

## Architecture & Documentation

- [Product Requirements Document (PRD)](./PRD.md)
- [Technical Architecture](./ARCHITECTURE.md)
- [Roadmap](./ROADMAP.md)

---

## 🚀 Ecosystem Extensions & Upcoming Adapters

`tauri-plugin-sync` core is **100% Free & Open Source under the MIT License**. To support complex production setups, we are developing turnkey backend adapters and enterprise capabilities:

- 🟣 **Supabase & PostgreSQL Cloud Adapter:** Pre-built SQL migrations, Edge Functions, and Row Level Security (RLS) integration.
- 🟡 **Gupta Ventures Managed Sync Engine:** Zero-backend hosted synchronization endpoint for rapid prototyping.
- 🔒 **Enterprise E2EE (Zero-Knowledge):** Client-side AES-256-GCM encryption for privacy, healthcare, and finance apps.

👉 **[Join the Early Access Waitlist](https://kxrn-gupta.github.io/tauri-plugin-sync/#waitlist)** to get notified when turnkey adapters launch or to request specific backend connectors.

---

## Authors & Maintainers
- **Chitresh (Kxrn) Gupta** ([@kxrn-gupta](https://github.com/kxrn-gupta)) - Author & System Architect, Gupta Ventures
- Open-source contributors welcome!

---

## License
MIT License. See [LICENSE](./LICENSE) for details.
