---
date: '2025-12-27T22:36:29+08:00'
draft: false
showtoc: true
tags: ["proxy", "network", "ai", "tutorial"]
title: 'Proxy AI Agents'
---
# Intercepting AI Agent Traffic: A Security Researcher's Guide (Ubuntu & macOS Edition)

*You've deployed an AI agent, in your browser, as a desktop app, or as a CLI tool. But do you actually know what it's sending out? This guide walks through how to intercept, inspect, and understand the network traffic of AI agents across every context on both Ubuntu and macOS, and what to do when the interception fails.*

---

## Why This Matters

AI agents make outbound API calls to large language model providers. Every call carries a payload: your system prompts, your conversation history, tool definitions, and potentially sensitive context from your local environment. As a security-conscious practitioner, you shouldn't have to take that on faith. You should be able to see it.

The challenge is that the right interception method depends entirely on *how* the agent is running. A browser extension, a native desktop app, and a Python CLI script each use different networking stacks, and each requires a different approach. The platform matters too: macOS routes most app traffic through its system `NSURLSession` stack and Keychain, while Ubuntu exposes the network more directly through environment variables and `iptables`.

---

## Setting Up Your Interception Toolkit

### mitmproxy

The workhorse for all three agent types. Install it once and use it everywhere.

**Ubuntu:**
```bash
sudo apt update
sudo apt install python3-pip pipx -y
pipx install mitmproxy

# Verify
mitmproxy --version
```

**macOS:**
```bash
# Via Homebrew (recommended)
brew install mitmproxy

# Or via pipx
brew install pipx
pipx install mitmproxy

# Verify
mitmproxy --version
```

---

### Proxyman (macOS only, highly recommended)

Proxyman is purpose-built for macOS traffic inspection. It auto-installs its CA cert into the system Keychain, handles SSL proxying for specific domains, and presents a native macOS UI. For macOS users it is often faster to set up than mitmproxy.

```bash
brew install --cask proxyman
# Launch Proxyman, then: Certificate > Install Certificate & Trust
```

---

### Burp Suite Community Edition

**Ubuntu:**
```bash
wget "https://portswigger.net/burp/releases/download?product=community&type=Linux" \
  -O burp-installer.sh
chmod +x burp-installer.sh
./burp-installer.sh
```

**macOS:**
```bash
# Download the macOS installer from https://portswigger.net/burp/communitydownload
# Or via Homebrew
brew install --cask burp-suite
```

---

### Wireshark

**Ubuntu:**
```bash
sudo apt install wireshark -y

# Allow capture without sudo
sudo usermod -aG wireshark $USER
newgrp wireshark
```

**macOS:**
```bash
brew install --cask wireshark

# Grant capture permissions when prompted, or manually:
sudo chmod +x /Applications/Wireshark.app/Contents/MacOS/Wireshark
sudo chown root:admin /usr/local/bin/dumpcap
sudo chmod 4750 /usr/local/bin/dumpcap
```

---

### Frida (Certificate Pinning Bypass)

**Ubuntu & macOS (same command):**
```bash
pip3 install frida-tools

# Verify
frida --version
```

---

### proxychains (Force Proxy at Syscall Level)

**Ubuntu:**
```bash
sudo apt install proxychains4 -y
```

**macOS:**
```bash
brew install proxychains-ng
```

---

## The Interception Toolkit at a Glance

| Tool | Best For | Ubuntu | macOS | Cost |
|---|---|---|---|---|
| **Browser DevTools** | Browser-based agents | ✓ | ✓ | Free |
| **Burp Suite** | Intercept + replay | ✓ | ✓ | Free (Community) |
| **mitmproxy / mitmweb** | CLI agents, scripting | ✓ | ✓ | Free |
| **Proxyman** | Native macOS apps |, | ✓ | Free tier |
| **Charles Proxy** | GUI-heavy workflows |, | ✓ | Paid |
| **Wireshark** | Raw packet inspection | ✓ | ✓ | Free |
| **Frida** | Cert pinning bypass | ✓ | ✓ | Free |
| **proxychains** | Force proxy (syscall) | ✓ | ✓ | Free |
| **tcpdump** | Low-level capture | ✓ | ✓ | Free (built-in) |
| **pf / pfctl** | Firewall / QUIC block |, | ✓ | Free (built-in) |
| **iptables** | Firewall / QUIC block | ✓ |, | Free (built-in) |

---

## 1. Browser-Based AI Agents

### The Easy Path: DevTools Network Tab

No proxy needed. Open Chrome or Firefox, press `F12`, go to the **Network** tab, and filter by `Fetch/XHR`. Every outbound API call appears here with full headers, request body, and response.

**Ubuntu, launch Chrome with DevTools open:**
```bash
google-chrome --auto-open-devtools-for-tabs
```

**macOS, launch Chrome with DevTools open:**
```bash
open -a "Google Chrome" --args --auto-open-devtools-for-tabs
```

You'll immediately see calls to endpoints like:
```
https://api.anthropic.com/v1/messages
https://api.openai.com/v1/chat/completions
```

---

### Going Deeper: mitmproxy + Browser Proxy

**Step 1, Generate the mitmproxy CA (same on both platforms):**
```bash
mitmproxy --listen-port 8080 &
sleep 2 && kill %1
# CA cert is now at ~/.mitmproxy/mitmproxy-ca-cert.pem
```

**Step 2, Trust the CA:**

*Ubuntu:*
```bash
sudo cp ~/.mitmproxy/mitmproxy-ca-cert.pem /usr/local/share/ca-certificates/mitmproxy.crt
sudo update-ca-certificates
```

*macOS:*
```bash
sudo security add-trusted-cert \
  -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  ~/.mitmproxy/mitmproxy-ca-cert.pem
```

**Step 3, Launch Chrome through the proxy:**

*Ubuntu:*
```bash
google-chrome \
  --proxy-server="http://127.0.0.1:8080" \
  --ignore-certificate-errors-spki-list=$(openssl x509 \
    -in ~/.mitmproxy/mitmproxy-ca-cert.pem -pubkey -noout \
    | openssl pkey -pubin -outform der \
    | openssl dgst -sha256 -binary | base64)
```

*macOS:*
```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --proxy-server="http://127.0.0.1:8080" \
  --ignore-certificate-errors-spki-list=$(openssl x509 \
    -in ~/.mitmproxy/mitmproxy-ca-cert.pem -pubkey -noout \
    | openssl pkey -pubin -outform der \
    | openssl dgst -sha256 -binary | base64)
```

**Start mitmweb for a browser-based inspection UI (both platforms):**
```bash
mitmweb --listen-port 8080
# Opens at http://127.0.0.1:8081
```

---

### Burp Suite for Replay and Modification

**Trust Burp's CA, Ubuntu:**
```bash
# Export from Burp: Proxy > Options > Export CA Certificate (DER format)
sudo cp burp-ca.der /usr/local/share/ca-certificates/burp-ca.crt
sudo update-ca-certificates
```

**Trust Burp's CA, macOS:**
```bash
# Export from Burp: Proxy > Options > Export CA Certificate (DER format)
sudo security add-trusted-cert \
  -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  burp-ca.der
```

In Firefox (both platforms): Preferences → Privacy & Security → View Certificates → Import → check "Trust this CA to identify websites". Set manual proxy to `127.0.0.1:8080`.

---

## 2. Desktop AI Apps

### Ubuntu, Electron / Snap / Deb Apps

Most AI desktop clients on Linux are Electron-based and accept proxy flags directly:

```bash
# Find the app binary
which claude
ls /snap/bin/   # For Snap-installed apps

# Launch with proxy flags
/path/to/app \
  --proxy-server="http://127.0.0.1:8080" \
  --ignore-certificate-errors

# Or via environment variable
export ELECTRON_EXTRA_LAUNCH_ARGS="--proxy-server=http://127.0.0.1:8080"
/path/to/app
```

**System-wide proxy via GNOME settings:**
```bash
gsettings set org.gnome.system.proxy mode 'manual'
gsettings set org.gnome.system.proxy.http host '127.0.0.1'
gsettings set org.gnome.system.proxy.http port 8080
gsettings set org.gnome.system.proxy.https host '127.0.0.1'
gsettings set org.gnome.system.proxy.https port 8080

# Reset when done
gsettings set org.gnome.system.proxy mode 'none'
```

---

### macOS, Native Apps (.app bundles)

macOS native apps use `NSURLSession` and respect the system network proxy settings. This makes interception straightforward, you set the proxy once and all well-behaved apps route through it automatically.

**Option A: Proxyman (easiest)**

Proxyman auto-configures the system proxy and Keychain trust on install. Just enable SSL Proxying for your target domain:

```
Proxyman → SSL Proxying → Enable SSL Proxying for: *.anthropic.com
```

Then launch your target app. All traffic appears decoded in Proxyman's UI with zero additional setup.

**Option B: mitmproxy + System Proxy via Network Settings**

```bash
# Start mitmproxy
mitmproxy --listen-port 8080

# Set system proxy via networksetup (macOS CLI)
# Replace "Wi-Fi" with your active interface, check with: networksetup -listallnetworkservices
networksetup -setwebproxy "Wi-Fi" 127.0.0.1 8080
networksetup -setsecurewebproxy "Wi-Fi" 127.0.0.1 8080
networksetup -setwebproxystate "Wi-Fi" on
networksetup -setsecurewebproxystate "Wi-Fi" on

# Trust the mitmproxy CA in macOS Keychain
sudo security add-trusted-cert \
  -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  ~/.mitmproxy/mitmproxy-ca-cert.pem

# When done, restore defaults
networksetup -setwebproxystate "Wi-Fi" off
networksetup -setsecurewebproxystate "Wi-Fi" off
```

**Option C: Charles Proxy**

Charles is a mature GUI proxy well-suited for macOS. It auto-configures system proxy settings on launch and has a clean JSON tree view for API payloads.

```bash
brew install --cask charles
# Launch Charles → Help > SSL Proxying > Install Charles Root Certificate
# Then in Keychain Access, find the Charles cert and set to "Always Trust"
# Charles enables System Proxy automatically on launch
```

---

### macOS, Electron-Based Apps

Many AI apps on macOS (e.g. Claude.app, various AI wrappers) are Electron-based. They accept the same proxy flags as their Linux counterparts:

```bash
# Find the app binary inside the .app bundle
ls /Applications/Claude.app/Contents/MacOS/

# Launch with proxy flags
/Applications/Claude.app/Contents/MacOS/Claude \
  --proxy-server="http://127.0.0.1:8080" \
  --ignore-certificate-errors

# Or set via env before launching
export ELECTRON_EXTRA_LAUNCH_ARGS="--proxy-server=http://127.0.0.1:8080"
open -a Claude
```

---

## 3. CLI-Based AI Agents

Command-line agents, whether Python, Node.js, or Go, almost universally respect standard proxy environment variables. This is the cleanest interception path on both platforms.

### The Universal Method: Environment Variables

```bash
# Start mitmproxy's web UI
mitmweb --listen-port 8080 &

# Export proxy settings (upper and lower case for maximum compatibility)
export HTTP_PROXY=http://127.0.0.1:8080
export HTTPS_PROXY=http://127.0.0.1:8080
export http_proxy=http://127.0.0.1:8080
export https_proxy=http://127.0.0.1:8080

# Point each runtime at the mitmproxy CA cert
export SSL_CERT_FILE=~/.mitmproxy/mitmproxy-ca-cert.pem        # Python
export NODE_EXTRA_CA_CERTS=~/.mitmproxy/mitmproxy-ca-cert.pem  # Node.js
export CURL_CA_BUNDLE=~/.mitmproxy/mitmproxy-ca-cert.pem       # curl

# Run your agent
claude
# or: python3 agent.py / npx your-agent
```

These commands work identically on Ubuntu and macOS.

### Per-Runtime CA Trust Reference

| Runtime | CA Trust Method |
|---|---|
| Python (`requests`, `httpx`) | `export SSL_CERT_FILE=~/.mitmproxy/mitmproxy-ca-cert.pem` |
| Python (`certifi`-based) | `export REQUESTS_CA_BUNDLE=~/.mitmproxy/mitmproxy-ca-cert.pem` |
| Node.js | `export NODE_EXTRA_CA_CERTS=~/.mitmproxy/mitmproxy-ca-cert.pem` |
| curl | `export CURL_CA_BUNDLE=~/.mitmproxy/mitmproxy-ca-cert.pem` |
| wget | `echo "ca-certificate=/path/to/cert.pem" >> ~/.wgetrc` |
| Java-based agents | `keytool -importcert -keystore $JAVA_HOME/lib/security/cacerts -file ~/.mitmproxy/mitmproxy-ca-cert.pem -alias mitmproxy` |
| **System-wide (Ubuntu)** | `sudo cp cert.pem /usr/local/share/ca-certificates/mitmproxy.crt && sudo update-ca-certificates` |
| **System-wide (macOS)** | `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain cert.pem` |

---

## What You'll Actually See

Once traffic is flowing through your proxy, API calls to LLM providers look like this:

```
POST https://api.anthropic.com/v1/messages

Headers:
  x-api-key: sk-ant-...
  anthropic-version: 2023-06-01
  content-type: application/json

Body:
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 8096,
  "system": "You are a helpful assistant...",
  "messages": [
    { "role": "user", "content": "..." }
  ],
  "tools": [...],
  "stream": true
}
```

### Key Things to Audit

- **System prompt contents**, what standing instructions has the agent been given?
- **Tool definitions**, what capabilities are exposed to the model?
- **Data leaving your machine**, are file paths, environment variables, or clipboard content being sent as context?
- **Third-party endpoints**, does the agent call anything besides the primary LLM API?
- **Streaming vs. batch**, `"stream": true` means responses arrive as Server-Sent Events. Your proxy shows these as long-lived connections emitting `data: {...}` chunks.

---

## When Proxying Fails, And How to Fix It

---

### 1. Certificate Pinning

**What it is:** The app bundles a specific TLS certificate or public key hash and refuses any connection that doesn't match it, including your proxy's CA.

**Symptoms:** `SSL handshake failed`, `certificate verify failed`, or the app silently hangs without connecting.

**Workaround, Frida runtime hook (Ubuntu & macOS):**

```bash
# Install Frida
pip3 install frida-tools

# List running processes
frida-ps -a

# Download a universal SSL unpinning script
# Reference: https://github.com/OWASP/owasp-mastg
wget https://raw.githubusercontent.com/nicowillis/Frida-Scripts/main/sslpinning.js \
  -O ssl-bypass.js

# Attach to a running process
frida --attach-pid <PID> -l ssl-bypass.js

# Or spawn with the hook from the start
frida -f /path/to/app-binary -l ssl-bypass.js
```

**Workaround, Binary inspection with Ghidra:**

*Ubuntu:*
```bash
sudo apt install default-jdk -y
wget https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_11.1_build/ghidra_11.1_PUBLIC_20240607.zip
unzip ghidra_11.1_PUBLIC_20240607.zip
cd ghidra_11.1_PUBLIC && ./ghidraRun
```

*macOS:*
```bash
brew install --cask ghidra
# Launch from Applications or: ghidraRun
```

In both cases, load the app binary and search for: `SecTrustEvaluate`, `SSL_CTX_set_verify`, `X509_verify_cert`.

---

### 2. Custom TLS Stacks / Bundled CA Stores

**What it is:** Some apps ship with their own CA root store (common in Go binaries and Python bundles) and ignore the system Keychain or trust store entirely.

**Symptoms:** The proxy cert is trusted system-wide, but this specific app still throws TLS errors through the proxy.

**Workaround, Electron apps (Ubuntu & macOS):**
```bash
/path/to/electron-app \
  --proxy-server=127.0.0.1:8080 \
  --ignore-certificate-errors
```
> ⚠️ Never leave `--ignore-certificate-errors` enabled outside a sandboxed test environment.

**Workaround, Python apps using `certifi` (Ubuntu & macOS):**
```bash
export REQUESTS_CA_BUNDLE=~/.mitmproxy/mitmproxy-ca-cert.pem
export SSL_CERT_FILE=~/.mitmproxy/mitmproxy-ca-cert.pem
python3 your-agent.py
```

**Workaround, Go binaries (Ubuntu & macOS):**
```bash
# Check if the binary is dynamically linked (CGO), if so, system certs may apply
file ./agent-binary
ldd ./agent-binary          # Ubuntu
otool -L ./agent-binary     # macOS

# Try SSL_CERT_FILE, some Go builds respect it
export SSL_CERT_FILE=~/.mitmproxy/mitmproxy-ca-cert.pem
./agent-binary

# Try GODEBUG
export GODEBUG=x509ignoreCN=0
./agent-binary
```

---

### 3. Mutual TLS (mTLS)

**What it is:** The server requires the *client* to present its own certificate. Your proxy intercepts the client side but cannot authenticate to the server as the legitimate client.

**Symptoms:** Connection resets or `403 Forbidden` through the proxy that succeed without it.

**Locate the client certificate:**

*Ubuntu:*
```bash
find ~/.config ~/.local /etc -name "*.pem" -o -name "*.p12" -o -name "*.crt" 2>/dev/null
```

*macOS:*
```bash
# Search filesystem
find ~/Library /etc -name "*.pem" -o -name "*.p12" -o -name "*.crt" 2>/dev/null

# Check the macOS Keychain for client identity certificates
security find-identity -v -p ssl-client
```

**Extract and configure mitmproxy (Ubuntu & macOS):**
```bash
# Extract from PKCS12 bundle
openssl pkcs12 -in client.p12 -out client-cert.pem -clcerts -nokeys
openssl pkcs12 -in client.p12 -out client-key.pem -nocerts -nodes

# Run mitmproxy presenting the client cert upstream
mitmproxy \
  --listen-port 8080 \
  --client-certs /path/to/client-cert.pem
```

---

### 4. Non-HTTP Transports (WebSockets, gRPC, HTTP/3 / QUIC)

**What it is:** The agent uses WebSockets (`wss://`), gRPC (HTTP/2 binary frames), or HTTP/3 (QUIC over UDP), formats a standard HTTP proxy may not decode natively.

**WebSockets**, mitmproxy handles WebSocket upgrades natively on both platforms. Look for `101 Switching Protocols` responses and expand the WebSocket messages panel in mitmweb.

**gRPC (Ubuntu & macOS):**
```bash
# Install grpcurl
# Ubuntu:
curl -sSL \
  https://github.com/fullstorydev/grpcurl/releases/download/v1.9.1/grpcurl_1.9.1_linux_amd64.tar.gz \
  | tar -xz -C /usr/local/bin

# macOS:
brew install grpcurl

# List services
grpcurl -plaintext localhost:50051 list

# Describe a service
grpcurl -plaintext localhost:50051 describe your.Service
```

**HTTP/3 / QUIC, Block UDP 443 to force TCP fallback:**

*Ubuntu (iptables):*
```bash
# Block QUIC
sudo iptables -A OUTPUT -p udp --dport 443 -j REJECT

# Verify
sudo iptables -L OUTPUT -n -v

# Run your proxy normally, clients fall back to TLS over TCP

# Remove when done
sudo iptables -D OUTPUT -p udp --dport 443 -j REJECT
```

*macOS (pf):*
```bash
# Create a pf rule to block outbound UDP 443
echo "block drop out proto udp to any port 443" | sudo pfctl -f -
sudo pfctl -e

# Run your proxy normally

# Restore default pf rules when done
sudo pfctl -f /etc/pf.conf
```

---

### 5. Direct IP Connections / DNS over HTTPS (DoH) Bypass

**What it is:** The agent bypasses DNS entirely (hardcoded IPs) or resolves via DNS over HTTPS, making domain-level blocking ineffective.

**Symptoms:** Traffic appears in Wireshark / tcpdump but not in your proxy. Blocking at `/etc/hosts` has no effect.

**Identify destination IPs:**

*Ubuntu:*
```bash
sudo tcpdump -i eth0 'tcp port 443' -nn -q
sudo tcpdump -i eth0 'port 443' -nn -q      # also catches UDP/QUIC
ss -tupn | grep <process-name>
```

*macOS:*
```bash
# Use en0 for Wi-Fi, en1 for Ethernet, check with: ifconfig
sudo tcpdump -i en0 'tcp port 443' -nn -q
sudo tcpdump -i en0 'port 443' -nn -q

# Check open connections by process
lsof -i -n -P | grep <process-name>
```

**Set up a transparent proxy:**

*Ubuntu (redsocks):*
```bash
sudo apt install redsocks -y
sudo nano /etc/redsocks.conf
# Set: local_ip = 127.0.0.1; local_port = 12345;
#      ip = 127.0.0.1; port = 8080; type = http-relay;
sudo systemctl start redsocks
sudo iptables -t nat -A OUTPUT -p tcp --dport 443 -j REDIRECT --to-port 12345
```

*macOS (redsocks via Homebrew):*
```bash
brew install redsocks
# Edit: /usr/local/etc/redsocks.conf (same format as Ubuntu)
# Set: local_ip = 127.0.0.1; local_port = 12345;
#      ip = 127.0.0.1; port = 8080; type = http-relay;
brew services start redsocks

# Redirect via pf NAT
sudo sh -c 'echo "rdr pass on lo0 proto tcp to any port 443 -> 127.0.0.1 port 12345" | pfctl -f -'
sudo pfctl -e
```

**Block DoH:**

*Ubuntu:*
```bash
sudo iptables -A OUTPUT -d 8.8.8.8 -p tcp --dport 443 -j DROP    # Google DoH
sudo iptables -A OUTPUT -d 1.1.1.1 -p tcp --dport 443 -j DROP    # Cloudflare DoH

sudo apt install dnsmasq -y
echo "no-resolv" | sudo tee -a /etc/dnsmasq.conf
echo "server=127.0.0.1" | sudo tee -a /etc/dnsmasq.conf
sudo systemctl restart dnsmasq
```

*macOS:*
```bash
# Block DoH providers via pf
sudo sh -c 'cat >> /etc/pf.conf << EOF
block drop out proto tcp to 8.8.8.8 port 443
block drop out proto tcp to 1.1.1.1 port 443
EOF'
sudo pfctl -f /etc/pf.conf
sudo pfctl -e

# Install and configure dnsmasq
brew install dnsmasq
echo "no-resolv" >> $(brew --prefix)/etc/dnsmasq.conf
echo "server=127.0.0.1" >> $(brew --prefix)/etc/dnsmasq.conf
sudo brew services start dnsmasq
```

---

### 6. Apps That Deliberately Bypass Proxy Settings

**What it is:** Some hardened agents detect proxy environment variables and route sensitive traffic around them, or use privileged sockets that bypass user-space proxy configuration entirely.

**Symptoms:** Traffic visible in Wireshark / tcpdump but completely absent from mitmproxy despite correct env vars being set.

**Workaround, proxychains:**

*Ubuntu:*
```bash
sudo nano /etc/proxychains4.conf
# Under [ProxyList], add: http  127.0.0.1  8080

proxychains4 python3 agent.py
proxychains4 claude
proxychains4 node agent.js
```

*macOS:*
```bash
# proxychains-ng config is at: /usr/local/etc/proxychains.conf
nano /usr/local/etc/proxychains.conf
# Under [ProxyList], add: http  127.0.0.1  8080

proxychains4 python3 agent.py
proxychains4 claude
proxychains4 node agent.js
```

**Workaround, Network namespace isolation:**

*Ubuntu (Linux namespaces):*
```bash
sudo ip netns add agent-ns
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth1 netns agent-ns
sudo ip addr add 10.0.0.1/24 dev veth0
sudo ip netns exec agent-ns ip addr add 10.0.0.2/24 dev veth1
sudo ip netns exec agent-ns ip link set veth1 up
sudo ip link set veth0 up
sudo ip netns exec agent-ns ip route add default via 10.0.0.1
sudo ip netns exec agent-ns su - $USER -c "claude"
```

*macOS (no network namespaces, use Little Snitch or Lulu instead):*
```bash
# Install Lulu (free, open-source application firewall)
brew install --cask lulu
# Launch Lulu, then run your app, Lulu alerts on every outbound connection attempt
# Block or allow per-connection, per-process
```

**Observe live connections:**

*Ubuntu:*
```bash
lsof -i -n -P | grep <process-name>
ss -tupn | grep pid=<PID>
watch -n 1 "ss -tupn | grep <process-name>"
```

*macOS:*
```bash
lsof -i -n -P | grep <process-name>
# macOS doesn't have ss, use netstat instead
netstat -anp tcp | grep <port>
# Or watch with nettop (built-in macOS tool)
nettop -p <PID>
```

---

## Streaming Responses: A Special Case

When `"stream": true`, the LLM response arrives as Server-Sent Events over a single long-lived HTTP connection. To observe raw SSE chunks directly (works on both platforms):

```bash
curl -X POST https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  --proxy http://127.0.0.1:8080 \
  --cacert ~/.mitmproxy/mitmproxy-ca-cert.pem \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 256,
    "stream": true,
    "messages": [{"role": "user", "content": "Hello"}]
  }' \
  --no-buffer
```

Each SSE chunk looks like:
```
data: {"type":"content_block_delta","delta":{"type":"text_delta","text":"Hello"}}
```

For security auditing, what matters most is the *request* payload, the full system prompt, tool definitions, and message history are all sent at request time, not streamed back.

---

## Defensive Takeaways

Once you have clear visibility into the traffic, audit for the following:

1. **Sensitive data in system prompts**, credentials, internal URLs, or business logic baked into standing instructions are visible to anyone who intercepts the call.
2. **Overly broad tool definitions**, if the agent's tools include filesystem access, shell execution, or database access, that schema appears in every API call.
3. **Context stuffing**, some agents inject large amounts of local context (files, code, clipboard) into every message regardless of relevance. Verify this is intentional.
4. **Third-party calls**, watch for calls to endpoints beyond the primary LLM provider. These may include analytics collectors, telemetry endpoints, or retrieval APIs.
5. **API key exposure in proxy logs**, your mitmproxy logs contain the `x-api-key` header in plaintext. Treat proxy log files with the same care as a secrets store.

---

## Quick Reference: Scenario → Best Tool

| Scenario | Ubuntu | macOS |
|---|---|---|
| Browser agent, first look | Chrome DevTools | Chrome DevTools |
| Browser agent, modify & replay | Burp Suite | Burp Suite |
| Desktop app (native) | mitmproxy + GNOME proxy | Proxyman or Charles |
| Desktop app (Electron) | `--proxy-server` + mitmproxy | `--proxy-server` + mitmproxy |
| CLI agent (Python / Node.js) | mitmproxy + env vars | mitmproxy + env vars |
| Certificate pinning | Frida + SSL bypass script | Frida + SSL bypass script |
| Custom CA / bundled TLS | `REQUESTS_CA_BUNDLE` | `REQUESTS_CA_BUNDLE` |
| mTLS | mitmproxy `--client-certs` | mitmproxy `--client-certs` + `security find-identity` |
| gRPC traffic | grpcurl + Envoy | grpcurl + Envoy |
| HTTP/3 / QUIC block | `iptables` (UDP 443) | `pfctl` (UDP 443) |
| Transparent proxy | redsocks + iptables | redsocks + pf NAT |
| DoH bypass | iptables + dnsmasq | pf + dnsmasq |
| Force proxy (syscall) | proxychains4 | proxychains-ng |
| Network isolation | `ip netns` | Lulu / Little Snitch |
| Live connection watch | `ss`, `lsof`, `watch` | `lsof`, `nettop`, `netstat` |

---

## Public References and Further Reading

### Tools

- **mitmproxy**, https://mitmproxy.org / https://github.com/mitmproxy/mitmproxy
- **mitmproxy Docs**, https://docs.mitmproxy.org/stable/
- **Proxyman (macOS)**, https://proxyman.io
- **Charles Proxy**, https://www.charlesproxy.com
- **Burp Suite Community Edition**, https://portswigger.net/burp/communitydownload
- **Wireshark**, https://www.wireshark.org
- **Frida Dynamic Instrumentation Toolkit**, https://frida.re / https://frida.re/docs/home/
- **Ghidra (NSA Reverse Engineering Tool)**, https://ghidra-sre.org
- **grpcurl**, https://github.com/fullstorydev/grpcurl
- **redsocks (transparent proxy redirector)**, https://github.com/darkk/redsocks
- **proxychains-ng**, https://github.com/rofl0r/proxychains-ng
- **Envoy Proxy**, https://www.envoyproxy.io / https://www.envoyproxy.io/docs/envoy/latest/start/install
- **Lulu (macOS open-source firewall)**, https://objective-see.org/products/lulu.html
- **Little Snitch (macOS)**, https://www.obdev.at/products/littlesnitch/

### API References

- **Anthropic API Reference**, https://docs.anthropic.com/en/api/getting-started
- **Anthropic Streaming Guide**, https://docs.anthropic.com/en/api/streaming
- **OpenAI API Reference**, https://platform.openai.com/docs/api-reference

### Security Standards and Guides

- **OWASP Mobile Application Security Testing Guide (MASTG)**, https://owasp.org/www-project-mobile-app-security/
- **OWASP Web Security Testing Guide**, https://owasp.org/www-project-web-security-testing-guide/
- **OWASP Top 10 for LLM Applications**, https://owasp.org/www-project-top-10-for-large-language-model-applications/
- **PortSwigger Web Security Academy (free)**, https://portswigger.net/web-security
- **NIST AI Risk Management Framework**, https://www.nist.gov/system/files/documents/2023/01/26/AI_RMF_1.0.pdf
- **CWE-295: Improper Certificate Validation**, https://cwe.mitre.org/data/definitions/295.html

### RFCs and Protocol References

- **RFC 8441, Bootstrapping WebSockets with HTTP/2**, https://datatracker.ietf.org/doc/html/rfc8441
- **RFC 9000, QUIC: A UDP-Based Multiplexed and Secure Transport**, https://datatracker.ietf.org/doc/html/rfc9000
- **RFC 9114, HTTP/3**, https://datatracker.ietf.org/doc/html/rfc9114
- **gRPC Core Concepts**, https://grpc.io/docs/what-is-grpc/core-concepts/

### Linux / Ubuntu Specific

- **Linux Network Namespaces (man7)**, https://man7.org/linux/man-pages/man7/network_namespaces.7.html
- **iptables Tutorial**, https://www.netfilter.org/documentation/HOWTO/packet-filtering-HOWTO.html
- **tcpdump Manual**, https://www.tcpdump.org/manpages/tcpdump.1.html
- **Ubuntu Certificate Trust (update-ca-certificates)**, https://manpages.ubuntu.com/manpages/noble/man8/update-ca-certificates.8.html
- **Ubuntu OpenSSL Documentation**, https://help.ubuntu.com/community/OpenSSL

### macOS Specific

- **macOS networksetup man page**, https://ss64.com/osx/networksetup.html
- **macOS security CLI (Keychain)**, https://ss64.com/osx/security.html
- **macOS pf Packet Filter Guide**, https://www.openbsd.org/faq/pf/
- **Apple TLS/SSL Documentation**, https://developer.apple.com/documentation/security/secure_transport
- **macOS Endpoint Security Framework**, https://developer.apple.com/documentation/endpointsecurity
- **Wireshark User's Guide**, https://www.wireshark.org/docs/wsug_html_chunked/

---

*Understanding what your AI agent sends is not paranoia, it's due diligence. The same transparency you'd expect from any other networked software applies here too.*