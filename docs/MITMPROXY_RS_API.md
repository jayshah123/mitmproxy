# mitmproxy_rs API and Contract (for mitmproxy)

This document captures the runtime API contract between Python mitmproxy and the Rust extension package `mitmproxy_rs`.

Scope:

- mitmproxy checkout: this repository.
- Rust module version used here: `mitmproxy_rs==0.12.9` (from `uv.lock`).
- Upstream source reference used for this doc: `mitmproxy/mitmproxy_rs` tag `v0.12.9` (`7e7cc99f8b1d71edad1558ab090e95ebd174fec9`).

## 1. Why mitmproxy_rs Exists

`mitmproxy_rs` is used for performance- and platform-critical capabilities that are difficult or inefficient in pure Python:

- WireGuard server mode.
- Local process redirect mode (Windows/macOS/Linux integration).
- TUN interface mode (Linux).
- UDP transport primitives.
- DNS resolver wrapper with async + error mapping.
- High-performance syntax highlighting and binary contentviews.
- Process discovery and icon extraction for UI.

## 2. Exported Python Surface

Top-level module: `mitmproxy_rs`

Submodules:

- `certs`
- `dns`
- `local`
- `process_info`
- `tun`
- `udp`
- `wireguard`
- `contentviews`
- `syntax_highlight`
- `Stream` class

Primary bindings are defined in upstream `mitmproxy-rs/src/lib.rs` (PyO3 module registration).

## 3. High-Level Runtime Architecture

```mermaid
flowchart LR
    PY[Python callbacks] <-->|Stream API| RPY[mitmproxy-rs bindings]
    RPY --> TASK[PyInteropTask]
    TASK --> CHAN[Transport command/event channels]
    CHAN --> NET[NetworkTask + smoltcp stack]
    NET --> SRC[Packet source task]
    SRC --> OS[UDP socket / WireGuard / Local redirector / TUN]
```

Key idea:

- Packet source implementations feed network events into a shared network layer.
- Network layer normalizes TCP/UDP behavior and exposes stream-like operations to Python callbacks.

## 4. Contract: Stream API

`Stream` is intentionally similar to asyncio streams.

Methods:

- `await read(n)`
  - TCP: returns up to `n` bytes.
  - UDP: returns one datagram; `n` is ignored.
  - Returns `b""` on closed connection or shutdown.
- `write(data)`
  - Queues outbound data.
  - Raises `OSError` if closed/shutdown.
- `await drain()`
  - Waits until write buffer can accept more.
  - Raises `OSError` if stream/server is unavailable.
- `write_eof()`
  - TCP: half-close request.
  - UDP: no-op behavior from transport perspective.
- `close()`
  - Full close request.
- `is_closing()`
  - True after half/full close requested.
- `await wait_closed()`
  - Currently a no-op completion shim (does not track peer close acknowledgement).
- `get_extra_info(name, default)`
  - Always: `transport_protocol`, `peername`, `sockname`.
  - WireGuard streams: `original_src`, `original_dst`.
  - Local redirect streams: `pid`, `process_name`, `remote_endpoint`.
  - Unknown key raises `KeyError` unless `default` provided.

Important caveat:

- `Stream` `Drop` calls `close()` best-effort, so dangling Python references may trigger closure on GC.

## 5. Contract: Server Objects (`UdpServer`, `WireGuardServer`, `LocalRedirector`, `TunInterface`)

Common lifecycle:

- Constructed asynchronously by `start_*` / `create_tun_interface`.
- `close()` requests graceful shutdown (idempotent).
- `wait_closed()` resolves when internal tasks terminate.

Contract notes:

- Shutdown signaling uses watch channels and coordinated task joins.
- On internal task failure, shutdown is triggered and error is logged.
- Graceful path tries to flush pending writes before termination.

## 6. Contract: Callback Invocation

Python handlers passed into Rust (`handle_tcp_stream`, `handle_udp_stream`) are:

- Called once per newly established stream.
- Executed as Python awaitables bridged into Tokio.
- Exceptions are logged by Rust; `asyncio.CancelledError` is treated as expected cancellation.
- Handler task completion removes active stream state.

## 7. Transport/NIO Contract Internals

Internal message contract (Rust side):

- `TransportEvent::ConnectionEstablished`
- `TransportCommand::{ReadData, WriteData, DrainWriter, CloseConnection}`

Channel/backpressure model:

- Python-facing transport events are bounded channels.
- If event channel capacity is unavailable, packet sources may drop inbound data and log warnings.
- Command channel for stream writes is unbounded (needed because Python `write()` is sync).

TCP behavior:

- Implemented with `smoltcp` virtual device.
- Separate buffering layer maps fixed-size smoltcp buffers to Python stream expectations.
- `close()` semantics prefer FIN-like behavior; hard-abort semantics are minimized.

UDP behavior:

- Connection identity is synthetic (4-tuple, LRU tracked).
- Idle UDP entries expire (timeout).
- `drain()` resolves immediately for UDP.

## 8. Submodule Contracts

### 8.1 `udp`

- `start_udp_server(host, port, handle_udp_stream) -> await UdpServer`
- `open_udp_connection(host, port, local_addr=None) -> await Stream`
- Supports IPv4/IPv6 binding logic; includes Windows-specific recv error workaround for reset cases.

### 8.2 `wireguard`

- `genkey() -> base64 private key`
- `pubkey(private_key) -> base64 public key`
- `start_wireguard_server(...) -> await WireGuardServer`

WireGuard runtime behavior:

- Uses `boringtun` tunnel engine.
- Peer mapping established from handshake/static key associations.
- Outbound destination peer resolution falls back to first configured peer if no direct mapping exists.

### 8.3 `local`

- `start_local_redirector(handle_tcp_stream, handle_udp_stream) -> await LocalRedirector`
- `LocalRedirector.describe_spec(spec)` validates and describes intercept spec.
- `set_intercept(spec)` updates active include/exclude rules.
- `unavailable_reason()` returns platform/privilege reason when unsupported.

Intercept spec contract:

- Comma-separated actions.
- Include token: `name` or `pid`.
- Exclude token: `!name` or `!pid`.
- Matching uses process name substring or exact PID.

Platform implementations:

- Windows: elevated redirector process via named pipe (`runas`).
- macOS: system extension/redirector app IPC over Unix sockets.
- Linux: external redirector with privileged startup and Unix datagram IPC.

### 8.4 `tun`

- `create_tun_interface(handle_tcp_stream, handle_udp_stream, tun_name=None) -> await TunInterface`
- Linux only.
- Requires sufficient privileges unless attaching to pre-configured persistent interface.
- Applies Linux-specific interface tuning (`rp_filter`, `route_localnet`, `accept_local`).

### 8.5 `dns`

- `DnsResolver(name_servers=None, use_hosts_file=True)`
- `lookup_ip`, `lookup_ipv4`, `lookup_ipv6` are async.
- `get_system_dns_servers()`

Error contract:

- Domain/no-data/connectivity failures are surfaced as `socket.gaierror` with mapped EAI codes.

### 8.6 `process_info`

- `active_executables() -> list[Process]`
- `executable_icon(path) -> bytes` (PNG)

Availability:

- `active_executables`: Windows/macOS/Linux.
- `executable_icon`: Windows/macOS.

Returned process fields:

- `executable`, `display_name`, `is_visible`, `is_system`.

### 8.7 `syntax_highlight`

- `highlight(text, language) -> list[(tag, chunk_text)]`
- `languages() -> list[str]`
- `tags() -> list[str]`

Supported languages in 0.12.9:

- `css`, `javascript`, `xml`, `yaml`, `none`, `error`.

### 8.8 `contentviews`

Exposes Rust-backed contentviews with mitmproxy metadata bridge:

- `hex_dump` (read-only)
- `hex_stream` (interactive reencode)
- `msgpack` (interactive)
- `protobuf` (interactive)
- `grpc` (interactive)

Python metadata contract consumed by Rust contentviews:

- `content_type`
- `http_message.headers`
- `flow.request.path`
- `protobuf_definitions`
- request/response context checks

### 8.9 `certs`

- `add_cert(pem)` and `remove_cert()`
- macOS-oriented certificate trust integration.

## 9. Platform/Privilege Contract Summary

- Local redirect mode:
  - Windows/macOS supported natively.
  - Linux supported with privileged helper path.
- TUN mode:
  - Linux only.
- Some APIs intentionally raise `NotImplementedError` on unsupported platforms.

## 10. Integration Points in This Repo

Main callsites in mitmproxy:

- Proxy runtime/modes: `mitmproxy/proxy/server.py`, `mitmproxy/proxy/mode_servers.py`, `mitmproxy/proxy/mode_specs.py`
- DNS addon: `mitmproxy/addons/dns_resolver.py`
- UI process metadata: `mitmproxy/tools/web/app.py`
- Syntax highlighting and contentviews:
  - `mitmproxy/addons/dumper.py`
  - `mitmproxy/tools/console/flowview.py`
  - `mitmproxy/contentviews/__init__.py`

## 11. Stability Expectations

Observed compatibility strategy:

- Python stubs (`*.pyi`) are shipped and should be treated as the user-facing contract.
- Internal Rust message/task/channel details are implementation details and may change between `mitmproxy_rs` releases.
- For this repo, always align docs/code assumptions with the pinned version in `uv.lock`.

