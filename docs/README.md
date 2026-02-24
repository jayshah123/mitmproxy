# Mitmproxy Documentation

This directory houses the mitmproxy documentation available at <https://docs.mitmproxy.org/>.

## Prerequisites

 1. Install [hugo "extended"](https://gohugo.io/getting-started/installing/). 
 2. Windows users: Depending on your git settings, you may need to manually create a symlink from `/docs/src/examples` to `/examples`.

## Editing docs locally

 1. Make sure the mitmproxy Python package is installed and the virtual python environment was activated. See [CONTRIBUTING.md](../CONTRIBUTING.md#development-setup) for details.
 2. Run `./docs/build.py` to generate additional documentation source files.
 3. Now you can change your working directory to `./docs/src` and run `hugo server -D`.

## Core Runtime Terms (Flow vs Addon vs Hook)

These three terms are easy to mix up, but they have different roles:

- **Addon**: Long-lived plugin object with behavior. It defines hook methods like `request(self, flow)` or `response(self, flow)`.
- **Flow**: Mutable data object for one traffic unit (for example one HTTP transaction).
- **Hook**: A named lifecycle moment where mitmproxy calls addon methods and passes data (often a flow).

In short:

- Addon = code
- Flow = state
- Hook = timing

### Example

In `examples/addons/http-add-header.py`, the addon increments a counter and writes to the current flow's response headers:

```python
class AddHeader:
    def __init__(self):
        self.num = 0

    def response(self, flow):
        self.num = self.num + 1
        flow.response.headers["count"] = str(self.num)
```

What this shows:

- `self.num` belongs to the addon instance and persists across requests.
- `flow` is per-request/per-transaction data.
- `response` is the hook name and indicates when this code runs.

## Quickstart: Common Switches, Config YAML, and Custom Addons

mitmproxy reads config from `~/.mitmproxy/config.yaml` (or `config.yml`) by default.

### Config vs Addons (Important)

- **Config (`config.yaml`)** is data: option values such as `mode`, `listen_port`, `ignore_hosts`, feature flags, and addon-specific option values.
- **Addons** are code: Python files/classes that implement hooks such as `request(self, flow)` and `response(self, flow)`.

How they combine:

1. Put addon script paths under `scripts:` in config (or pass `-s` on CLI).
2. Put addon option values in the same config file.

Can config YAML hold an addon implementation?

- **No**. YAML cannot embed executable addon code.
- **Yes** for referencing addon files via `scripts:`.

Useful commands:

```bash
# Show all options with defaults (canonical option reference)
mitmproxy --options

# Show all command signatures
mitmproxy --commands

# Run with common runtime switches
mitmproxy --mode regular --listen-host 127.0.0.1 --listen-port 8080 --showhost

# Disable upstream TLS verification (debug use only)
mitmproxy -k

# Override options ad hoc
mitmproxy --set block_global=false --set confdir=~/.mitmproxy-dev
```

Common switches (CLI aliases for options):

- `--mode` / `-m`: Proxy mode, e.g. `regular`, `reverse:https://example.com`, `socks5`.
- `--listen-host`: Bind address.
- `--listen-port` / `-p`: Bind port.
- `--showhost`: Display URL host from `Host` header.
- `-k` / `--ssl-insecure`: Do not verify upstream TLS certs.
- `--set key=value`: Set any option explicitly.
- `-s` / `--scripts`: Load custom addon script(s).
- `-q` / `-v`: Quiet / verbose logging.

### Sample `config.yaml`

```yaml
# ~/.mitmproxy/config.yaml
mode:
  - regular

listen_host: 127.0.0.1
listen_port: 8080

showhost: true
ssl_insecure: false

ignore_hosts:
  - '(^|\\.)example\\.com:443$'

scripts:
  - ./examples/addons/http-add-header.py
```

Notes:

- `mode`, `ignore_hosts`, and `scripts` are sequences in YAML.
- Script paths in config are resolved relative to the config file directory.

### Invoke mitmproxy with a custom addon

Minimal addon:

```python
# myaddon.py
class MyAddon:
    def request(self, flow):
        flow.request.headers["x-lab"] = "1"

addons = [MyAddon()]
```

Run with each tool:

```bash
# Interactive TUI
mitmproxy -s ./myaddon.py

# CLI/non-interactive
mitmdump -s ./myaddon.py

# Web UI
mitmweb -s ./myaddon.py
```

Pass addon options at startup:

```bash
mitmdump -s ./myaddon.py --set some_addon_option=true
```
