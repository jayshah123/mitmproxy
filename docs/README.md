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
