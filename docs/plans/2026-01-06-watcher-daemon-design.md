# Audiobook Forge Watcher Daemon - Design Document

## Overview

This document captures the design for a separate repository `audiobook-forge-watcher` that will implement GitHub Issue #7: an automatic Docker container that watches directories for new audiobooks and converts them without user interaction.

**Repository:** New separate repo (not a workspace within audiobook-forge)

**Relationship to audiobook-forge:** The watcher daemon invokes the `audiobook-forge` CLI binary as a black box. It treats audiobook-forge as an external tool and doesn't depend on its internal libraries.

---

## Architecture

### High-Level Components

```
┌─────────────────────────────────────────────────────────┐
│                  audiobook-forge-watcher                │
│                                                           │
│  ┌──────────────┐      ┌─────────────┐                 │
│  │   Scanner    │─────▶│    Queue    │                 │
│  │   (Polling)  │      │   (FIFO)    │                 │
│  └──────────────┘      └─────────────┘                 │
│         │                      │                         │
│         ▼                      ▼                         │
│  ┌──────────────┐      ┌─────────────┐                 │
│  │ State Manager│◀─────│  Processor  │                 │
│  │  (SQLite)    │      │ (Sequential)│                 │
│  └──────────────┘      └─────────────┘                 │
│                               │                          │
│                               ▼                          │
│                        ┌─────────────┐                  │
│                        │   Cleanup   │                  │
│                        │   Handler   │                  │
│                        └─────────────┘                  │
│                                                           │
└───────────────────────────┬───────────────────────────┘
                            │ spawns
                            ▼
                   ┌──────────────────┐
                   │ audiobook-forge  │
                   │  CLI (external)  │
                   └──────────────────┘
```

### 1. Directory Scanner

**Purpose:** Discover audiobook directories that need processing

**Behavior:**
- Polls watch directory at configurable interval (default: 60 seconds)
- Recursively scans all subdirectories
- Identifies directories containing audio files (MP3, M4A, AAC)
- Validates using audiobook-forge's logic (checks if directory is processable)
- Skips directories that already contain an M4B file
- Skips directories already tracked in SQLite as processed

**Implementation approach:**
- Use `std::fs::read_dir` with recursion
- File extension matching for audio files
- Could invoke `audiobook-forge build --dry-run` to validate (if such flag exists/added)
- Or: implement minimal heuristic (directory with 1+ audio files, no M4B)

### 2. State Manager (SQLite)

**Purpose:** Track conversion history and prevent reprocessing

**Schema:**
```sql
CREATE TABLE audiobooks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    path TEXT UNIQUE NOT NULL,
    status TEXT NOT NULL,  -- 'pending', 'processing', 'completed', 'failed'
    processed_at TIMESTAMP,
    error_message TEXT,
    attempts INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_status ON audiobooks(status);
CREATE INDEX idx_path ON audiobooks(path);
```

**Operations:**
- `mark_discovered(path)` - Add new audiobook with status='pending'
- `mark_processing(path)` - Update to status='processing'
- `mark_completed(path)` - Update to status='completed' with timestamp
- `mark_failed(path, error)` - Update to status='failed' with error message
- `get_pending()` - Retrieve all pending audiobooks
- `is_processed(path)` - Check if path already completed

**Database location:** `/data/watcher.db` (mounted Docker volume)

### 3. Conversion Queue & Processor

**Purpose:** Process audiobooks one at a time

**Queue behavior:**
- Simple in-memory FIFO queue
- Populated by Scanner
- Processed by Processor (single-threaded, sequential)

**Processor behavior:**
- Dequeues one audiobook at a time
- Marks as 'processing' in SQLite
- Spawns child process: `audiobook-forge build --root <path>`
- Streams stdout/stderr to logs
- On success (exit code 0):
  - Mark as 'completed' in SQLite
  - Invoke Cleanup Handler
- On failure (non-zero exit):
  - Log error message
  - Mark as 'failed' in SQLite
  - Continue to next item (no retries)

**Why sequential?**
- Simpler coordination
- Avoids resource contention
- Users can enable parallel *file* encoding via audiobook-forge config
- Can add concurrency later if needed (YAGNI)

### 4. Cleanup Handler

**Purpose:** Manage original audio files after successful conversion

**Modes (via `CLEANUP_MODE` env var):**

- **`keep`** (default): Do nothing, leave originals in place
- **`delete`**: Remove all original audio files (MP3, M4A, AAC) from directory
- **`archive`**: Move original audio files to `ARCHIVE_DIR` (preserves directory structure)

**Implementation:**
- Only runs after confirmed successful conversion (M4B exists, exit code 0)
- Recursively finds all audio files in the audiobook directory
- Applies configured cleanup mode
- Logs cleanup actions

**Safety:**
- Never deletes/moves the M4B file
- Never deletes directories (only files)
- Logs all cleanup operations

---

## Configuration

### Environment Variables

The daemon uses minimal configuration - only orchestration settings. Audio quality/encoding settings are inherited from audiobook-forge's config file.

**Required:**
- `WATCH_DIR` - Directory to watch for audiobooks (e.g., `/audiobooks`)

**Optional:**
- `SCAN_INTERVAL` - Seconds between directory scans (default: `60`)
- `DATABASE_PATH` - SQLite database location (default: `/data/watcher.db`)
- `CLEANUP_MODE` - What to do with originals: `keep|delete|archive` (default: `keep`)
- `ARCHIVE_DIR` - Where to move files when `CLEANUP_MODE=archive` (default: `/data/archive`)
- `LOG_LEVEL` - Logging verbosity: `error|warn|info|debug|trace` (default: `info`)

**Audiobook-forge config:**
- Mount audiobook-forge config file at `~/.config/audiobook-forge/config.yaml`
- Or: Use audiobook-forge's default config if not mounted
- This controls quality, encoding, metadata, parallel file encoding, etc.

### Example Docker Compose

```yaml
version: '3.8'

services:
  audiobook-forge-watcher:
    image: audiobook-forge-watcher:latest
    container_name: audiobook-forge-watcher
    environment:
      WATCH_DIR: /audiobooks
      SCAN_INTERVAL: 60
      CLEANUP_MODE: keep
      LOG_LEVEL: info
    volumes:
      - /path/to/audiobooks:/audiobooks
      - /path/to/data:/data
      - /path/to/audiobook-forge-config.yaml:/root/.config/audiobook-forge/config.yaml:ro
    restart: unless-stopped
```

---

## Integration with Readarr/Chaptarr

**Expected directory structure:**
```
/audiobooks/
  ├── Author Name/
  │   ├── Book Title [ASIN]/
  │   │   ├── chapter01.mp3
  │   │   ├── chapter02.mp3
  │   │   └── cover.jpg
  │   └── Another Book/
  │       └── ...
  └── Another Author/
      └── ...
```

**Behavior:**
- Scanner recursively finds "Book Title [ASIN]/" directories
- Detects them as audiobooks (contains MP3 files, no M4B)
- Processes them in place
- Resulting structure:
```
/audiobooks/
  └── Author Name/
      └── Book Title [ASIN]/
          ├── Book Title.m4b          ← Created
          ├── chapter01.mp3            ← Kept/deleted/archived based on config
          ├── chapter02.mp3
          └── cover.jpg
```

**ASIN detection:**
- audiobook-forge already supports ASIN in folder names
- Use `--fetch-audible` flag if desired (via audiobook-forge config)
- Watcher doesn't need special logic for this

---

## Error Handling

**Philosophy:** Log and skip. Keep processing other books.

**Failure scenarios:**

1. **audiobook-forge invocation fails** (exit code != 0)
   - Log stderr output
   - Mark as 'failed' in SQLite
   - Continue to next book

2. **Directory becomes invalid during scan** (deleted, permissions)
   - Skip directory
   - Log warning

3. **SQLite operations fail**
   - Log error
   - Crash daemon (SQLite is critical, can't continue without state)

4. **Cleanup operations fail** (permissions, disk full)
   - Log error
   - Mark conversion as 'completed' anyway (M4B exists)
   - Don't crash (cleanup is non-critical)

**No retry logic:** Keep it simple. Users can manually re-trigger by deleting the M4B file.

---

## Validation Strategy

**How to identify "audiobooks":**

Use the same logic as audiobook-forge CLI:
- Directory contains 1+ audio files (MP3, M4A, AAC)
- Directory does NOT already contain an M4B file
- Directory is not already marked as 'completed' in SQLite

**Optional enhancement:** If audiobook-forge adds a `--dry-run` or `--validate` flag, use that to validate directories. Otherwise, use simple heuristic above.

---

## Technology Stack

**Language:** Rust (consistency with audiobook-forge)

**Key dependencies:**
- `tokio` - Async runtime for interval-based scanning
- `sqlx` - SQLite database operations
- `tracing` - Structured logging
- `serde` - Configuration/serialization
- `walkdir` - Recursive directory traversal

**Deployment:** Docker container with:
- audiobook-forge binary installed
- All audiobook-forge dependencies (ffmpeg, atomicparsley, gpac)
- Watcher daemon as entrypoint

---

## Future Enhancements (Out of Scope for v1)

These are explicitly **not** included in the initial implementation:

- **Statistics/API:** No HTTP endpoints, no metrics (can add later)
- **Web UI:** No web interface (can add later)
- **Retry logic:** No automatic retries on failure (can add later)
- **Inotify/FSEvents:** No native file watching (polling only for v1)
- **Concurrent processing:** No parallel book processing (sequential only for v1)
- **Pause/resume controls:** No runtime control (stop container to pause)

---

## Success Criteria

**v1 is successful if:**

1. ✅ Daemon runs continuously in Docker container
2. ✅ Discovers new audiobook directories recursively
3. ✅ Invokes audiobook-forge to convert them
4. ✅ Skips already-processed books (via M4B presence check + SQLite)
5. ✅ Handles cleanup according to `CLEANUP_MODE`
6. ✅ Logs all operations clearly
7. ✅ Continues processing on individual failures
8. ✅ Works with Readarr/Chaptarr directory structures

---

## Open Questions / Decisions Needed

None - all key decisions have been made during brainstorming:
- ✅ Separate repository
- ✅ SQLite for state tracking
- ✅ Configurable cleanup modes
- ✅ Sequential processing
- ✅ ENV vars for daemon config, inherit audiobook-forge config
- ✅ No statistics/API in v1
- ✅ Log and skip error handling

---

## Next Steps

1. Create new repository `audiobook-forge-watcher`
2. Set up Rust project structure
3. Implement in this order:
   - State Manager (SQLite)
   - Directory Scanner
   - Conversion Processor
   - Cleanup Handler
   - Daemon main loop
   - Dockerfile
   - Documentation

See implementation plan for detailed step-by-step tasks.
