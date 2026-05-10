# Issue 4 & 5 Investigation Report

## Issue 4: Multiple Concurrent Messages Handling

**Claim:** Hermes can't handle multiple messages at once, or doesn't do it gracefully.

### Finding: **PARTIALLY TRUE — the claim is outdated.** The codebase now has extensive, mature concurrent message handling.

### Key Mechanisms

#### 1. Session-Level Lock (`_running_agents` dict)
**File:** `gateway/run.py`, lines 682-686

```python
self._running_agents: Dict[str, Any] = {}       # Key: session_key, Value: AIAgent instance
self._running_agents_ts: Dict[str, float] = {}  # start timestamp per session
self._pending_messages: Dict[str, str] = {}      # Queued messages during interrupt
self._busy_ack_ts: Dict[str, float] = {}         # last busy-ack timestamp per session (debounce)
self._session_run_generation: Dict[str, int] = {}
```

Each session key maps to either an AIAgent instance or the sentinel. A second message for the same session hits the guard and is **not dropped** — it's handled intelligently.

#### 2. Pending Sentinel (Race Condition Prevention)
**File:** `gateway/run.py`, lines 311-315

```python
_AGENT_PENDING_SENTINEL = object()
```

Placed into `_running_agents` **synchronously before any `await`** (line 3862):

```python
self._running_agents[_quick_key] = _AGENT_PENDING_SENTINEL
self._running_agents_ts[_quick_key] = time.time()
```

This prevents a second message from bypassing the guard during the async gap between the guard check and actual agent creation.

#### 3. The Guard Logic (What happens when a second message arrives)
**File:** `gateway/run.py`, lines 3298-3543

When `_quick_key in self._running_agents`, the handler checks:

| Condition | Action |
|-----------|--------|
| `/status` command | Returns status without interruption |
| `/stop` command | Force-interrupts + clears session lock |
| `/new`/`/reset` command | Interrupts agent, clears pending queue, dispatches reset |
| `/queue`/`/q` command | Queues prompt for next turn, does NOT interrupt |
| `/steer` command | Injects mid-run after next tool call |
| `/model` command | Rejected — "wait or /stop first" |
| `/approve`/`/deny` | Routes directly to approval handler (bypasses interrupt) |
| `MessageType.PHOTO` | Queued without interrupt (photo burst handling) |
| Telegram follow-up grace period (default 3s) | Queued without interrupt (avoids interrupting for rapid typing) |
| Agent in PENDING state | Queued with `merge_text=True` |
| `queue` mode (`_busy_input_mode == "queue"`) | Queued, no interrupt |
| Default (interrupt mode) | Calls `running_agent.interrupt(event.text)` |

#### 4. Pending Message Queue
**File:** `gateway/run.py`, lines 1490-1494

```python
def _queue_or_replace_pending_event(self, session_key: str, event: MessageEvent) -> None:
    adapter = self.adapters.get(event.source.platform)
    if not adapter:
        return
    merge_pending_message_event(adapter._pending_messages, session_key, event)
```

Messages are stored in `adapter._pending_messages` (a dict keyed by session_key) and dequeued after the current run finishes.

#### 5. Intelligent Message Merging
**File:** `gateway/platforms/base.py`, lines 874-924

`merge_pending_message_event()` intelligently handles:
- **Photo bursts**: Extends `media_urls` lists, merges captions
- **Text follow-ups with `merge_text=True`**: Appends text with newline separator
- **Media + text combinations**: Merges media and caption properly

```python
if existing_is_photo and incoming_is_photo:
    existing.media_urls.extend(event.media_urls)
    existing.media_types.extend(event.media_types)
    if event.text:
        existing.text = BasePlatformAdapter._merge_caption(existing.text, event.text)
```

#### 6. Busy Acknowledgment with Debounce
**File:** `gateway/run.py`, lines 1540-1593

Sends feedback every 30 seconds per session (cooldown via `_busy_ack_ts`). Includes iteration count, elapsed time, and current tool name.

```python
_BUSY_ACK_COOLDOWN = 30
now = time.time()
last_ack = self._busy_ack_ts.get(session_key, 0)
if now - last_ack < _BUSY_ACK_COOLDOWN:
    return True  # interrupt sent, ack already delivered recently
```

#### 7. Staleness Eviction
**File:** `gateway/run.py`, lines 3246-3296

Detects leaked locks from hung/crashed handlers using:
- `HERMES_AGENT_TIMEOUT` env var (default 1800s)
- Seconds since last activity (from agent's `get_activity_summary()`)
- Wall-clock age (10x timeout or 2h minimum)

#### 8. Two Busy Modes
**File:** `gateway/run.py`, line 655, 1528

```python
self._busy_input_mode = self._load_busy_input_mode()
```
- **`"interrupt"`** (default): New message interrupts the running agent
- **`"queue"`**: New messages are queued without interruption

#### 9. Thread-Safe Interrupt Mechanism
**File:** `tools/interrupt.py` (98 lines total)

Per-thread interrupt tracking so interrupting one agent session does not kill tools running in other sessions:

```python
_interrupted_threads: set[int] = set()
_lock = threading.Lock()

def set_interrupt(active: bool, thread_id: int | None = None) -> None:
    tid = thread_id if thread_id is not None else threading.current_thread().ident
    with _lock:
        if active:
            _interrupted_threads.add(tid)
        else:
            _interrupted_threads.discard(tid)
```

#### 10. Agent Cache (Performance)
**File:** `gateway/run.py`, lines 41-42, 698-700

```python
_AGENT_CACHE_MAX_SIZE = 128
_AGENT_CACHE_IDLE_TTL_SECS = 3600.0  # evict agents idle for >1h
```

AIAgent instances cached per-session with LRU eviction + idle TTL. Avoids rebuilding system prompt (memory, tools) on every message.

#### 11. Run Generation (Staleness Guard)
**File:** `gateway/run.py`, line 686, 8698-8710

`_session_run_generation` prevents older async runs from clobbering newer ones during cleanup:

```python
def _release_running_agent_state(self, session_key, *, run_generation=None):
    if run_generation is not None and not self._is_session_run_current(session_key, run_generation):
        return False  # older run, don't clobber
```

### Verdict on Issue 4
The claim **may have been true in an earlier version**, but the current codebase has extremely thorough concurrent message handling:
- Per-session locking with sentinel (race-condition-proof)
- Multiple modes (interrupt, queue)
- Intelligent photo burst handling
- Staleness eviction for crashed handlers
- Thread-safe interrupt isolation between sessions

---

## Issue 5: Config File Unfamiliarity

**Claim:** Hermes knows less about its own config than the user does.

### Finding: **LARGELY TRUE.** There is no formal config schema, no programmatic config key discovery, and the `/config` command is CLI-only.

### Key Findings

#### 1. DEFAULT_CONFIG — The Only "Schema" (Python Dict with Comments)
**File:** `hermes_cli/config.py`, lines 346-816+

The `DEFAULT_CONFIG` dict is the closest thing to a schema, but it's a Python dict with **inline comments** — not a formal schema:

```python
DEFAULT_CONFIG = {
    "model": "",
    "providers": {},
    "fallback_providers": [],
    "agent": {
        "max_turns": 90,
        # Inactivity timeout for gateway agent execution (seconds).
        # The agent can run indefinitely as long as it's actively calling
        # tools or receiving API responses.  Only fires when the agent has
        # been completely idle for this duration.  0 = unlimited.
        "gateway_timeout": 1800,
        ...
    },
    "terminal": { ... },
    "browser": { ... },
    ...
}
```

**Total size:** ~170KB, ~4130 lines. The DEFAULT_CONFIG alone spans ~470 lines of deeply nested dicts with inline comments.

#### 2. No Formal Config Schema
There is **no `config_schema.py`**, no JSON Schema file, no Pydantic model validating the full config structure.

The only structured schemas are:
- **`GatewayConfig`** in `gateway/config.py` (dataclass with typed fields)
- **`PlatformConfig`** in `gateway/config.py` (dataclass)
- **`SessionResetPolicy`** in `gateway/config.py` (dataclass)
- **`StreamingConfig`** in `gateway/config.py` (dataclass)
- **`SkinConfig`** in `hermes_cli/skin_engine.py` (dataclass)
- **Provider-level schemas** in plugins (e.g., `memory_provider.get_config_schema()`) — these have formal field definitions with `env_var`, `secret`, etc.

#### 3. CLI `/config` Command (CLI-Only, Not Available in Gateway)
**File:** `hermes_cli/commands.py`, lines 104-105

```python
CommandDef("config", "Show current configuration", "Configuration",
           cli_only=True),
```

The `cli_only=True` flag means the `/config` command is **not accessible through the messaging gateway** (Telegram, Discord, etc.). Gateway users cannot query config options through the agent.

Config subcommands (from config.py docstring):
- `hermes config` — Show current configuration
- `hermes config edit` — Open config in editor
- `hermes config set` — Set a specific value
- `hermes config wizard` — Re-run setup wizard

#### 4. Config Keys Lack Described Help Text
The `DEFAULT_CONFIG` dict uses Python `#` comments, not a format that can be queried programmatically. There is:

- **No `description` field** per config key
- **No help text** associated with keys
- **No way for the agent** to enumerate "what config options exist?"
- **No tool** that the agent can call to introspect config schema

#### 5. No Gateway `/config` Route
The `_handle_message()` method in `gateway/run.py` (line 3105+) dispatches many commands (`/help`, `/status`, `/model`, `/reasoning`, etc.) but has **no handler for `/config`** — it's simply absent from the gateway dispatch tree.

#### 6. Config Loading is Ad-Hoc
Various parts of the codebase load and parse config independently:

- **`_load_gateway_config()`** (run.py, line 511): Raw YAML load, returns `{}` on error
- **`load_config()`** (hermes_cli/config.py): Merges user config + defaults
- **`load_cli_config()`** (cli.py, line 299): CLI-specific loading with env var precedence
- **Various `_load_*()` methods** on `GatewayRunner`: `_load_busy_input_mode()`, `_load_reasoning_config()`, `_load_service_tier()`, `_load_fallback_model()`, `_load_provider_routing()`, `_load_show_reasoning()`, `_load_prefill_messages()` — each reads config keys independently

### Verdict on Issue 5
**The claim is confirmed.** The system lacks:
1. A formal, queryable config schema with key descriptions
2. Any gateway-accessible way to ask "what config options exist?"
3. A gateway `/config` command
4. Programmatic introspection of config keys with help text

The only semi-structured reference is the `DEFAULT_CONFIG` dict in `hermes_cli/config.py` (with inline comments), but this cannot be queried by the agent or by gateway users.
