# MCP Server Research — Remote Machine Access Setup

Goal: Give an OpenHuman agent SSH/terminal access to every machine on a tailnet via MCP servers.

## Architecture

```
[OpenHuman] → config.toml → [[mcp_client.servers]]
       │
       ├── "kali"     → ssh-mcp → Kali Linux (tailscale IP)
       ├── "windows"  → ssh-mcp → Windows box (tailscale IP)
       └── ...any more machines
```

## Step 1 — Install Tailscale on every machine

```bash
# Linux (Kali, etc.):
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Windows: download from https://tailscale.com/download
```

Verify: `tailscale status` shows all machines with `100.x.y.z` IPs.

## Step 2 — Set up SSH key auth from the OpenHuman host to each target

On the machine running OpenHuman:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/mcp_key -N ""
```

Copy the public key to each target:
```bash
ssh-copy-id -i ~/.ssh/mcp_key.pub user@kali-tailscale-ip
```

Verify passwordless login works:
```bash
ssh -i ~/.ssh/mcp_key user@100.x.y.z "hostname"
```

## Step 3 — Install ssh-mcp on the OpenHuman host

The MCP server runs on the OpenHuman host, not on the targets. It uses SSH to reach each target.

```bash
npm install -g ssh-mcp
# or just use npx (no install needed)
```

## Step 4 — Configure OpenHuman's config.toml

Edit `~/.openhuman/config.toml` and add:

```toml
[mcp_client]
enabled = true

# --- Kali Linux ---
[[mcp_client.servers]]
name = "kali"
description = "Kali Linux pentesting machine"
command = "npx"
args = ["-y", "ssh-mcp", "--", "--host=100.x.y.z", "--user=youruser", "--key=/home/you/.ssh/mcp_key"]
enabled = true
timeout_secs = 120

# --- Windows dev box ---
[[mcp_client.servers]]
name = "windows"
description = "Windows development machine"
command = "npx"
args = ["-y", "ssh-mcp", "--", "--host=100.x.y.z", "--user=youruser", "--key=/home/you/.ssh/mcp_key"]
enabled = true
timeout_secs = 120
```

Replace `100.x.y.z` with each machine's actual Tailscale IP. Replace `youruser` with the SSH username.

For Windows targets, you may need to install OpenSSH Server on Windows and enable it.

## Step 5 — Restart OpenHuman

Once the config is saved, restart OpenHuman.

## Verification

After restart:
- `mcp_list_servers` → should show `["kali", "windows"]`
- `mcp_list_tools(server="kali")` → should show `exec` and `sudo-exec`
- `mcp_call_tool(server="kali", tool="exec", args={"command": "whoami"})` → should return the remote username

## Optional: Tailscale Aperture (centralized gateway)

If you want a single endpoint instead of per-machine config, set up Aperture on one machine:

```json
{
  "mcp": {
    "servers": {
      "kali": { "url": "http://kali-machine.tail-scale.ts.net:8080/v1/mcp" },
      "windows": { "url": "http://win-dev.tail-scale.ts.net:8080/v1/mcp" }
    }
  }
}
```

Then in config.toml, just one server pointing at the Aperture host.

## ssh-mcp CLI Reference

| Flag | Required | Description |
|------|----------|-------------|
| `--host` | Yes | Target hostname/IP |
| `--user` | Yes | SSH username |
| `--port` | No | SSH port (default 22) |
| `--password` | No | SSH password (use key instead if possible) |
| `--key` | No | Path to SSH private key |
| `--sudoPassword` | No | Password for sudo |
| `--timeout` | No | Command timeout in ms (default 60000) |

## OpenHuman MCP Config Schema Reference

```toml
[mcp_client]
enabled = true

[[mcp_client.servers]]
name         = "server-slug"
endpoint     = "https://example.com/v1/mcp"   # HTTP transport
command      = "npx"                           # stdio transport (used if non-empty)
args         = ["-y", "ssh-mcp", "--", "--host=..."]
env          = { KEY = "val" }
cwd          = "/path/to/dir"
description  = "Optional description"
enabled      = true
allowed_tools    = []
disallowed_tools = []
timeout_secs     = 30

# Auth options:
auth = { kind = "bearer_token", token = "sk-..." }
auth = { kind = "basic", username = "u", password = "p" }
auth = { kind = "header", name = "X-Key", value = "v" }
auth = { kind = "query_param", name = "api_key", value = "v" }
```

## Sources

- [tufantunc/ssh-mcp](https://github.com/tufantunc/ssh-mcp)
- [OpenHuman MCP Registry docs](https://tinyhumans.gitbook.io/openhuman/developing/architecture/mcp-registry)
- [Tailscale Aperture docs](https://tailscale.com/docs/aperture/mcp-server)