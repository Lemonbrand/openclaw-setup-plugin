# Agent Patterns Reference

Common patterns for building cron agents. Each pattern includes the structure, CLI usage, and a starter implementation.

## Table of Contents
1. [Core Architecture](#core-architecture)
2. [Shared Library Pattern](#shared-library-pattern)
3. [Morning Brief](#morning-brief)
4. [Meeting Prep](#meeting-prep)
5. [Follow-Up Checker](#follow-up-checker)
6. [Content Digest](#content-digest)
7. [Health Check](#health-check)
8. [Database Schema](#database-schema)

## Core Architecture

Every agent follows the same structure:

```
1. Read context (files, database, CLIs)
2. Decide if action is needed
3. If yes: compose message, send via IPC
4. Log what happened
```

Agents are glue code. Business logic lives in `lib/`. Agents import from lib, read context, call functions, send results. Keep agents under 100 lines.

**CLI-first data access:** Agents fetch external data through CLI tools, not raw HTTP. The CLI handles authentication, pagination, rate limiting, and output formatting. The agent parses JSON output.

```python
# Good: CLI handles auth + formatting
result = subprocess.run(["gws", "calendar", "events", "list", ...], capture_output=True, text=True)
events = json.loads(result.stdout)

# Bad: raw HTTP with manual auth
response = requests.get("https://www.googleapis.com/calendar/v3/...",
                        headers={"Authorization": f"Bearer {token}"})
```

## Shared Library Pattern

```
lib/
  db.py          # All database operations
  ipc.py         # Message sending via OpenClaw
  entities.py    # Entity file CRUD
  calendar.py    # Calendar operations via gws CLI
```

### lib/ipc.py (Starter)

```python
"""Send messages via OpenClaw IPC."""
import json
import os
from datetime import datetime
from pathlib import Path

IPC_DIR = Path.home() / ".openclaw" / "data" / "ipc" / "main" / "messages"
CHAT_JID = os.environ.get("AGENT_CHAT_JID", "")

def send(text: str, chat_jid: str = ""):
    """Write a message to OpenClaw IPC queue."""
    IPC_DIR.mkdir(parents=True, exist_ok=True)
    msg = {
        "type": "message",
        "chatJid": chat_jid or CHAT_JID,
        "text": text
    }
    filename = f"agent-{datetime.now().strftime('%Y%m%d%H%M%S%f')}.json"
    (IPC_DIR / filename).write_text(json.dumps(msg))
```

### lib/calendar.py (Starter, uses gws CLI)

```python
"""Calendar operations via gws CLI (not raw Google API)."""
import json
import subprocess
from datetime import datetime, timedelta

def run_gws(args: list[str]) -> dict | None:
    """Run a gws command and return parsed JSON."""
    try:
        result = subprocess.run(
            ["gws"] + args,
            capture_output=True, text=True, timeout=30
        )
        if result.returncode == 0 and result.stdout.strip():
            return json.loads(result.stdout)
    except (subprocess.TimeoutExpired, json.JSONDecodeError):
        pass
    return None

def get_events(calendar_id: str = "primary", days: int = 1) -> list[dict]:
    """Get events for the next N days."""
    now = datetime.utcnow()
    time_min = now.isoformat() + "Z"
    time_max = (now + timedelta(days=days)).isoformat() + "Z"

    data = run_gws([
        "calendar", "events", "list",
        "--calendarId", calendar_id,
        "--timeMin", time_min,
        "--timeMax", time_max,
        "--singleEvents", "true",
        "--orderBy", "startTime"
    ])
    return data.get("items", []) if data else []

def get_event_attendees(event: dict) -> list[str]:
    """Extract attendee names from a calendar event."""
    return [
        a.get("displayName", a.get("email", "unknown"))
        for a in event.get("attendees", [])
    ]
```

### lib/entities.py (Starter)

```python
"""Entity file operations."""
from pathlib import Path

ENTITY_DIR = Path.home() / "agent-workspace" / "memory" / "entities" / "people"

def get_entity(name: str) -> str | None:
    """Read an entity file by person name (slug)."""
    slug = name.lower().replace(" ", "-")
    path = ENTITY_DIR / f"{slug}.md"
    return path.read_text() if path.exists() else None

def list_entities() -> list[str]:
    """List all entity file names."""
    if not ENTITY_DIR.exists():
        return []
    return [f.stem for f in ENTITY_DIR.glob("*.md")]
```

## Morning Brief

**Frequency:** Daily, weekday mornings (e.g., `0 8 * * 1-5`)
**CLIs used:** `gws` (calendar)
**Purpose:** Day overview with calendar context + entity enrichment

```python
#!/usr/bin/env python3
"""Morning brief agent."""
import sys
sys.path.insert(0, str(Path(__file__).parent.parent / "lib"))

from pathlib import Path
from datetime import datetime
from calendar import get_events, get_event_attendees
from entities import get_entity
from ipc import send

def main():
    now = datetime.now()
    lines = [f"MORNING BRIEF — {now.strftime('%A, %B %d')}\n"]

    events = get_events(days=1)
    if events:
        lines.append("TODAY:")
        for e in events:
            start = e.get("start", {}).get("dateTime", "all day")
            time_str = start[11:16] if "T" in start else "all day"
            summary = e.get("summary", "No title")
            lines.append(f"  {time_str}  {summary}")

            # Enrich with entity context
            for name in get_event_attendees(e):
                entity = get_entity(name)
                if entity:
                    # Extract first line of "Current Status" section
                    for line in entity.split("\n"):
                        if line.startswith("## Current Status"):
                            idx = entity.split("\n").index(line)
                            status = entity.split("\n")[idx + 1:idx + 3]
                            lines.append(f"    [{name}]: {' '.join(status).strip()}")
                            break
        lines.append("")

    # Check suggestions queue
    suggestions = Path.home() / "agent-workspace" / "memory" / "queue" / "suggestions.md"
    if suggestions.exists():
        content = suggestions.read_text().strip()
        if content:
            first_item = content.split("\n")[0]
            lines.append(f"SUGGESTED: {first_item}")
            lines.append("")

    lines.append("What's the priority today?")
    send("\n".join(lines))

if __name__ == "__main__":
    main()
```

## Meeting Prep

**Frequency:** Every 15 minutes (`*/15 * * * *`)
**CLIs used:** `gws` (calendar)
**Purpose:** T-60 minute prep card for upcoming meetings

```python
#!/usr/bin/env python3
"""Meeting prep agent. Fires 60 min before calendar events."""
from datetime import datetime, timedelta
from calendar import get_events, get_event_attendees
from entities import get_entity
from ipc import send

PREP_WINDOW_MINUTES = 60

def main():
    now = datetime.utcnow()
    events = get_events(days=1)

    for e in events:
        start_str = e.get("start", {}).get("dateTime")
        if not start_str:
            continue

        event_time = datetime.fromisoformat(start_str.replace("Z", "+00:00"))
        minutes_until = (event_time.replace(tzinfo=None) - now).total_seconds() / 60

        if 45 <= minutes_until <= 75:  # Within prep window
            summary = e.get("summary", "Meeting")
            attendees = get_event_attendees(e)

            lines = [f"PREP: {summary} in {int(minutes_until)} minutes\n"]

            for name in attendees:
                entity = get_entity(name)
                if entity:
                    lines.append(f"WHO: {name}")
                    # Extract key sections
                    for section in ["How They Think", "Current Status", "Watch Items"]:
                        if f"## {section}" in entity:
                            idx = entity.index(f"## {section}")
                            content = entity[idx:].split("##")[0].strip()
                            lines.append(content[:200])
                    lines.append("")

            send("\n".join(lines))

if __name__ == "__main__":
    main()
```

## Follow-Up Checker

**Frequency:** Daily (`0 10 * * *`)
**CLIs used:** None (reads local files)
**Purpose:** Find stale relationships and overdue items

```python
#!/usr/bin/env python3
"""Follow-up checker. Flags overdue items and stale relationships."""
import re
from datetime import datetime, timedelta
from entities import list_entities, get_entity
from ipc import send

STALE_DAYS = 7

def extract_date(text: str) -> datetime | None:
    """Find the most recent date in text (YYYY-MM-DD format)."""
    dates = re.findall(r'\d{4}-\d{2}-\d{2}', text)
    if dates:
        return max(datetime.strptime(d, "%Y-%m-%d") for d in dates)
    return None

def main():
    now = datetime.now()
    overdue = []
    stale = []

    for name in list_entities():
        entity = get_entity(name)
        if not entity:
            continue

        last_date = extract_date(entity)
        if last_date:
            days_ago = (now - last_date).days
            if days_ago > STALE_DAYS:
                stale.append((name, days_ago))

    if overdue or stale:
        lines = [f"FOLLOW-UP CHECK — {now.strftime('%B %d')}\n"]

        if stale:
            lines.append("GOING STALE:")
            for name, days in sorted(stale, key=lambda x: -x[1]):
                lines.append(f"  - {name}: no activity for {days} days")

        send("\n".join(lines))

if __name__ == "__main__":
    main()
```

## Content Digest

**Frequency:** 3x/week (`0 9 * * 1,3,5`)
**CLIs used:** `gws` (gmail)
**Purpose:** Pull newsletter content, extract hooks

```python
#!/usr/bin/env python3
"""Content digest. Fetches newsletters via gws Gmail CLI."""
import json
import subprocess

def get_newsletters(label: str = "AI Newsletter", max_results: int = 10):
    """Fetch recent emails from a Gmail label via gws CLI."""
    result = subprocess.run(
        ["gws", "gmail", "users", "messages", "list",
         "--userId", "me",
         "--q", f"label:{label} newer_than:2d",
         "--maxResults", str(max_results)],
        capture_output=True, text=True, timeout=30
    )
    if result.returncode != 0:
        return []
    return json.loads(result.stdout).get("messages", [])

# ... fetch each message, extract subject/snippet, compose digest
```

## Health Check

**Frequency:** Every 30 minutes (`*/30 * * * *`)
**CLIs used:** `doctl` (optional, for droplet metrics)
**Purpose:** Alert if agents are failing

```python
#!/usr/bin/env python3
"""Health check. Monitors agent logs for errors."""
from pathlib import Path
from datetime import datetime, timedelta
from ipc import send

LOG_DIR = Path.home() / "logs"

def check_agent_health():
    """Check log files for recent errors or missing output."""
    issues = []

    for log_file in LOG_DIR.glob("*.log"):
        if not log_file.exists():
            continue

        content = log_file.read_text()
        lines = content.strip().split("\n")

        # Check for recent errors
        for line in lines[-20:]:  # Last 20 lines
            if "ERROR" in line or "Traceback" in line:
                issues.append(f"{log_file.stem}: error detected")
                break

        # Check for stale logs (no output in 24h for daily agents)
        if log_file.stat().st_mtime < (datetime.now() - timedelta(hours=24)).timestamp():
            issues.append(f"{log_file.stem}: no output in 24h")

    return issues

def main():
    issues = check_agent_health()
    if issues:
        msg = "HEALTH CHECK ALERT\n\n" + "\n".join(f"- {i}" for i in issues)
        send(msg)

if __name__ == "__main__":
    main()
```

## Database Schema

For users ready to graduate from flat files to SQLite:

```sql
-- Starter schema for agent data

-- Pipeline: track prospects and client stages
CREATE TABLE IF NOT EXISTS pipeline (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    contact_name TEXT NOT NULL,
    contact_email TEXT,
    entity_file TEXT,
    stage TEXT DEFAULT 'new',  -- new, booked, had_call, follow_up, won, dormant
    next_action TEXT,
    next_action_date TEXT,
    created_at TEXT DEFAULT (datetime('now')),
    updated_at TEXT DEFAULT (datetime('now'))
);

-- Agent tasks: queue for async work
CREATE TABLE IF NOT EXISTS agent_tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    task_type TEXT NOT NULL,
    prompt TEXT,
    status TEXT DEFAULT 'pending',  -- pending, running, completed, failed
    scheduled_for TEXT,
    completed_at TEXT,
    result TEXT,
    created_at TEXT DEFAULT (datetime('now'))
);

-- Ops log: audit trail for every agent action
CREATE TABLE IF NOT EXISTS ops_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT DEFAULT (datetime('now')),
    script TEXT NOT NULL,
    action_type TEXT,
    target TEXT,
    result TEXT,
    error_message TEXT
);
```

Use SQLite with WAL mode for concurrent access from multiple cron agents:

```python
import sqlite3

def get_connection(db_path: str = "~/agent-workspace/agent.db"):
    conn = sqlite3.connect(os.path.expanduser(db_path))
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA foreign_keys=ON")
    conn.row_factory = sqlite3.Row
    return conn
```
