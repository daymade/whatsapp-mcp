# WhatsApp MCP Server

This is a Model Context Protocol (MCP) server for WhatsApp.

With this you can search and read your personal Whatsapp messages (including images, videos, documents, and audio messages), search your contacts and send messages to either individuals or groups. You can also send media files including images, videos, documents, and audio messages.

It connects to your **personal WhatsApp account** directly via the Whatsapp web multidevice API (using the [whatsmeow](https://github.com/tulir/whatsmeow) library). All your messages are stored locally in a SQLite database and only sent to an LLM (such as Claude) when the agent accesses them through tools (which you control).

Here's an example of what you can do when it's connected to Claude.

![WhatsApp MCP](./example-use.png)

> To get updates on this and other projects I work on [enter your email here](https://docs.google.com/forms/d/1rTF9wMBTN0vPfzWuQa2BjfGKdKIpTbyeKxhPMcEzgyI/preview)

> *Caution:* as with many MCP servers, the WhatsApp MCP is subject to [the lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/). This means that project injection could lead to private data exfiltration.

## About this fork

Upstream ([lharries/whatsapp-mcp](https://github.com/lharries/whatsapp-mcp)) could not log in at the time of this fork (September 2026). WhatsApp rejects its pinned whatsmeow build and closes the connection before a QR code is ever shown, so pairing never starts. At the default log level the only symptom is `websocket: close 1006 (abnormal closure)`; the actual reason, visible at `DEBUG`, is `<failure location="cln" reason="405"/>` — "client outdated".

This fork:

- bumps whatsmeow so login works, and passes `context.Context` to the five APIs that grew one in the meantime
- binds the local REST API to `127.0.0.1` instead of every interface — `/api/send` has no authentication, so a wildcard bind lets anyone who can reach the machine on that port send messages as the paired account
- makes the REST API port configurable (`WHATSAPP_API_PORT`), because 8080 was hardcoded in two files that have to agree, and a port collision fails quietly
- makes the log level configurable (`WA_LOG_LEVEL`), because the failure above is invisible without it
- allows 10 minutes to scan the QR code rather than 3

No other behaviour is changed. Outside the Go and Python source, this fork also ignores the local session store and build output, and carries a regenerated `uv.lock` (lock-format only; no dependency versions differ).

**This will break again.** The whatsmeow version is pinned, and WhatsApp rejects clients that fall too far behind. When login stops working, see [Login fails, no QR code](#login-fails-no-qr-code) — the fix is two commands.

## Installation

### Prerequisites

- Go 1.26 or newer. The upgraded whatsmeow requires it; Go's default `GOTOOLCHAIN=auto` will fetch a newer toolchain for you if your installed Go is older.
- Python 3.11 or newer (`whatsapp-mcp-server/pyproject.toml` sets `requires-python = ">=3.11"`)
- Anthropic Claude Desktop app (or Cursor)
- UV (Python package manager), install with `curl -LsSf https://astral.sh/uv/install.sh | sh`
- FFmpeg (_optional_) - Only needed for audio messages. If you want to send audio files as playable WhatsApp voice messages, they must be in `.ogg` Opus format. With FFmpeg installed, the MCP server will automatically convert non-Opus audio files. Without FFmpeg, you can still send raw audio files using the `send_file` tool.

### Steps

1. **Clone this repository**

   ```bash
   git clone https://github.com/daymade/whatsapp-mcp.git
   cd whatsapp-mcp
   ```

2. **Run the WhatsApp bridge**

   Navigate to the whatsapp-bridge directory and run the Go application:

   ```bash
   cd whatsapp-bridge
   go run main.go
   ```

   The first time you run it, you will be prompted to scan a QR code. Scan the QR code with your WhatsApp mobile app to authenticate.

   After approximately 20 days, you will might need to re-authenticate.

   Two environment variables are available:

   | Variable | Default | Purpose |
   | --- | --- | --- |
   | `WHATSAPP_API_PORT` | `8080` | Port for the bridge's local REST API. If 8080 is taken, set this **and** point the Python server at the same port with `WHATSAPP_API_BASE_URL=http://127.0.0.1:<port>/api` — the two must agree. |
   | `WA_LOG_LEVEL` | `INFO` | Set to `DEBUG` to see the WhatsApp handshake and the XML stanzas. Connection failures are not diagnosable at `INFO`. |

3. **Connect it to your agent**

   The Python MCP server is launched by your agent, not by you — you only
   need to tell the agent how to start it. Every client below runs the same
   command; only the config format differs.

   Substitute two values:

   - `<UV>` — the output of `which uv`
   - `<REPO>` — the output of `pwd` at the root of this repository

   **Claude Code**

   ```bash
   claude mcp add whatsapp -s user -- <UV> --directory <REPO>/whatsapp-mcp-server run main.py
   ```

   Verify with `claude mcp list`; it should report `whatsapp ... ✔ Connected`.

   **Claude Desktop** — merge this into the `mcpServers` object in
   `~/Library/Application Support/Claude/claude_desktop_config.json`.
   Do not replace the file: it holds your other servers and your settings.
   If the file does not exist, create it with just this content.

   ```json
   {
     "mcpServers": {
       "whatsapp": {
         "command": "<UV>",
         "args": ["--directory", "<REPO>/whatsapp-mcp-server", "run", "main.py"]
       }
     }
   }
   ```

   **Cursor** — same JSON, in `~/.cursor/mcp.json`.

   **Codex** — append to `~/.codex/config.toml`.

   ```toml
   [mcp_servers.whatsapp]
   type = "stdio"
   command = "<UV>"
   args = ["--directory", "<REPO>/whatsapp-mcp-server", "run", "main.py"]
   ```

   If you moved the bridge off port 8080, every client above also needs
   `WHATSAPP_API_BASE_URL=http://127.0.0.1:<port>/api` in its environment
   (`claude mcp add -e ...`, an `"env"` object, or `[mcp_servers.whatsapp.env]`).

4. **Restart the agent**

   Claude Desktop and Cursor read their config at startup. Claude Code picks
   up a new server on the next session, not the current one.

5. **Keep the bridge running**

   The Go bridge is what actually holds the WhatsApp connection. The MCP
   server reads the database it writes and calls its local API — so when the
   bridge is not running, your agent sees stale messages and cannot send.

   `go run main.go` ties it to a terminal window: close the window and the
   connection is gone, silently. For anything beyond trying it out, build a
   binary and run it under a supervisor.

   ```bash
   cd whatsapp-bridge
   go build -o bin/whatsapp-bridge .
   ```

   On macOS, a LaunchAgent at
   `~/Library/LaunchAgents/whatsapp-bridge.plist`:

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0">
   <dict>
     <key>Label</key>
     <string>whatsapp-bridge</string>
     <key>ProgramArguments</key>
     <array>
       <string><REPO>/whatsapp-bridge/bin/whatsapp-bridge</string>
     </array>
     <key>WorkingDirectory</key>
     <string><REPO>/whatsapp-bridge</string>
     <key>RunAtLoad</key>
     <true/>
     <key>KeepAlive</key>
     <true/>
     <key>StandardOutPath</key>
     <string>/tmp/whatsapp-bridge.out.log</string>
     <key>StandardErrorPath</key>
     <string>/tmp/whatsapp-bridge.err.log</string>
   </dict>
   </plist>
   ```

   ```bash
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/whatsapp-bridge.plist
   ```

   `WorkingDirectory` is required, not cosmetic: the bridge resolves `store/`
   relative to the current directory, so without it you get a second, empty
   session store in whatever directory the supervisor started from.

   Pair first, in a terminal, before putting it under a supervisor — the QR
   code has to be scanned by a human.

   **Do not treat "the process is alive" as healthy.** On logout the bridge
   logs the event and keeps running, so the process, the port and your
   database all look normal while sending fails. Check for it with:

   ```bash
   grep -i "logged out" /tmp/whatsapp-bridge.out.log
   ```

### Windows Compatibility

If you're running this project on Windows, be aware that `go-sqlite3` requires **CGO to be enabled** in order to compile and work properly. By default, **CGO is disabled on Windows**, so you need to explicitly enable it and have a C compiler installed.

#### Steps to get it working:

1. **Install a C compiler**  
   We recommend using [MSYS2](https://www.msys2.org/) to install a C compiler for Windows. After installing MSYS2, make sure to add the `ucrt64\bin` folder to your `PATH`.  
   → A step-by-step guide is available [here](https://code.visualstudio.com/docs/cpp/config-mingw).

2. **Enable CGO and run the app**

   ```bash
   cd whatsapp-bridge
   go env -w CGO_ENABLED=1
   go run main.go
   ```

Without this setup, you'll likely run into errors like:

> `Binary was compiled with 'CGO_ENABLED=0', go-sqlite3 requires cgo to work.`

## Architecture Overview

This application consists of two main components:

1. **Go WhatsApp Bridge** (`whatsapp-bridge/`): A Go application that connects to WhatsApp's web API, handles authentication via QR code, and stores message history in SQLite. It serves as the bridge between WhatsApp and the MCP server.

2. **Python MCP Server** (`whatsapp-mcp-server/`): A Python server implementing the Model Context Protocol (MCP), which provides standardized tools for Claude to interact with WhatsApp data and send/receive messages.

### Data Storage

- All message history is stored in a SQLite database within the `whatsapp-bridge/store/` directory
- The database maintains tables for chats and messages
- Messages are indexed for efficient searching and retrieval

## Usage

Once connected, you can interact with your WhatsApp contacts through Claude, leveraging Claude's AI capabilities in your WhatsApp conversations.

### MCP Tools

Claude can access the following tools to interact with WhatsApp:

- **search_contacts**: Search for contacts by name or phone number
- **list_messages**: Retrieve messages with optional filters and context
- **list_chats**: List available chats with metadata
- **get_chat**: Get information about a specific chat
- **get_direct_chat_by_contact**: Find a direct chat with a specific contact
- **get_contact_chats**: List all chats involving a specific contact
- **get_last_interaction**: Get the most recent message with a contact
- **get_message_context**: Retrieve context around a specific message
- **send_message**: Send a WhatsApp message to a specified phone number or group JID
- **send_file**: Send a file (image, video, raw audio, document) to a specified recipient
- **send_audio_message**: Send an audio file as a WhatsApp voice message (requires the file to be an .ogg opus file or ffmpeg must be installed)
- **download_media**: Download media from a WhatsApp message and get the local file path

### Media Handling Features

The MCP server supports both sending and receiving various media types:

#### Media Sending

You can send various media types to your WhatsApp contacts:

- **Images, Videos, Documents**: Use the `send_file` tool to share any supported media type.
- **Voice Messages**: Use the `send_audio_message` tool to send audio files as playable WhatsApp voice messages.
  - For optimal compatibility, audio files should be in `.ogg` Opus format.
  - With FFmpeg installed, the system will automatically convert other audio formats (MP3, WAV, etc.) to the required format.
  - Without FFmpeg, you can still send raw audio files using the `send_file` tool, but they won't appear as playable voice messages.

#### Media Downloading

By default, just the metadata of the media is stored in the local database. The message will indicate that media was sent. To access this media you need to use the download_media tool which takes the `message_id` and `chat_jid` (which are shown when printing messages containing the meda), this downloads the media and then returns the file path which can be then opened or passed to another tool.

## Technical Details

1. Claude sends requests to the Python MCP server
2. The MCP server queries the Go bridge for WhatsApp data or directly to the SQLite database
3. The Go accesses the WhatsApp API and keeps the SQLite database up to date
4. Data flows back through the chain to Claude
5. When sending messages, the request flows from Claude through the MCP server to the Go bridge and to WhatsApp

## Troubleshooting

- If you encounter permission issues when running uv, you may need to add it to your PATH or use the full path to the executable.
- Make sure both the Go application and the Python server are running for the integration to work properly.

### Login fails, no QR code

If the bridge prints `websocket: close 1006 (abnormal closure): unexpected EOF` and no QR code ever appears, restarting will not help — WhatsApp is rejecting the client. Confirm it:

```bash
WA_LOG_LEVEL=DEBUG go run main.go
```

`<failure location="cln" reason="405"/>` in the output means the pinned whatsmeow build is too old for WhatsApp's current server. Fix it in place:

```bash
cd whatsapp-bridge
go get go.mau.fi/whatsmeow@latest && go mod tidy
go build .
```

If the build then fails, it is because whatsmeow changed an API signature; the errors name each call site and are usually a matter of passing `context.Background()` as a new first argument.

### Messages you send do not appear in the database

The bridge records incoming messages, but `/api/send` does not write to the
message store, and WhatsApp does not echo a message back to the device that
sent it. So a message sent through `send_message` reaches its recipient and
is visible in WhatsApp on your phone, while `list_messages` and
`get_last_interaction` still show only what came in.

This matters when an agent sends something and then checks whether it did:
it will conclude that it did not. Treat the tool's own success response as
the record of a send.

This is upstream behaviour, unchanged here.

### Authentication Issues

- **QR Code Not Displaying (terminal)**: If the log shows a QR code was emitted but you see nothing legible, your terminal may not render half-block characters. Try a different terminal or enlarge the window.
- **WhatsApp Already Logged In**: If your session is already active, the Go bridge will automatically reconnect without showing a QR code.
- **Device Limit Reached**: WhatsApp limits the number of linked devices. If you reach this limit, you'll need to remove an existing device from WhatsApp on your phone (Settings > Linked Devices).
- **No Messages Loading**: After initial authentication, it can take several minutes for your message history to load, especially if you have many chats.
- **WhatsApp Out of Sync**: If your WhatsApp messages get out of sync with the bridge, delete both database files (`whatsapp-bridge/store/messages.db` and `whatsapp-bridge/store/whatsapp.db`) and restart the bridge to re-authenticate.

For additional Claude Desktop integration troubleshooting, see the [MCP documentation](https://modelcontextprotocol.io/quickstart/server#claude-for-desktop-integration-issues). The documentation includes helpful tips for checking logs and resolving common issues.
