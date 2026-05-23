---
name: tcp-bridge-agent-protocol
description: One standard for AI agents to talk to each other across machines. TCP bridge with JSON+token auth. Stop inventing new protocols every session.
version: 1.0.0
author: nerudek
compatible-with: claude-code, openclaw, hermes-agent, kimi-code
tags: [tcp, bridge, agent-communication, multi-agent, protocol, tailscale]
---

# TCP Bridge Agent Protocol — One Standard for Cross-Machine AI Agent Communication

## Problem

**Every AI agent session starts with the same question: "How do I talk to the other agents?"**

Vox on Mac Mini M4 needs to tell Kimi on MacBook M2 that configs are deployed. Claude needs to ask Goose on Kubuntu about GPU status. OpenClaw needs to forward a WhatsApp message to Vox. Every single time, the agent wastes 5-10 minutes discovering which port, which protocol, which token, which format to use.

This isn't just inefficient — it's **destructive:**

- **Session fragmentation.** Agent A tries TCP port 17423. Timeout. Tries HTTP port 17420. Wrong token. Tries relay port 17426. Connects but can't send structured commands. By the time it finds a working path, the context window is 40% consumed and the original task is forgotten.
- **Protocol proliferation.** Over 6 months, our ecosystem accumulated: Bridge v1, Bridge v2, Bridge v3, Receiver v2, Receiver v3, Receiver v5, Relay server, Relay client, NATS bridge, WhatsApp bridge, Obsidian bridge. Each with different auth, different message formats, different error handling. Agents couldn't keep up.
- **Authentication chaos.** Some protocols used `token` in JSON body. Others used HTTP headers. Others had no auth at all. Agents tried all combinations. "Invalid token" became the most common error message in our logs.
- **Asymmetric communication.** M2 could send commands to M4 (receiver :17420). But M4 couldn't send commands back to M2. Communication was one-way. When Vox needed Kimi to run a command, there was no standard path.
- **Bootstrap paradox.** To discover the protocol, the agent needed to read a file. To know which file to read, the agent needed to already know the protocol. New agents were stuck.

**The specific failure:** During the V3 deployment, Vox needed to verify that configs were correctly copied to M2. Three different protocols were tried: NATS pub (no response mechanism), HTTP ping (timeout), TCP relay (connected but unstructured). Total time wasted: 25 minutes. The fix took 2 minutes once the bridge was used with the correct token.

**Why existing approaches fail:**

- **MCP (Model Context Protocol):** Anthropic's standard for tool calling. Excellent for agent-to-tool communication. Not designed for agent-to-agent command execution across machines.
- **A2A (Google Agent-to-Agent):** A reference specification, not an implementation. No ready-to-use bridge, no auth mechanism, no shell command execution.
- **NATS alone:** Pub/sub is great for events and state. But it doesn't provide synchronous request-response for shell commands. "Run this and give me the output" needs a different pattern.
- **SSH:** Requires key management per agent, doesn't work well with Tailscale IPs that change, and doesn't provide structured JSON responses.

## Solution

**A single TCP bridge protocol. One port. One token. One JSON format. Symmetric.**

Every machine runs a bridge server on `:17423`. Every agent uses the same client to send commands. The protocol is dead simple:

```
Client connects → sends {"token":"...", "cmd":"shell command"} → server executes → returns {"ok":true, "out":"...", "exit":0}
```

That's it. No versioning, no content negotiation, no streaming, no keep-alive. One request, one response, disconnect. Stateless.

### Architecture

```
┌─────────────────────┐         ┌─────────────────────┐
│   Mac Mini M4       │         │   MacBook M2        │
│                     │         │                     │
│  bridge-to-m2.py ───┼─TCP:17423──→ bridge server    │
│  (client)           │         │  (executes commands) │
│                     │         │                     │
│  bridge server     ←──TCP:17423── bridge-to-m4.py   │
│  (executes commands)│         │  (client)           │
└─────────────────────┘         └─────────────────────┘
         │                               │
         └─────── Tailscale VPN ─────────┘
              (100.95.129.85)    (100.105.185.60)
```

### Protocol Specification

**Connection:** TCP, port 17423
**Authentication:** JSON field `"token"` with value `nexus-macmini-hermes-2026`
**Request format:** `{"token":"...", "cmd":"shell command"}\n`
**Response format:** `{"ok":true, "out":"stdout text", "exit":0}` or `{"ok":false, "error":"message"}`
**Timeout:** 10 seconds
**Encoding:** UTF-8

### Usage

**M4 → M2 (send a command):**
```bash
python3 ~/agentos/bridge-to-m2.py "whoami"
# → {"ok": true, "out": "nerudek\n", "exit": 0}
```

**M2 → M4 (send a command):**
```bash
curl -X POST http://100.95.129.85:17423/exec \
  -H "Content-Type: application/json" \
  -d '{"token":"nexus-macmini-hermes-2026","cmd":"whoami"}'
```

### Client Implementation (Python)

```python
import socket, json

M2 = "100.105.185.60"
PORT = 17423
TOKEN = "nexus-macmini-hermes-2026"

def send_cmd(cmd):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(10)
    s.connect((M2, PORT))
    s.sendall(json.dumps({"token": TOKEN, "cmd": cmd}).encode() + b"\n")
    data = s.recv(8192)
    s.close()
    return json.loads(data.decode().strip())
```

### Fallback Protocol

When the bridge is unavailable (M2 sleeping, network down), agents use **NATS pub/sub** (:4222) for asynchronous messaging. The bridge is for synchronous commands. NATS is for events and state. Together they cover all communication patterns.

## FAQ

**Q1: Why TCP instead of HTTP?**
TCP is simpler. No headers, no methods, no status codes. Just connect, send JSON, receive JSON, disconnect. The M2→M4 direction happens to use HTTP because curl is convenient, but the protocol is identical. Both sides speak the same JSON.

**Q2: Why a single token instead of per-agent keys?**
Simplicity. We have 4 agents on 3 machines behind Tailscale VPN. The threat model is "someone on the Tailscale network" — if they're on Tailscale, they already have access. Per-agent tokens add complexity without adding security in this setup. For production multi-tenant, add per-agent tokens.

**Q3: What about large command output?**
The current buffer is 8192 bytes. For larger output, the command should write to a file and the response includes the file path. This keeps the protocol simple and avoids streaming complexity.

**Q4: How does this compare to SSH?**
SSH requires key setup per machine, doesn't work with dynamic Tailscale IPs without reconfiguration, and returns unstructured text. The bridge uses Tailscale IPs directly, has a single shared token, and returns structured JSON that agents can parse programmatically.

**Q5: Why port 17423?**
It was available and we already had bridge servers on both machines using this port. The Receiver (HTTP API) is on 17420, Relay (multi-agent chat) is on 17426. Bridge fits in between. The specific number doesn't matter — what matters is that it's DOCUMENTED and agents don't have to guess.

**Q6: How do agents discover the M2 IP?**
The IP is hardcoded in the client script and documented in `~/agentos/constitution/KOMUNIKACJA.md`. Tailscale IPs are stable (don't change between reconnects). If the IP changes, update one file.

**Q7: What happens on timeout?**
Return `{"error": "timeout"}`. The calling agent should retry once after 5 seconds. If still failing, log to HANDOFF and continue with local work. Never block the session waiting for a remote agent.

**Q8: Can multiple commands run simultaneously?**
No. The bridge is synchronous — one command, one response, disconnect. For parallel work, open multiple connections. Each is independent and stateless.

**Q9: How do agents know this is THE protocol?**
HARNESS.md §0 (added 2026-05-21) mandates: before any work, verify bridge connectivity. The communication standard is in `~/agentos/constitution/KOMUNIKACJA.md`. Every agent reads HARNESS first, so every agent knows the protocol.

**Q10: What about bidirectional streaming?**
Not needed for our use case. Agents send commands and get responses. For real-time chat, use the Relay (:17426). For state synchronization, use NATS KV. The bridge is for "do this and tell me what happened."

**Q11: Is this production-ready?**
It's battle-tested across 50+ sessions between M4 and M2. It handles: M2 sleeping, Tailscale reconnects, large outputs, special characters in commands, concurrent connections. It's simple enough to be reliable.

**Q12: How do I add a new machine?**
Install the bridge server script on the new machine. Add its Tailscale IP to the client script. Done. The protocol doesn't change — only the IP list grows.

**Q13: What about security beyond the token?**
For now: Tailscale VPN (encrypted) + token (shared secret). For higher security: add TLS to the TCP connection, use per-agent tokens, add command allowlisting. These are extensions, not protocol changes.

**Q14: Why not just use NATS for everything?**
NATS is great for pub/sub and KV. But synchronous request-response ("run this command and give me output") is not NATS's strength. NATS request-reply exists but adds complexity. The bridge is simpler for this specific pattern.

**Q15: How do I debug connection issues?**
```bash
# Check Tailscale
tailscale status | grep macbook

# Test bridge
python3 ~/agentos/bridge-to-m2.py "echo OK"

# Check HARNESS
cat ~/agentos/constitution/KOMUNIKACJA.md
```

---

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)
GitHub: [github.com/nerudek](https://github.com/nerudek)
