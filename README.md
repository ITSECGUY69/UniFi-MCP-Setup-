UniFi MCP Setup for Claude Desktop

Getting enuno/unifi-mcp-server (a UniFi Network API MCP server) working as a Claude Desktop extension on Windows. Working end-to-end as of tonight — this is what it actually took.

TL;DR

Claude Desktop no longer reads claude_desktop_config.json for MCP servers (that file is just internal app state now). Local MCP servers have to be installed as Desktop Extensions (.mcpb) via Settings → Extensions → Advanced settings → Extension Developer. If you're wrapping a uvx-launched Python package, don't point the manifest straight at uvx — bundle a tiny launcher script that uses subprocess.run, not os.execv (breaks on Windows).

What it took, in order
Manual JSON config doesn't work anymore. On this build (Microsoft Store/MSIX), %APPDATA%\Claude\claude_desktop_config.json isn't even the real file — MSIX redirects app data to AppData\Local\Packages\<PackageFamilyName>\LocalCache\Roaming\Claude\. Even the real file there holds Cowork's internal UI state, not a server list.
The real mechanism is .mcpb Desktop Extensions. Package a manifest.json (+ any bundled code) into a zip with a .mcpb extension, install via Settings → Extensions → Advanced settings → Extension Developer.
First manifest attempt pointed straight at uvx with no bundled entry point → installer error node.js not found (checks for Node regardless of the server's real runtime) → then couldn't install extension → then, once switched to "Install Unpacked Extension" for clearer errors: Invalid manifest: server: Required.
Fix: bundle a real entry_point file matching Anthropic's documented working example shape (server.type: "uv", mcp_config.command: "uv", args: ["run","--directory","${__dirname}","server/main.py"]) instead of trying to shortcut with an external command.
Extension installed and started, but hung on "starting" forever. Root cause: the bundled server/main.py used os.execv() to hand off to uvx unifi-mcp-server — but os.execv is only emulated on Windows (Python spawns a child and waits, it doesn't truly replace the process), which mangled the stdio pipes Claude Desktop needs for the MCP handshake.
Real fix: swap os.execv for subprocess.run([uvx, "unifi-mcp-server"]) (inherits stdio properly). Extension went from "starting" → "announced" (connected), and real tool calls started returning live data from the UniFi console.
Prerequisites
uv installed (irm https://astral.sh/uv/install.ps1 | iex on Windows)
Node.js installed (winget install OpenJS.NodeJS.LTS) — required by the extension installer itself, not by the server
A local API key from your UniFi console: Settings → Control Plane → Integrations → Create API Key
Your console's local IP address
Final working manifest.json
json
{
  "manifest_version": "0.4",
  "name": "unifi-network",
  "display_name": "UniFi Network",
  "version": "0.2.5",
  "description": "Manage and monitor a UniFi Network controller via the official local API.",
  "author": { "name": "enuno", "url": "https://github.com/enuno/unifi-mcp-server" },
  "server": {
    "type": "uv",
    "entry_point": "server/main.py",
    "mcp_config": {
      "command": "uv",
      "args": ["run", "--directory", "${__dirname}", "server/main.py"],
      "env": {
        "UNIFI_API_KEY": "${user_config.api_key}",
        "UNIFI_API_TYPE": "local",
        "UNIFI_LOCAL_HOST": "${user_config.console_host}",
        "UNIFI_LOCAL_VERIFY_SSL": "false"
      }
    }
  },
  "user_config": {
    "api_key": {
      "type": "string",
      "title": "UniFi API Key",
      "sensitive": true,
      "required": true
    },
    "console_host": {
      "type": "string",
      "title": "Console IP address",
      "default": "192.168.1.1",
      "required": true
    }
  },
  "compatibility": { "platforms": ["win32", "darwin", "linux"] }
}

server/main.py:

python
import shutil, subprocess, sys

uvx = shutil.which("uvx")
if uvx is None:
    sys.stderr.write("uvx not found on PATH. Install from https://astral.sh/uv\n")
    sys.exit(1)

result = subprocess.run([uvx, "unifi-mcp-server"])
sys.exit(result.returncode)

Zip manifest.json + server/ into a .mcpb (or install the unpacked folder directly — better error messages while iterating).

Security note

This server exposes ~200 tools including writes (VLANs, firewall rules, device restarts, backups), gated only by a confirm flag the model must pass — no hard read-only mode. ryanbehan/unifi-network-mcp is a read-only alternative if that's too much exposure.

References
anthropics/mcpb — the .mcpb spec
support.claude.com — Local MCP Servers on Claude Desktop
enuno/unifi-mcp-server
