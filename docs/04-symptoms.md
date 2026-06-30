# 4. Symptoms

Part of the [MAMP PRO FastCGI Runtime Failure investigation](../README.md).

Previous: [← Timeline](03-timeline.md) · Next: [Investigation Log →](05-investigation.md)

---

Symptoms are recorded as observed. Two epistemic tags are used:

- **Confirmed (captured)** — backed by a stored artifact in
  [Evidence](06-evidence.md); the relevant `E#` reference is cited.
- **Observed (not captured)** — observed during the investigation but without a
  stored artifact in this repository. These are reported as observations, not as
  captured evidence, and are not used to support the root-cause conclusion.

## S1 — MAMP PRO reports an Apache startup failure

**Observed (not captured).** The MAMP PRO interface reported that Apache had
failed to start. The UI state was not stored as an artifact; the contradicting
process/socket state, however, is captured (S2, S3).

## S2 — Apache is actually running

**Confirmed (captured — [E1](06-evidence.md)).** Independent process inspection
showed Apache running, directly contradicting S1. The MAMP PRO startup-failure
report is therefore a false negative with respect to the actual process state.

## S3 — Apache is listening on ports 8888 and 8890

**Confirmed (captured — [E2](06-evidence.md)).** Two Apache listeners were bound:
port 8888 (HTTP) and port 8890 (HTTPS). Apache was accepting connections.

## S4 — Requests served through FastCGI fail

**Confirmed (captured — [E3](06-evidence.md)) for the server-side failure.**
Requests routed through Apache to PHP via FastCGI fail: server-side, the Apache
logs record `idle timeout (30 sec)` and `incomplete headers (0 bytes)` for the
`php8.3.30.fcgi` server (see S9). The corresponding client-facing response is an
HTTP 500 (the expected result of empty headers); a literal client-side capture
was not stored ([E13](06-evidence.md), marked Unknown). The failure originates in
the serving path rather than in the application logic (see S7–S8).

## S5 — Incomplete `hosts` file

**Observed (not captured).** The `hosts` file was missing expected local entries.
This affected name resolution and was repaired (see [Timeline](03-timeline.md)
T2–T3). It was a separate defect and not the cause of S4. No `hosts`-file artifact
was stored in this repository.

## S6 — The FastCGI failure persists after host resolution is restored

**Observed (not captured).** Repairing the `hosts` file restored resolution but
did not resolve the FastCGI failures. This step is recorded as an observation; the
persistent failure itself is independently captured in [E3](06-evidence.md).

## S7 — PHP works outside Apache

**Confirmed (captured — [E5](06-evidence.md), [E6](06-evidence.md)).** PHP
executed correctly through the CLI and direct-CGI paths, outside Apache.

## S8 — A full application renders outside FastCGI

**Observed (not captured).** WordPress rendered correctly when served outside the
FastCGI path, indicating the application and runtime are healthy on their own. The
runtime health is independently captured (S7); the specific WordPress render was
not stored as an artifact.

## S9 — FastCGI timeout behavior

**Confirmed (captured — [E3](06-evidence.md)).** When the same execution is routed
through Apache's `mod_fastcgi` path, the exchange times out and returns empty
headers rather than an application-level error. The captured signatures are:

```text
FastCGI: comm with server ".../php8.3.30.fcgi" aborted: idle timeout (30 sec)
FastCGI: incomplete headers (0 bytes) received from server ".../php8.3.30.fcgi"
AH00524: Handler for fastcgi-script returned invalid result code 53
```

These appear 55 / 55 / 76 times respectively in the current log, across multiple
local projects, dated 2026-04-04 to 2026-06-13. The "incomplete headers (0
bytes)" plus "idle timeout" combination is the distinguishing behavior between the
failing path (FastCGI) and the working path (CGI/CLI).

## Symptom matrix

| Path | Outcome | Status |
| --- | --- | --- |
| MAMP PRO status indicator | Reports Apache failed to start | Observed (not captured) |
| Apache process / sockets (8888, 8890) | Running and listening | Confirmed (E1, E2) |
| Request → Apache → FastCGI → PHP | Fails: idle timeout / incomplete headers | Confirmed (E3) |
| PHP via CLI/CGI (outside Apache) | Works | Confirmed (E5, E6) |
| WordPress via non-FastCGI path | Renders correctly | Observed (not captured) |

---

Previous: [← Timeline](03-timeline.md) · Next: [Investigation Log →](05-investigation.md)
