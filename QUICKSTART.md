<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright (C) 2026 Starshard contributors -->

# Quickstart: Self-Hosting Your Starshard Instance

This guide walks you from zero to a running personal memory hub. Target audience: someone who has used Claude Code before and can navigate a terminal.

Time estimate: 30–45 minutes for Phase 0 (minimum viable hub). An hour or two if you haven't used Cloudflare before.

---

## Prerequisites

You need four things before starting:

**1. A machine that stays on.** This is where your hub will run. Options: a Mac mini, a spare laptop that lives under your desk, a Raspberry Pi 4, or a cheap cloud VPS ($5/month). It needs to be online when you want to use the hub.

**2. A domain name.** About $10/year from Namecheap, Porkbun, or Cloudflare Registrar. You'll set the DNS to Cloudflare. If you already have a domain on Cloudflare, you can add a subdomain.

**3. A free Cloudflare account.** At cloudflare.com. The free tier covers everything in this guide. Cloudflare handles routing (Cloudflare Tunnel) and authentication (Cloudflare Access).

**4. Python 3.11 or later.** Check: `python3 --version`. On modern macOS this is pre-installed. On Ubuntu: `sudo apt install python3.11 python3.11-venv`.

You do not need Docker. You do not need Kubernetes. You do not need a cloud provider account (unless you're using a VPS).

---

## What You're Building

Three services, all running on your machine:

| Service | What it does | Technology |
|---------|-------------|------------|
| Memory Hub | Stores and serves memory records via MCP-over-HTTP | FastAPI + SQLite |
| Cloudflare Tunnel | Routes traffic from your domain to localhost | cloudflared |
| Cloudflare Access | Authenticates you (email OTP) before letting traffic through | Cloudflare dashboard |

Total disk footprint: ~50 MB (excluding your memories). Total cost if on existing hardware: $0/month.

---

## Phase 0 — Minimum Viable Hub

### Step 1: Install cloudflared

**macOS (with Homebrew)**:
```bash
brew install cloudflare/cloudflare/cloudflared
```

**macOS (without Homebrew)**:
```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-darwin-arm64 \
  -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared
```
(Replace `darwin-arm64` with `darwin-amd64` for Intel Mac.)

**Linux (x86_64)**:
```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 \
  -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared
```

Verify: `cloudflared --version`

### Step 2: Create hub directory and database

```bash
mkdir -p ~/.starshard/data ~/.starshard/logs
```

Save the following as `~/.starshard/init_db.py` and run it:

```python
#!/usr/bin/env python3
import sqlite3, pathlib

db_path = pathlib.Path.home() / ".starshard" / "data" / "memories.db"
conn = sqlite3.connect(db_path)
conn.executescript("""
    CREATE TABLE IF NOT EXISTS memories (
        mem_id      TEXT PRIMARY KEY,
        type        TEXT NOT NULL CHECK (type IN ('episodic', 'semantic', 'procedural')),
        tags        TEXT NOT NULL DEFAULT '[]',
        summary     TEXT NOT NULL DEFAULT '',
        content     TEXT NOT NULL DEFAULT '',
        provenance  TEXT NOT NULL DEFAULT '{}',
        archived    INTEGER NOT NULL DEFAULT 0,
        created_at  TEXT NOT NULL,
        updated_at  TEXT NOT NULL
    );
    CREATE INDEX IF NOT EXISTS idx_type     ON memories(type);
    CREATE INDEX IF NOT EXISTS idx_archived ON memories(archived);
    CREATE VIRTUAL TABLE IF NOT EXISTS memories_fts USING fts5(
        mem_id UNINDEXED, summary, content, tags,
        content=memories, content_rowid=rowid
    );
""")
conn.commit()
conn.close()
print(f"Database initialized: {db_path}")
```

```bash
python3 ~/.starshard/init_db.py
```

### Step 3: Install Python dependencies

```bash
cd ~/.starshard
python3 -m venv venv
venv/bin/pip install fastapi uvicorn mcp pydantic "python-ulid"
```

### Step 4: Get the hub server

The full reference implementation is in the Starshard repository at `src/memory_hub/`. For a quick smoke test, save this minimal server as `~/.starshard/server.py`:

```python
#!/usr/bin/env python3
# Minimal Starshard hub — use src/memory_hub/ for production
import json, sqlite3, datetime, pathlib
from fastapi import FastAPI, HTTPException

app = FastAPI()
DB = pathlib.Path.home() / ".starshard/data/memories.db"

def get_db():
    conn = sqlite3.connect(str(DB))
    conn.row_factory = sqlite3.Row
    return conn

@app.get("/health")
def health():
    return {"status": "ok", "db": str(DB)}

@app.post("/mcp")
async def mcp_handler(body: dict):
    tool = body.get("tool")
    params = body.get("params", {})

    if tool == "search_memories":
        query = params.get("query", "")
        limit = params.get("limit", 20)
        with get_db() as conn:
            rows = conn.execute(
                "SELECT mem_id, type, tags, summary, created_at FROM memories "
                "WHERE archived=0 AND (summary LIKE ? OR content LIKE ?) LIMIT ?",
                (f"%{query}%", f"%{query}%", limit)
            ).fetchall()
        return {"memories": [dict(r) for r in rows]}

    if tool == "get_memory":
        mem_id = params.get("mem_id")
        with get_db() as conn:
            row = conn.execute(
                "SELECT * FROM memories WHERE mem_id=? AND archived=0", (mem_id,)
            ).fetchone()
        if not row:
            raise HTTPException(status_code=404, detail="Memory not found")
        return dict(row)

    if tool == "create_memory":
        import ulid as ulid_lib
        mem_id = str(ulid_lib.new())
        now = datetime.datetime.utcnow().isoformat() + "Z"
        with get_db() as conn:
            conn.execute(
                "INSERT INTO memories VALUES (?,?,?,?,?,?,0,?,?)",
                (
                    mem_id,
                    params.get("type", "episodic"),
                    json.dumps(params.get("tags", [])),
                    params.get("summary", ""),
                    params.get("content", ""),
                    json.dumps(params.get("provenance", {"source": "api"})),
                    now, now,
                )
            )
        return {"mem_id": mem_id, "created": True}

    if tool == "list_memories":
        limit = params.get("limit", 50)
        with get_db() as conn:
            rows = conn.execute(
                "SELECT mem_id, type, tags, summary, created_at FROM memories "
                "WHERE archived=0 ORDER BY created_at DESC LIMIT ?",
                (limit,)
            ).fetchall()
        return {"memories": [dict(r) for r in rows]}

    return {"error": f"Unknown tool: {tool}"}
```

Start it:
```bash
~/.starshard/venv/bin/uvicorn server:app \
  --host 127.0.0.1 --port 8792 --app-dir ~/.starshard --log-level info &
```

Check it's running:
```bash
curl http://localhost:8792/health
# {"status":"ok","db":"/home/you/.starshard/data/memories.db"}
```

### Step 5: Set up Cloudflare Tunnel

```bash
cloudflared tunnel login
# Opens browser window. Authenticate with your Cloudflare account.

cloudflared tunnel create starshard-hub
# Note the Tunnel ID in the output (a UUID).
```

Create `~/.cloudflared/config.yml`:
```yaml
tunnel: YOUR-TUNNEL-ID-HERE
credentials-file: /home/YOUR_USERNAME/.cloudflared/YOUR-TUNNEL-ID.json

ingress:
  - hostname: hub.yourdomain.com
    service: http://localhost:8792
  - service: http_status:404
```

Create the DNS record:
```bash
cloudflared tunnel route dns starshard-hub hub.yourdomain.com
```

Start the tunnel:
```bash
cloudflared tunnel run starshard-hub &
```

### Step 6: Set up Cloudflare Access (authentication)

Without this step, your hub is publicly accessible. Do this before testing from outside your local network.

1. Log into the Cloudflare dashboard (dash.cloudflare.com)
2. Navigate to: **Zero Trust** → **Access** → **Applications**
3. Click **Add an Application** → **Self-Hosted**
4. Fill in: Application name: `Starshard Hub`, Session duration: 24 hours, Application domain: `hub.yourdomain.com`
5. Configure the policy: Action: Allow, Rule type: Emails, Value: your email address
6. Save

Now visiting `https://hub.yourdomain.com` requires a one-time passcode sent to your email. Sessions last 24 hours.

### Step 7: Connect Claude Code

Add to `~/.claude/mcp.json` (create if it doesn't exist):

```json
{
  "mcpServers": {
    "starshard-hub": {
      "type": "http",
      "url": "https://hub.yourdomain.com/mcp",
      "description": "My personal Starshard memory hub"
    }
  }
}
```

Restart Claude Code. In a new session, try:
```
Save a memory: today I set up my Starshard hub. Type: episodic. Tags: setup, milestone.
```

Verify it worked:
```bash
curl -s http://localhost:8792/mcp \
  -H "Content-Type: application/json" \
  -d '{"tool":"search_memories","params":{"query":"Starshard hub"}}' \
  | python3 -m json.tool
```

**You now have a working personal memory hub.**

---

## Make It Persistent

Without persistence, your hub stops when you close the terminal.

### macOS — launchd

```bash
# Replace YOUR_USERNAME and paths accordingly
cat > ~/Library/LaunchAgents/com.starshard.hub.plist << 'PLIST'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.starshard.hub</string>
  <key>ProgramArguments</key>
  <array>
    <string>/Users/YOUR_USERNAME/.starshard/venv/bin/uvicorn</string>
    <string>server:app</string>
    <string>--host</string><string>127.0.0.1</string>
    <string>--port</string><string>8792</string>
    <string>--app-dir</string><string>/Users/YOUR_USERNAME/.starshard</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/Users/YOUR_USERNAME/.starshard/logs/hub.log</string>
  <key>StandardErrorPath</key><string>/Users/YOUR_USERNAME/.starshard/logs/hub.log</string>
</dict>
</plist>
PLIST

cat > ~/Library/LaunchAgents/com.starshard.tunnel.plist << 'PLIST'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.starshard.tunnel</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/cloudflared</string>
    <string>tunnel</string><string>run</string><string>starshard-hub</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/Users/YOUR_USERNAME/.starshard/logs/tunnel.log</string>
  <key>StandardErrorPath</key><string>/Users/YOUR_USERNAME/.starshard/logs/tunnel.log</string>
</dict>
</plist>
PLIST

launchctl load ~/Library/LaunchAgents/com.starshard.hub.plist
launchctl load ~/Library/LaunchAgents/com.starshard.tunnel.plist
```

### Linux — systemd

```bash
sudo tee /etc/systemd/system/starshard-hub.service << 'UNIT'
[Unit]
Description=Starshard Memory Hub
After=network.target

[Service]
Type=simple
User=YOUR_USERNAME
WorkingDirectory=/home/YOUR_USERNAME/.starshard
ExecStart=/home/YOUR_USERNAME/.starshard/venv/bin/uvicorn \
    server:app --host 127.0.0.1 --port 8792 --app-dir /home/YOUR_USERNAME/.starshard
Restart=always
RestartSec=5
StandardOutput=append:/home/YOUR_USERNAME/.starshard/logs/hub.log
StandardError=append:/home/YOUR_USERNAME/.starshard/logs/hub.log

[Install]
WantedBy=multi-user.target
UNIT

sudo tee /etc/systemd/system/starshard-tunnel.service << 'UNIT'
[Unit]
Description=Starshard Cloudflare Tunnel
After=network.target

[Service]
Type=simple
User=YOUR_USERNAME
ExecStart=/usr/local/bin/cloudflared tunnel run starshard-hub
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
UNIT

sudo systemctl daemon-reload
sudo systemctl enable --now starshard-hub starshard-tunnel
```

---

## Phase 1 — Add Task Dispatch

Once the hub is running, you can add the task dispatch layer: a poller daemon that picks up `status-pending` task memories and routes them to Claude Code for execution.

**How it works**:
1. You (or an agent) write a task memory with tags `["status-pending", "assignee-<executor-name>"]`
2. The poller on that executor queries for these every 30 seconds
3. When it finds one, it updates to `status-in-progress` and spawns a Claude Code session
4. The session executes the task and writes results back
5. The original task is updated to `status-done`

The poller reference implementation is in `src/poller/` in the Starshard repository.

---

## Phase 2 — Add Mirror Consolidation

The Mirror runs offline and produces proposals to improve your memory corpus. Minimal setup: a cron entry or Claude.ai scheduled routine that runs weekly:

```
Every Sunday at 2am:
  search_memories(limit=200, archived=0, created_after=7-days-ago)
  → apply C1 (dedup) + C2 (contradiction) + C3 (compression)
  → write proposals with tag mirror-proposal
  → notify me for review
```

The Mirror never modifies production memories directly. Every proposal requires your approval. See ARCHITECTURE.md §4 for the full 7-pass spec.

---

## Deployment Modes

| Mode | Hardware | Monthly cost | Best for |
|------|----------|-------------|---------|
| Mac mini (home) | M2 Mac mini | $0/mo | Always-on, fast |
| Spare Linux box | Any x86_64 | $0/mo | Same |
| Hetzner CX22 | VPS in EU | ~€4/mo | Best cloud price |
| AWS t4g.micro | VPS | $0 (1yr free tier) | AWS ecosystem |
| Raspberry Pi 4 | ARM Linux | $0/mo + ~$80 hardware | Low power |

---

## Gotchas

**SQLite on NFS**: Don't. SQLite's WAL mode does not work correctly on network filesystems. Use local disk only.

**Cloudflare Access vs Zero Trust naming**: Same product, renamed. Steps above apply to both.

**MCP SSE vs HTTP**: Starshard uses MCP-over-HTTP (not SSE). HTTP is simpler for self-hosting — no long-lived persistent connections through the tunnel.

**Claude Code headless access**: For Claude Code to access your hub without browser OTP, create a Cloudflare Access Service Token (Zero Trust → Access → Service Auth → Service Tokens → Create). Add to `~/.claude/mcp.json`:

```json
{
  "mcpServers": {
    "starshard-hub": {
      "type": "http",
      "url": "https://hub.yourdomain.com/mcp",
      "headers": {
        "CF-Access-Client-Id": "your-client-id.access",
        "CF-Access-Client-Secret": "your-client-secret"
      }
    }
  }
}
```

**Port conflicts**: If port 8792 is in use, change it consistently in the uvicorn command, the service file, and `~/.cloudflared/config.yml`.

**Database backup**: `sqlite3 ~/.starshard/data/memories.db ".backup ~/.starshard/backups/memories-$(date +%Y%m%d).db"` — add to cron for daily backups.

---

## Security Defaults

1. **One token per executor**: If you add multiple executors, give each its own Cloudflare Access Service Token. Revoke compromised tokens individually.

2. **Tag external input**: When writing memories from external sources, set `provenance.source = "external"`. The hub auto-applies `external-user-input` tag (see SAFETY-CHARTER.md HM-3).

3. **Rate limit your hub**: Add rate limiting (60 writes/minute per client IP) to prevent runaway agent loops.

4. **Protect tunnel credentials**: `chmod 600 ~/.cloudflared/*.json` — don't let tunnel credential files be world-readable.

5. **Monitor logs**: `tail -f ~/.starshard/logs/hub.log` occasionally. Unexpected write volume (high count, unusual agent_ids) is the first sign of a problem.
