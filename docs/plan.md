# hash-cull — Implementation Plan

Status: draft
Author: Ruanne Bornman

## 1. Goal

A command-line tool that indexes images and videos from one or more folders by
hashing their contents, persists those hashes in a local database, and then
detects duplicate media so the redundant copies can be removed safely.

The tool is built around three verbs:

1. **Index** — walk the media tree, hash every file, store the result.
2. **Dedupe** — group files by content, report duplicate groups, act on them.
3. **Undo** — reverse a removal from a recorded manifest/quarantine.

Correctness and safety come before raw speed: it is acceptable for the tool to be
slower, but it must never destroy unique data or the only copy of a file.

## 2. Scope

In scope:

- Recursive scanning of image and video files by extension and/or MIME sniffing.
- Content hashing of files (exact byte hashing).
- Optional perceptual (content) hashing of images, and video fingerprinting.
- Persistent local index so repeated runs are incremental.
- Duplicate detection and reporting.
- Safe removal: dry-run, quarantine, XDG trash, or hard delete.
- Reversible removal via an audit log / manifest.

Out of scope (initially):

- Any GUI.
- Cloud storage integration.
- Deduplication of audio, documents, archives (design should not preclude it).
- Cross-machine / shared central index.
- Rewriting or re-encoding media.

## 3. Language and stack

- **Language:** Python 3.11+ (rich ecosystem for media: Pillow, imagehash,
  ffmpeg bindings; good CLI/DB story). To be confirmed — see Questions.
- **Packaging:** `pyproject.toml`, installable console script `hash-cull`.
- **CLI:** `argparse` (stdlib, zero deps) unless a richer TUI is wanted later.
- **Database:** SQLite (stdlib `sqlite3`), WAL mode.
- **Exact hashing:** BLAKE3 (`blake3` package) — very fast, parallel-friendly;
  fallback to `hashlib.blake2b` if we want zero non-stdlib crypto deps.
- **Image perceptual hashing:** `Pillow` + `ImageHash` (phash/dhash/whash).
- **Video probing/frames:** `ffmpeg`/`ffprobe` via subprocess (system binary,
  not a Python dep) — extract frames for fingerprinting, probe duration.
- **Trash:** `Send2Trash` for XDG trash integration (optional extra).

## 4. High-level architecture

```
              +-----------+
   CLI -----> |  Scanner  |  walk dirs, filter, stat
              +-----+-----+
                    |
                    v
              +-----------+     (size pre-filter, skip unchanged)
              |  Hasher   | <-- worker pool (threads/processes)
              +-----+-----+
                    |
                    v
              +-----------+
              |   Store   |  SQLite: files, runs, groups
              +-----+-----+
                    |
                    v
              +-----------+
              | Comparator|  group by hash, apply policy
              +-----+-----+
                    |
                    v
              +-----------+     +----------------+
              | Reporter  |---->| Remover        |  dry-run / quarantine /
              +-----------+     | (manifest/log) |  trash / delete + undo
                                +----------------+
```

Each stage is a separate module so it can be unit-tested in isolation.

## 5. Data model (SQLite)

```sql
CREATE TABLE files (
    id         INTEGER PRIMARY KEY,
    path       TEXT    NOT NULL UNIQUE,
    size       INTEGER NOT NULL,
    mtime      REAL    NOT NULL,
    dev        INTEGER,            -- device id (hardlink detection)
    ino        INTEGER,            -- inode    (hardlink detection)
    kind       TEXT,               -- 'image' | 'video'
    exact_hash TEXT,               -- BLAKE3 of full contents
    phash      TEXT,               -- perceptual hash (nullable)
    hashed_at  REAL,               -- when hashed
    status     TEXT NOT NULL DEFAULT 'indexed'  -- indexed | error | missing
);

CREATE INDEX idx_files_exact ON files(exact_hash);
CREATE INDEX idx_files_phash ON files(phash);
CREATE INDEX idx_files_size  ON files(size);

CREATE TABLE runs (
    id          INTEGER PRIMARY KEY,
    started_at  REAL,
    finished_at REAL,
    root        TEXT,
    args_json   TEXT
);

CREATE TABLE groups (            -- persisted duplicate groups for reporting/undo
    id          INTEGER PRIMARY KEY,
    run_id      INTEGER REFERENCES runs(id),
    exact_hash  TEXT,
    member_ids  TEXT,            -- JSON array of file ids
    keep_id     INTEGER,         -- surviving file id
    action      TEXT             -- none | quarantine | trash | delete
);
```

Design notes:

- `path` is the natural key; a file is re-hashed when `size` or `mtime` changes.
- `exact_hash` is nullable so an interrupted run can resume.
- `dev`/`ino` let us distinguish true duplicates from **hardlinks** to the same
  physical file (see Concerns).
- Per-folder database (`.hash-cull.db` at the scan root) vs central
  (`$XDG_CACHE_HOME/hash-cull/`) is an open decision.

## 6. Hashing strategy

### 6.1 Exact hashing (primary, MVP)

Optimised, cheapest-first pipeline:

1. Stat every candidate file (size, mtime, dev, ino).
2. Skip files whose `(path, size, mtime)` already match the index.
3. **Size pre-filter:** a file whose size is unique within the scan cannot have
   an exact duplicate — record it as `indexed` with no hash or a cheap marker and
   never read its bytes. This avoids reading most of a large library.
4. For files sharing a size, compute the **full BLAKE3** hash, streamed in
   chunks (1–4 MiB), using a worker pool.

Optional faster variant (only if profiling shows the read cost dominating):
hash a *partial* signature first (size + first 64 KiB + last 64 KiB), and only
compute the full hash for files whose partial signature collides. Trade-off:
more false positives to resolve, fewer bytes read for large videos.

### 6.2 Perceptual hashing — images (phase 2)

- Decode each image (Pillow), compute a 64-bit perceptual hash (phash, optionally
  dhash/whash).
- Compare by **Hamming distance ≤ threshold** (default ~5–8 of 64 bits).
- Must be opt-in: it can produce false positives, and decoding is CPU-heavy.
- Especially relevant for decoded formats: a PNG re-saved as JPEG, a resized
  copy, a screenshot of a photo. Exact hashing misses all of these.

### 6.3 Video fingerprinting (phase 3, hardest)

- Exact byte hashing already catches true copies of videos. This stage targets
  *visually* equivalent videos at different encodings/bitrates.
- Approach: sample N frames at fixed positions via `ffmpeg`, compute a phash per
  frame, build a sequence signature; compare sequences with a distance metric
  tolerant to minor timing differences.
- This is expensive, alignment-sensitive, and prone to false positives on
  videos sharing intros/black frames. Treated as a research-y stretch goal.

## 7. CLI surface (proposed)

```
hash-cull index   PATH... [--db DB] [--jobs N] [--exclude GLOB]... 
                          [--min-size BYTES] [--follow-symlinks] [--rehash]
hash-cull report  [--db DB] [--json] [--group ID] [--summary]
hash-cull dedupe  [--db DB] [--perceptual] [--threshold N]
                          [--keep {newest,oldest,shortest-path,longest-path,first}]
                          [--quarantine DIR | --trash | --delete]
                          [--dry-run] [--yes]
hash-cull undo    MANIFEST
```

Behavioural contract:

- `index` and `report` are always read-only with respect to the media files.
- `dedupe` is **dry-run by default**; a removal target must be passed explicitly.
- `--yes` is required to skip the interactive confirmation for destructive modes.

## 8. Removal policy and safety

Ordered from safest to most dangerous; the tool should push users toward the top:

1. **Dry-run (default)** — print groups and what *would* happen; write nothing.
2. **Quarantine** — move duplicates into a user-specified directory, preserving
   relative structure. Fully reversible; recommended default action.
3. **XDG trash** — move to system trash via `send2trash`.
4. **Hard delete** — `os.remove`, only with `--delete --yes`.

Guarantees:

- Always keep **at least one** member per duplicate group (the `keep_id`).
- Choose the survivor using `--keep` policy (default: `oldest` mtime, i.e. the
  original capture is presumed most valuable). Ties broken by shortest path.
- Never follow symlinks unless `--follow-symlinks`; never delete a directory.
- Every action appended to an operation log (JSON lines) plus an `groups` row:
  `{timestamp, group_id, hash, action, from, to}` — the input to `undo`.
- `undo` replays a quarantine/trash manifest in reverse.
- Hardlink pairs (`dev`,`ino` equal) are reported but never treated as
  removable duplicates by default.

## 9. Performance

- Worker pool sized to CPU cores; hashing is I/O-bound → threads for pure reads,
  processes for image decode (CPU-bound).
- SQLite: WAL mode, batched inserts inside a single transaction per N files.
- Incremental indexing via `(path, size, mtime)`.
- Resumable: `status`/`exact_hash` columns allow an interrupted run to continue.
- Progress reporting to stderr (files/s, bytes read, ETA).

## 10. Testing

- Unit tests per module (scanner, hasher, store, comparator, remover).
- Fixture media generated in `tests/fixtures/` with Pillow and `ffmpeg`:
  - two byte-identical images / videos,
  - a re-encoded / resized image (exact miss, perceptual hit),
  - a hardlink pair,
  - a truncated/corrupt file,
  - a symlink to a file (must be skipped by default).
- Never run destructive paths in tests against real files; quarantine into a
  temp dir and assert `undo` restores the tree.
- A property test: after any random index+dedupe+undo cycle the filesystem is
  byte-identical to the starting state.

## 11. Milestones

- **M0 — Scaffolding.** `pyproject.toml`, package layout, `hash-cull` entry point,
  pytest, CI (lint + tests). No behaviour.
- **M1 — Index.** Scanner + size pre-filter + BLAKE3 exact hashing + SQLite store
  + `index` command. Incremental and resumable.
- **M2 — Detect & report.** Grouping, `report`, JSON output, human summary,
  `--summary` stats. Still read-only.
- **M3 — Remove & undo.** Dry-run, quarantine, trash, delete, manifest, `undo`,
  op log. The safety-critical milestone.
- **M4 — Perceptual images.** Optional phash pass with threshold and
  `--perceptual`; merged into grouping.
- **M5 — Video fingerprinting.** Frame sampling + sequence compare (stretch).
- **M6 — Polish.** Config file, exclude rules, packaging (AUR), docs.

Each milestone ends green: tests pass, and no destructive path is reachable
without an explicit flag.

## 12. Future ideas

- Config file (`~/.config/hash-cull/config.toml`) for default exclude globs,
  keep policy, db location.
- Audio/document support.
- "Similar but not identical" interactive review (burst photos, near-dup frames).
- AUR packaging (name availability already checked: `hash-cull` free).
- Dry-run HTML report (thumbnails side by side).

---

## Questions & Concerns

### Decisions needed from the maintainer

1. **Language/stack.** Confirm Python + Pillow/ffmpeg, or do you want something
   else (Rust/Go for a single static binary)? This shapes everything below.
2. **Exact vs perceptual by default.** Ship exact byte hashing first and add
   perceptual later, or is perceptual matching a day-one requirement?
3. **Video depth.** Videos are the hard part. Is byte-exact detection of videos
   enough for v1, with perceptual video fingerprinting deferred/optional?
4. **Default removal action.** Quarantine (reversible, my recommendation),
   XDG trash, or hard delete? What is the acceptable default?
5. **Keep policy default.** Oldest (assume original), newest, shortest path, or
   prompt per group?
6. **Database location.** Per-scan-root `.hash-cull.db` (portable, travels with
   the media) vs central `$XDG_CACHE_HOME` (single index across many roots)?
7. **Scale.** Roughly how many files / how much data is the target library? This
   decides concurrency, batching, and whether an on-disk index is even needed
   versus a streaming pass.

### Concerns and risks

- **False positives are catastrophic.** Perceptual hashing can flag distinct
  media as duplicates; any misuse combined with delete mode could destroy unique
  files. Mitigation: perceptual removals force quarantine, never hard delete.
- **Hardlinks.** Two paths, same `dev`+`ino` = one physical file. Deleting both
  paths loses the file; these must never be grouped as deletable duplicates.
  `rsync`/backup tools and package managers create these.
- **Symlinks and bind mounts.** Must not be followed by default; deleting through
  a symlink can remove an unrelated file. Watch for symlink loops.
- **Case-insensitive / network filesystems.** Path casing and `mtime` resolution
  differ; `(path,size,mtime)` incremental logic may re-hash unnecessarily or, in
  the worst case, miss changes. Possibly add optional content re-verify.
- **Metadata value.** Two byte-identical files can differ in xattrs, permissions,
  or which one sits in an "originals" folder. Keep-policy must be deliberate.
- **Capture-date / EXIF.** "Oldest mtime" ≠ oldest capture; EXIF
  `DateTimeOriginal` is more reliable when present. Do we parse it for the keep
  policy?
- **Live Photos / HEIC / RAW + sidecars.** A `.heic`+`.mov` pair or an image plus
  `.xmp` sidecar should arguably be treated as one unit; naive per-file dedupe
  could split a logical asset. Out of scope for v1 but must be stated.
- **Corrupt / partial files.** Unreadable files, changing files (still copying),
  zero-byte files, and permission errors need explicit handling; a file that
  changes mid-hash gives a bogus hash. Mitigation: skip zero-byte, re-stat after
  hashing and warn on change.
- **Partial-hash collision risk.** The size+head+tail optimization can produce
  false positives; the full hash must always be the final arbiter before any
  removal.
- **Undo durability.** If quarantine lives on the same disk and is later emptied,
  undo is impossible. Document retention; never auto-purge the quarantine.
- **Interrupted runs.** Crash mid-move could leave a file neither indexed nor in
  the manifest. Do atomic moves and fsync the log before moving.
- **Scope creep.** Perceptual image and especially video matching can balloon
  into a research project. Keep them strictly optional and off the critical path.
- **Packaging.** Console script name `hash-cull`; confirm the packaged name and
  whether to publish to the AUR (availability looked good: no official or AUR
  collisions).
