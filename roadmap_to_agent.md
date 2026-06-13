# Roadmap: Evolving AstroBurst for AI-Agent Integration

This document outlines the strategic roadmap for transforming AstroBurst from a tightly-coupled desktop application into an AI-agent-friendly ecosystem. The final goal is to establish a purely headless backend capable of intensive astronomical image processing, and a distinct frontend repository/process, thus enabling an AI agent to programmatically control the backend without manual user interface interactions.

## The End State

- **Pure Headless Backend (Rust):** A standalone high-performance Rust service (exposing a REST/gRPC API or a CLI interface) responsible for FITS I/O, intensive memory-mapped analysis, rendering, alignment, stacking, and all core logic.
- **Distinct Frontend Repository/Process:** The React/Web application will be completely separated from the Rust core. It will communicate with the backend solely through standard network protocols (HTTP/WebSocket) rather than native IPC bindings.
- **AI-Agent Interface:** An AI agent can send declarative processing pipelines (e.g., "Load M51, extract background, apply SHO stretch, and save to PNG") to the backend. The backend executes the pipeline autonomously and returns results (JSON, generated images).
- **Tauri Elimination:** The Tauri framework, which currently binds the Rust and React sides into a single executable process, will be completely removed.

---

## Phase 1: Decoupling Tauri and Library Extraction

Currently, the core processing logic (`src-tauri/src/core`, `math`, `infra`) is heavily interwoven with Tauri command handlers (`src-tauri/src/cmd`), which serialize state and handle IPC.

1. **Extract Core Library (`astroburst-core`):**
   - Refactor `core`, `math`, and `infra` into a pure Rust library crate.
   - Strip out any remaining Tauri-specific types (`tauri::State`, `tauri::ipc::Response`) from these modules.
   - Ensure the `cache` and `infra` layers do not rely on Tauri’s app directory context, using standard `dirs` or explicit configuration instead.
2. **Abstract Command Handlers:**
   - Rewrite the logic inside `src-tauri/src/cmd` into protocol-agnostic service functions that accept plain Rust structs and return standard `Result<T, E>`.
   - The Tauri handlers will temporarily become thin wrappers calling the service layer.

## Phase 2: Building the Headless Backend

Create a new executable target (`astroburst-server` or `astroburst-cli`) that consumes the `astroburst-core` library.

1. **API Server Implementation:**
   - Implement an HTTP/WebSocket server using `axum` or `actix-web`.
   - Expose the protocol-agnostic service functions (from Phase 1) as RESTful endpoints or WebSocket RPC calls.
   - Example endpoints: `/api/io/process`, `/api/compose/align`, `/api/stack/calibrate`.
2. **CLI / Declarative Pipeline Engine:**
   - Implement a CLI interface (e.g., via `clap`) that can consume a JSON or YAML manifest describing a full processing pipeline.
   - Example: `astroburst-cli run pipeline.json --output ./results`
3. **State Management:**
   - Replace Tauri-managed application state with standard server-side session management or a stateless request-based architecture for processing caches (like the ORIG/KEY dual cache).

## Phase 3: Frontend Migration

With the headless backend operational, the React frontend must be transitioned away from Tauri IPC.

1. **Repository Split:**
   - Move the React codebase (`src/`) into a separate repository (or a clearly separated package in a monorepo).
   - Remove `@tauri-apps/api` dependencies from `package.json`.
2. **API Client Integration:**
   - Create a TypeScript API client layer that mimics the old `invoke` calls but uses `fetch` or WebSocket connections to communicate with the `astroburst-server`.
   - Update `src/infrastructure/tauri/client.ts` to become `src/infrastructure/api/client.ts`.
3. **Asset Handling:**
   - Replace Tauri's custom `asset://` protocol for image previews with standard HTTP static file serving from the Rust backend.
4. **Desktop App Alternative (Optional):**
   - If a unified desktop deliverable is still required, wrap the standalone React frontend in Electron or pure WebKit, which will spawn and manage the Rust backend process.

## Phase 4: Designing the AI-Agent Interface

Optimize the backend API specifically for programmatic reasoning and execution by LLMs or specialized agents.

1. **Agent-Friendly Schema:**
   - Generate and publish OpenAPI/Swagger specifications for the Rust API. This allows AI agents to automatically generate tools and understand endpoint signatures.
2. **Pipeline Manifest Definition:**
   - Formalize the JSON/YAML schema for declarative image processing pipelines. An AI agent should be able to construct a single JSON payload that defines:
     ```json
     {
       "inputs": ["path/to/R.fits", "path/to/G.fits", "path/to/B.fits"],
       "steps": [
         { "action": "align", "method": "phase_correlation" },
         { "action": "background_extraction", "degree": 4 },
         { "action": "stretch", "type": "ghs", "params": { "symmetry_point": 0.5 } }
       ],
       "exports": ["png", "fits"]
     }
     ```
3. **Robust Error Handling & Feedback:**
   - Ensure the backend returns rich, deterministic error messages (e.g., "Alignment failed: insufficient star matches in R.fits") so an AI agent can self-correct and retry gracefully.
4. **Headless Output Artifacts:**
   - Ensure the backend cleanly returns generated artifacts (e.g., file paths, base64 thumbnails, statistics, logs) to the caller for immediate ingestion by the agent's context.

---

**Summary:** By systematically untangling the core processing engine from the Tauri desktop context and exposing it via standard web protocols and CLI tools, AstroBurst will evolve into a powerful, automated headless engine. This architecture inherently supports AI-driven discovery, allowing an agent to explore, process, and analyze massive astronomical datasets without human intervention.
