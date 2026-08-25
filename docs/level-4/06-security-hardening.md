# Security Hardening — TLS & Secure Storage

A device that phones home over the internet, holds provisioned
credentials, and sits physically accessible in the field is a real
attack surface. Security hardening here means TLS for the network
side, encrypted storage for the credentials side, disabling debug
interfaces that make physical attacks easier, and thinking through what
an attacker with each level of access could actually do. Reviewed
against MicroPython's documented `ssl` module support and general
embedded threat-modeling practice; the certificate-handling and
threat-model logic below is verified with `python3` where the logic is
port-agnostic.

## TLS with `ssl` on constrained devices

```python
import ssl
import socket

def tls_connect(host, port, ca_cert_path=None):
    sock = socket.socket()
    sock.connect(socket.getaddrinfo(host, port)[0][-1])
    context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    if ca_cert_path:
        with open(ca_cert_path, "rb") as f:
            context.load_verify_locations(cadata=f.read())
    else:
        context.verify_mode = ssl.CERT_NONE   # see the warning below
    return context.wrap_socket(sock, server_hostname=host)
```

`ssl.CERT_NONE` disables certificate verification entirely — the
connection is encrypted but authenticates nothing, meaning a
man-in-the-middle attacker can transparently intercept it. It exists in
practice because some very memory-constrained MicroPython builds or
older ports have historically had limited or no certificate-chain
verification support, and some prototypes skip it to move faster — but
shipping a production device with `CERT_NONE` gives up TLS's actual
security property while paying its CPU/RAM cost, which is close to the
worst of both outcomes. Loading and verifying against a specific
pinned CA (or better, pinning the server's exact certificate/public key
for a device that only ever talks to one known server) is the correct
production configuration.

## Certificate handling realities

TLS handshakes and certificate verification are measurably heavier on
a microcontroller than on a server — both in CPU time (RSA/ECC
operations) and in RAM (the certificate chain and handshake buffers
must fit in an already-small heap). Practical mitigations:

- **Pin a single CA or a single certificate** rather than trusting a
  full public CA bundle — a full bundle is large and, for a device
  that only ever talks to one backend, verifying against dozens of
  irrelevant CAs is wasted memory and wasted verification time for no
  security benefit.
- **Use ECC certificates over RSA** where the backend supports it —
  ECC signature verification is typically cheaper on constrained CPUs
  for equivalent security strength.
- **Reuse TLS sessions** (session resumption) across reconnects where
  the library supports it, rather than paying a full handshake's cost
  every time a flaky connection drops and reconnects.

```python
def load_pinned_cert(context, cert_path):
    with open(cert_path, "rb") as f:
        context.load_verify_locations(cadata=f.read())
    # From here, only a server presenting a certificate chaining to
    # this specific cert (or CA) will be accepted — not "any valid
    # certificate from any trusted CA," which is the right posture
    # for a device with exactly one known backend.
```

## Encrypted storage — what MicroPython does and doesn't give you

MicroPython's filesystem is, on most ports and by default, **not**
encrypted at rest — a `.py` or `.json` file on flash is readable by
anyone with physical access and the right tools to dump flash contents,
same as any plaintext file on any storage medium. Where the underlying
chip provides hardware support for encrypted storage (ESP32's flash
encryption and its NVS encryption, for instance), that's a
**chip/toolchain-level** feature enabled at build/provisioning time,
not something toggled from MicroPython application code — it's
configured in the ESP-IDF build for that board (`sdkconfig` settings
for flash encryption and secure boot), and it's the honest answer to
"is my secrets.json actually protected" more often than any
application-level scheme:

```python
# Application-level obfuscation is NOT the same as encryption at
# rest, and shouldn't be presented as equivalent protection —
# XOR-with-a-constant, for instance, defeats nothing but the most
# casual inspection and must never be relied on as the real control.
```

Where hardware flash encryption is available and enabled for the
target chip, secrets stored via the normal filesystem API benefit from
it transparently — that's the mechanism to reach for, not an
application-level cipher implemented in Python competing with actual
silicon-backed protection.

## Disabling debug interfaces

A device shipping to the field with its REPL reachable over UART with
no authentication, or with JTAG/SWD debug access left enabled, hands an
attacker with physical access a much easier path than reverse-engineering
firmware from scratch:

- **Disable or password-gate the REPL** on production builds — some
  ports support disabling the UART REPL or gating it behind a
  challenge; leaving an unauthenticated interactive Python prompt
  reachable from a physical UART header is a significant exposure on
  any device that isn't explicitly meant to be end-user-hackable.
- **Disable JTAG/SWD** (or burn the relevant efuse/lock bit the chip
  provides for this, on chips that support it) once a product build is
  finalized — leaving hardware debug access open defeats every other
  hardening measure, since it typically allows dumping flash contents
  directly, encryption settings aside.
- **Remove or disable secure-boot bypass paths** used during
  development (a "hold this pin at boot to force recovery mode" is
  useful during bring-up and dangerous left fully open in a shipped
  product without at least a physical-access requirement matching the
  product's actual threat model).

## Threat-modeling an IoT product

A structured threat model doesn't need to be elaborate to be useful —
walking through attacker capability levels and what each realistically
achieves is often enough to prioritize correctly:

| Attacker capability | What's exposed if hardening is missing |
|---|---|
| Network-only (no physical access) | Everything TLS/cert-pinning is meant to prevent: MITM, credential theft in transit |
| Physical access, no debug tools | UART REPL if left open — full code execution and filesystem access |
| Physical access + JTAG/SWD | Full flash dump regardless of REPL state — recovers secrets unless flash encryption is enabled |
| Access to one compromised unit's provisioned credentials | Should not compromise other units — per-device identity and revocable, not shared, credentials |

The last row is the test most designs fail without noticing: if
extracting secrets from **one** physical unit lets an attacker
impersonate the **entire fleet** to your backend (a shared API key
baked into firmware, say), the design has already failed regardless of
how good the TLS configuration is — per-device credentials
(provisioning module) are what contain a single-unit compromise to that
one unit.

## Cheat sheet

| Hardening step | Defends against |
|---|---|
| Real cert verification, not `CERT_NONE` | Man-in-the-middle on the network path |
| Pinned CA/cert, ECC over RSA | Handshake cost and memory on constrained hardware |
| Chip-level flash encryption (where available) | Physical flash extraction reading secrets in plaintext |
| Disabled/gated REPL and JTAG in production | Physical-access code execution and flash dumping |
| Per-device credentials, not shared | One compromised unit becoming a fleet-wide compromise |

## Exercise

In plain Python, write a small function `classify_exposure(has_tls_verification, has_open_repl, has_jtag_enabled, shared_credentials)` that returns a list of specific risk strings based on which flags are `True` (e.g. `"MITM possible: TLS verification disabled"`, `"physical code execution: REPL open"`, `"flash dump exposes secrets: JTAG enabled"`, `"single-unit compromise becomes fleet-wide: shared credentials"`). Call it with a few different flag combinations, including the worst case (all `True`) and the best case (all `False`), and print the resulting risk lists to confirm the function correctly reports zero risks in the hardened configuration and all four in the worst case.
