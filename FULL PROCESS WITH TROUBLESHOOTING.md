# UniFi-MCP-Setup-
Getting a UniFi MCP server working in Claude Desktop (Windows, MSIX/Cowork build)

A record of everything that went wrong and how it was actually fixed, so the next setup (or the next person hitting the same wall) doesn't have to rediscover it.

TL;DR

On current Claude Desktop builds — especially the Microsoft Store / MSIX build on Windows — you cannot add a local MCP server by hand-editing claude_desktop_config.json anymore. That file still exists but no longer holds an mcpServers list; it's just the app's internal UI state. Local MCP servers must be installed as Desktop Extensions (.mcpb files) via Settings → Extensions → Advanced settings → Extension Developer.

If you're wrapping a Python package that's normally launched with uvx <package>, don't point mcp_config.command at uvx directly — bundle a tiny launcher script and use subprocess.run, not os.execv (details below).

Background

Goal: give Claude Desktop tool access to a home UniFi Network console (a UDM-class gateway) — device/client listing, VLANs, firewall, etc. — using the community enuno/unifi-mcp-server PyPI package (run via uv's uvx), talking to the console's local API with an API key generated from Settings → Control Plane → Integrations on the console itself.

Attempt 1: manual claude_desktop_config.json — doesn't do anything

The classic, widely-documented approach:

json
{
  "mcpServers": {
    "unifi": {
      "command": "uvx",
      "args": ["unifi-mcp-server"],
      "env": {
        "UNIFI_API_KEY": "...",
        "UNIFI_API_TYPE": "local",
        "UNIFI_LOCAL_HOST": "192.168.1.1",
        "UNIFI_LOCAL_VERIFY_SSL": "false"
      }
    }
  }
}

This is still what most MCP server READMEs tell you to do. It no longer works on current Cowork-enabled / MSIX Claude Desktop builds. Two gotchas stacked on top of each other:

The path itself is virtualized. On an MSIX-packaged install, the app's "Roaming AppData" is redirected to C:\Users\<you>\AppData\Local\Packages\<PackageFamilyName>\LocalCache\Roaming\Claude\, not the classic %APPDATA%\Claude\. Editing the classic path edits a file nobody reads.
Even the real file isn't the right one anymore. Opening the actual claude_desktop_config file at that virtualized path shows it's just the app's internal UI/session state (window layout, feature flags, etc.) — there's no mcpServers key for it to read in the first place. This isn't a bug so much as the app having moved on to a different mechanism.

Lesson: if a fresh mcpServers edit doesn't show up in Settings → Developer / doesn't appear as a running server after a full restart, stop debugging the JSON and go check whether your build even supports this path anymore.

Attempt 2: Desktop Extensions (.mcpb) — the actual current mechanism

Local MCP servers are packaged as .mcpb files ("MCP Bundle" / formerly "DXT", Desktop Extensions) — a zip archive containing at minimum a manifest.json. Install via:

Settings → Extensions → Advanced settings → Extension Developer → Install Extension… (packaged .mcpb) or Install Unpacked Extension… (a folder — gives much clearer error messages, so prefer this while iterating).

Useful references:

anthropics/mcpb — spec + MANIFEST.md
support.claude.com — Getting Started with Local MCP Servers
Manifest iteration log

v1 — tried to shortcut it: server.type: "binary", no entry_point, mcp_config.command pointed straight at uvx:

json
"server": {
  "type": "binary",
  "mcp_config": {
    "command": "uvx",
    "args": ["unifi-mcp-server"],
    "env": { "...": "..." }
  }
}

Result: installer said "node.js not found" (this build's extension installer checks for a system Node.js regardless of the server's actual runtime — install Node, e.g. winget install OpenJS.NodeJS.LTS, if you hit this). After installing Node: "couldn't preview extension" — an opaque failure that turned out to just mean "invalid manifest," not a deeper platform bug (see below).

v2 — added entry_point, kept the direct uvx command, switched type to "uv":

json
"server": {
  "type": "uv",
  "entry_point": "unifi-mcp-server",
  "mcp_config": {
    "command": "uvx",
    "args": ["unifi-mcp-server"],
    "env": { "...": "..." }
  }
}

Using "Install Unpacked Extension" instead of the packaged flow surfaced a real, specific validation error this time: Invalid manifest: server: Required. Packaged installs swallow this into the generic "couldn't preview extension" message — unpacked installs give you the real error, so prefer it for iterating.

v3 — the actual fix for the manifest error: stop trying to point mcp_config.command straight at an external uvx binary. Bundle a real entry_point file and match Anthropic's own documented working example shape exactly:

extension/
├── manifest.json
└── server/
    └── main.py
json
"server": {
  "type": "uv",
  "entry_point": "server/main.py",
  "mcp_config": {
    "command": "uv",
    "args": ["run", "--directory", "${__dirname}", "server/main.py"],
    "env": { "...": "..." }
  }
}

server/main.py just hands off to uvx:

python
import os, shutil, sys
uvx = shutil.which("uvx")
if uvx is None:
    sys.stderr.write("uvx not found on PATH. Install from https://astral.sh/uv\n")
    sys.exit(1)
os.execv(uvx, [uvx, "unifi-mcp-server"])

This installed successfully (manifest valid) and get_device_info showed the extension as "state": "starting" — real progress. But it got permanently stuck on "starting" for 5+ minutes, never erroring, never connecting.

Ruled out at this point:

uvx missing — confirmed installed and working (uvx --version → fine).
The wrapped package itself failing — running uvx unifi-mcp-server manually in PowerShell with the real env vars set started clean and sat waiting on stdin, which is the correct behavior for an MCP stdio server.
First-run download timeout — package was already cached locally from the manual test; retried after caching, same stuck-starting result.

v4 — the actual fix: os.execv is only emulated by Python on Windows — it spawns a child process and waits, rather than truly replacing the current process image the way POSIX execve does. That can mishandle the stdio pipe handles Claude Desktop set up for the MCP JSON-RPC transport, which matches "starts, but the handshake never completes" exactly. Swap it for subprocess.run with default (inherited) stdio:

python
import shutil, subprocess, sys
uvx = shutil.which("uvx")
if uvx is None:
    sys.stderr.write("uvx not found on PATH. Install from https://astral.sh/uv\n")
    sys.exit(1)
result = subprocess.run([uvx, "unifi-mcp-server"])
sys.exit(result.returncode)

This worked. get_device_info moved from "starting" to "announced" (connected), and real tool calls (health_check, list_sites, list_devices_by_type) returned live data from the console.

Final working manifest.json
json
{
  "manifest_version": "0.4",
  "name": "unifi-network",
  "display_name": "UniFi Network",
  "version": "0.2.5",
  "description": "Manage and monitor a UniFi Network controller (devices, clients, VLANs, firewall, QoS, backups, topology) via the official local API.",
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
      "description": "Local API key from your UniFi console: Settings → Control Plane → Integrations → Create API Key.",
      "sensitive": true,
      "required": true
    },
    "console_host": {
      "type": "string",
      "title": "Console IP address",
      "description": "Local IP of your UniFi OS console (gateway/UDM).",
      "default": "192.168.1.1",
      "required": true
    }
  },
  "keywords": ["unifi", "network", "ubiquiti", "firewall", "vlan"],
  "license": "MIT",
  "compatibility": { "platforms": ["win32", "darwin", "linux"] }
}

server/main.py:

python
#!/usr/bin/env python3
"""Thin launcher: hands off to `uvx unifi-mcp-server`, inheriting stdio and env.

Uses subprocess (not os.execv) because os.execv is only emulated on Windows
(Python spawns a new process and waits rather than truly replacing the
current one), which can mishandle the stdio pipes Claude Desktop set up for
the MCP JSON-RPC transport. subprocess.run() with default (inherited)
stdin/stdout/stderr is the reliable cross-platform way to hand off stdio.
"""
import shutil
import subprocess
import sys

uvx = shutil.which("uvx")
if uvx is None:
    sys.stderr.write(
        "uvx was not found on PATH. Install uv from https://astral.sh/uv "
        "and restart Claude Desktop.\n"
    )
    sys.exit(1)

result = subprocess.run([uvx, "unifi-mcp-server"])
sys.exit(result.returncode)

Package it: zip manifest.json and server/ together, name it whatever you like with a .mcpb extension (it's just a zip). For iterating, skip the zip step and use "Install Unpacked Extension" pointed at the folder directly — the error messages are much more useful.

Prerequisites checklist
uv installed and on PATH (irm https://astral.sh/uv/install.ps1 | iex on Windows) — needed for uvx to exist at all.
Node.js installed (winget install OpenJS.NodeJS.LTS on Windows) — this build's extension installer checks for it regardless of the wrapped server's actual runtime.
A UniFi local API key from the console: Settings → Control Plane → Integrations → Create API Key (shown once — copy it immediately).
Your console's local IP address (the gateway/UDM you log into).
Verifying it's actually connected

From a session with device-bridge access (or just by testing in a normal chat): ask Claude something that requires a UniFi tool call. A generic "no connector found" response, or get_device_info/extension status stuck on anything other than "announced"/"connected", means it's not live yet — don't trust "it installed without an error" as proof it's working.

Server & security notes

enuno/unifi-mcp-server exposes on the order of 200 tools, spanning read-only queries (devices, clients, topology, stats) and mutations (VLANs, firewall rules, port forwarding, device restarts, backups). Every mutating tool takes a confirm flag the caller must set to true — there is no hard, config-level read-only mode. If that's more exposure than you want, ryanbehan/unifi-network-mcp is a drop-in alternative with no write tools at all.

References
anthropics/mcpb — the .mcpb/MCPB spec and manifest reference
support.claude.com: Getting Started with Local MCP Servers on Claude Desktop
enuno/unifi-mcp-server
ryanbehan/unifi-network-mcp (read-only alternative)
Related open issues describing similar Windows MSIX-build extension pain: modelcontextprotocol/mcpb#281, anthropics/claude-code#82469
