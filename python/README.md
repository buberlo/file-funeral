# File Funeral Python Service

`funeral.py` is the local scanning and mausoleum service for File Funeral. It is intentionally self-contained: it uses only the Python standard library, reads no file contents by default, and communicates with Electron through simple command-line invocations and JSON output.

## Running the service

Use Python 3.9 or newer.

```bash
python3 funeral.py scan ~/Documents --stale-days 45
```

All commands print exactly one JSON object to `stdout`. Human-readable diagnostics, warnings, and tracebacks go to `stderr`.

Exit codes:

- `0` — success
- `1` — runtime error, such as a missing file or unreadable folder
- `2` — usage error, such as an unknown command or bad flag

## Command protocol

### `scan`

Scan a folder for stale files and generate metadata-based life stories.

```bash
python3 funeral.py scan <folder> [--stale-days 30] [--include-hidden] [--max-files 500]
```

Example output:

```json
{
  "ok": true,
  "command": "scan",
  "folder": "/Users/example/Documents",
  "stale_days": 30,
  "count": 1,
  "files": [
    {
      "file_id": "sha1-of-absolute-path",
      "path": "/Users/example/Documents/old-report.txt",
      "name": "old-report.txt",
      "extension": ".txt",
      "size": 4821,
      "created": "2024-01-02T09:14:00+00:00",
      "modified": "2024-03-18T16:42:11+00:00",
      "accessed": "2024-03-18T16:42:11+00:00",
      "last_touched": "2024-03-18T16:42:11+00:00",
      "age_days": 120,
      "story": "Born in January, last touched in March, and quiet for 120 days."
    }
  ]
}
```

Scan behavior:

- Files are stale when their latest meaningful touch time is older than `--stale-days`.
- The latest meaningful touch time is the maximum available value of `modified` and `accessed`.
- Directories, symbolic links, and hidden files are skipped unless `--include-hidden` is provided.
- `--max-files` stops the walk after that many stale files have been collected.
- `file_id` is a stable hash of the normalized absolute path. It is not the mausoleum ID.

### `bury`

Move one file into the mausoleum and record its epitaph.

```bash
python3 funeral.py bury <path> [--epitaph "text"]
```

Example output:

```json
{
  "ok": true,
  "command": "bury",
  "mausoleum_id": "8f2c1b6e-1a2b-4c3d-9e4f-5a6b7c8d9e0f",
  "path": "/Users/example/Documents/old-report.txt",
  "name": "old-report.txt",
  "epitaph": "Thanks for the drafts.",
  "story": "Born in January, last touched in March, and quiet for 120 days.",
  "mausoleum_path": "~/.file-funeral/mausoleum/8f2c1b6e-1a2b-4c3d-9e4f-5a6b7c8d9e0f.file"
}
```

Bury behavior:

- The original file is moved into the mausoleum.
- A metadata sidecar is written next to the stored file.
- If the file no longer exists, the command fails with a JSON error object.
- The returned `mausoleum_id` is the only ID needed for restore or forget.

### `restore`

Restore a buried file from the mausoleum.

```bash
python3 funeral.py restore <mausoleum_id> [--target <folder>] [--force]
```

Example output:

```json
{
  "ok": true,
  "command": "restore",
  "mausoleum_id": "8f2c1b6e-1a2b-4c3d-9e4f-5a6b7c8d9e0f",
  "original_path": "/Users/example/Documents/old-report.txt",
  "restored_path": "/Users/example/Documents/old-report.txt",
  "conflict": false
}
```

Restore behavior:

- If the original path is available, the file is restored there.
- If the original path is occupied and `--force` is not set, the command reports a conflict without overwriting.
- If the original folder no longer exists, the file is restored to `--target` when provided, otherwise