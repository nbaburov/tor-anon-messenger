# Tor Anonymous Messenger

A terminal chat that runs over a Tor onion service, with encrypted messages and traffic-analysis countermeasures, plus a packet-capture tool that measures what a network observer can still see.

Research proof of concept (2025). The research write-up is [on Notion](https://nbaburov.notion.site/Research-Document-1cf258cb140c8007bca7fcdd6543120e?pvs=74).

> Shared as a reference. Not actively maintained for external contributions, and not audited: do not rely on it for real-world anonymity.

## Video

https://github.com/user-attachments/assets/641436fe-7abd-4170-9aea-6da263156596

## What it does

One peer runs as a server and publishes a Tor onion service; the others connect with a connection string (`<onion address>:<key>`). Nobody's IP address is exposed to the other side.

| Layer | What the code does |
|---|---|
| Network | Tor onion service via `stem`; new circuit period 30 s, forced circuit refresh every 5 min, distinct-subnet relays, IPv6 off, Tor logging to `/dev/null` |
| Encryption | Fernet (AES-128-CBC + HMAC-SHA256) with a key shared in the connection string. Forward secrecy is not implemented. |
| Traffic analysis | messages padded to 512 B / 1 KB / 2 KB / 4 KB, 0.5–3 s random send delay, dummy messages every 30–60 s |
| Local traces | Python logging disabled, best-effort memory zeroing and `mlockall` when available, temporary Tor data directories removed on exit |

`pentest_anon_messenger.py` captures loopback traffic while the messenger runs, then reports what an observer can infer (packet sizes, timing, Tor detection, correlation) with a 0–100 score and optional plots.

![Traffic visibility analysis](security_plots/traffic_visibility_analysis.png)

## Quickstart

Requires Python 3.10+ and Tor (`brew install tor` / `sudo apt install tor`).

```bash
git clone https://github.com/nbaburov/tor-anon-messenger.git
cd tor-anon-messenger
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

python anon_messenger.py --server                     # prints the connection string
python anon_messenger.py --client "<onion>:<key>"     # in another terminal
```

Run with no arguments for interactive mode. Type `quit` or `/exit` to leave.

Security test (needs root for packet capture, plus `pip install -r pentest_requirements.txt`):

```bash
sudo python3 pentest_anon_messenger.py --duration 60 --plots --report security_analysis.json
```

Flags: `--quick` (30 s), `--interface <name>` (default loopback), `--plot-dir <dir>`.

## Architecture

```mermaid
graph LR
    C[Client] -->|SOCKS5| TC[Tor]
    TC -->|3-hop circuit| HS[Onion service]
    HS --> TS[Tor] --> S[Server :8080 on localhost]
```

| Component | Responsibility |
|---|---|
| `TorManager` | starts or reuses Tor, creates the onion service, refreshes circuits |
| `SecureMessenger` | key handling, encryption, padding, timing jitter, dummy traffic |
| `AnonymousServer` | accepts clients over the onion service and relays messages |
| `AnonymousClient` | connects through Tor's SOCKS port and runs the chat UI |

Full diagram, threat model, vulnerability assessment and pentest methodology: [`docs/security-analysis.md`](docs/security-analysis.md).

## Configuration

No environment variables or config files. Ports are chosen automatically (SOCKS from 9050 upward, the control port after it); the server listens on `localhost:8080` behind the onion service.

## License

MIT: see [LICENSE](LICENSE). For education and research; you are responsible for complying with local law.
