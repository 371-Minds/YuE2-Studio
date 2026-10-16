# YuE2 Studio guidelines

YuE2 Studio is a native desktop music studio: Tauri 2 shell, Rust/Axum service, and native C++/CUDA/Vulkan/Metal engines for Windows and Linux (with macOS support).

## Core Invariants

- **No runtime Python or Node.js:** The application runs on compiled Rust, native C++ engines, and the native web shell. Do not add Python or Node.js to the runtime path.
- **Model engines are adapters:** No UI or server code may hardcode one model's behaviour. All model runners connect via standardized engine adapters.
- **Capability-level provider choice:** Music, ASR, LLM, and cover art can independently be configured as Local or OpenRouter.
- **OpenRouter identifiers:** Keep OpenRouter model identifiers in provider configuration, never hardcoded in source code.
- **Engine contracts:** Every local engine must expose install state, capability metadata, cancellation, and progress reporting before it reaches the UI.
- **User data isolation:** Model weights, API keys, generated media, and caches are runtime user data; never commit them to the repository.

---

## Workspace Structure

The repository is organized into a Cargo workspace, a React frontend, and a Tauri 2 desktop shell:

```text
├── app/                          # React 19 + TypeScript + Vite frontend
├── desktop/                      # Tauri 2 desktop application wrapper
│   └── src-tauri/                # Tauri backend, bundle configs (Linux deb/appimage, Windows NSIS, macOS dmg)
├── crates/
│   ├── music-core/               # Domain models, music theory, ABC score parser/transposer, library data
│   ├── music-engine/             # Adapters for yue2.cpp, model manager, downloads, process supervision
│   ├── music-server/             # Axum HTTP/SSE server, REST API, Model Context Protocol (MCP) server
│   └── audio-post/               # Audio processing, stems, mastering, VST3 hosting
├── engines/                      # Pinned commit metadata for yue2.cpp and music-midi engines
├── scripts/                      # Build and packaging scripts for Windows, Linux, and macOS
└── docs/                         # Documentation, MCP skill specifications, screenshots, and samples
```

---

## Architecture

```text
React UI (WebView2 on Windows, WebKitGTK on Linux)
  │
  ▼
YuE2-Studio desktop binary
  │
  ├─ Rust / Axum Service (http://127.0.0.1:8791)
  │    ├─ REST API & SSE event streams
  │    ├─ MCP Server (http://127.0.0.1:8791/mcp)
  │    ├─ Library & song storage
  │    └─ Engine supervisor & health monitors
  │
  ├─ yue2.cpp `yue-server` (C++/CUDA/Vulkan, GGUF, 127.0.0.1:18087)
  ├─ music-midi (C++/ggml, audio-to-MIDI transcription via ace-midi)
  ├─ music-train (HOT-Step ace-train, LoRA training, optional)
  └─ vst-host (Audio plugin host process)
```

---

## Cross-Platform Conventions

### Linux (x86-64)
- **Data directory:** `$XDG_DATA_HOME/yue2-studio` (defaults to `~/.local/share/yue2-studio/`).
- **Models directory:** `~/.local/share/yue2-studio/models/yue2-cpp/`.
- **Runtime libraries:** Engine libraries are located relative to binaries using `$ORIGIN`.
- **System packages:** Packaged as `.deb` (installed to `/usr/bin/YuE2 Studio` with resources in `/usr/lib/YuE2 Studio/resources/`) and standalone `.AppImage`.
- **Desktop dependencies:** Requires WebKitGTK 4.1 (`libwebkit2gtk-4.1-0`), AppIndicator (`libayatana-appindicator3-1`), ALSA (`libasound2`), and Vulkan drivers.

### Windows (x64)
- **Data directory:** App-local `./data/` for portable/standard installations; falls back to `%LOCALAPPDATA%\YuE2 Studio\` if installed in a read-only path.
- **Engine backends:** Ships CUDA 12 and CUDA 13 backends, Vulkan, and CPU variants.
- **Web view:** Microsoft Edge WebView2.

---

## Development & Testing Commands

### Rust Workspace
```bash
# Run workspace tests
cargo test --workspace

# Run music-server service tests
cargo test -p music-server

# Check formatting and lints
cargo fmt --check
cargo clippy --workspace
```

### Frontend (`app/`)
```bash
cd app
npm ci
npm test                  # Vitest unit and integration tests
npm run build             # Production Vite build
npx tsc --noEmit -p .     # TypeScript typechecking
```

### Running Locally
```bash
# Start backend service (port 8791)
cargo run -p music-server

# In another terminal, start frontend dev server (port 3791)
npm --prefix app run dev
```

### Linux Packaging
```bash
# Build release binaries and bundles (.deb, .AppImage) locally
bash scripts/build-release-linux.sh

# Or build in an isolated Ubuntu 22.04 container (glibc 2.35 compatibility)
bash scripts/build-linux-docker.sh /path/to/output
```

---

## MCP Server & Agent Interaction

When the studio is running, an MCP server is available at `http://127.0.0.1:8791/mcp`.

- Connect using Claude Code or other MCP-compatible agents:
  ```bash
  claude mcp add --transport http yue2-studio http://127.0.0.1:8791/mcp
  ```
- The server provides tools across song generation, ABC score manipulation, track library management, stem extraction, MIDI conversion, and UI window inspection.
- Refer to `docs/mcp-skill.md` for full tool schemas, system prompts, and recipes.

