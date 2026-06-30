# 7. Root Cause Analysis

Part of the [MAMP PRO FastCGI Runtime Failure investigation](../README.md).

Previous: [← Evidence](06-evidence.md) · Next: [Workarounds →](08-workarounds.md)

---

This analysis separates what is proven from what is inferred. Assumptions are not
presented as facts. All findings reference verified evidence in
[Evidence](06-evidence.md).

## Confirmed findings

1. Apache is running and listening on ports 8888 and 8890 (E1, E2), even when MAMP
   PRO reports a startup problem. The process and socket state contradict the
   indicator.
2. The FastCGI path fails with three recurring signatures (E3):
   - `FastCGI: comm with server ".../php8.3.30.fcgi" aborted: idle timeout (30 sec)`
   - `FastCGI: incomplete headers (0 bytes) received from server ".../php8.3.30.fcgi"`
   - `AH00524: Handler for fastcgi-script returned invalid result code 53`
3. The same PHP runtime executes correctly outside FastCGI, via direct CGI (E5)
   and via the CLI (E6).
4. The FastCGI implementation in use is `mod_fastcgi`, build
   `mod_fastcgi-SNAP-0910052141` — a 2009 development snapshot (E4, E7).
5. The failing FastCGI server is the `php8.3.30.fcgi` wrapper, which execs
   `php-cgi` with `PHP_FCGI_CHILDREN=4` and `PHP_FCGI_MAX_REQUESTS=200` (E7).
6. The failures are **not** specific to one project: they occur across multiple
   local vhosts routed through the same FastCGI server (E3).
7. The failures span **2026-04-04 to 2026-06-13** (E3) — a period of roughly two
   months, not a single isolated event.
8. The same FastCGI failure signature is present in logs that **predate** the
   operating-system and developer-tooling update that made the issue blocking
   (E3). The failure is therefore not new to the update window.
9. The stack runs the latest available release of MAMP PRO at the time of the
   investigation — **7.4.3, build 57687** (E7, [Environment](02-environment.md)).
   The failure therefore cannot be attributed to an outdated version of the
   software.

**Conclusion from the confirmed findings:** the failure is isolated to the Apache
→ `mod_fastcgi` → `php-cgi` execution path. PHP itself, invoked directly, is
healthy (E5/E6 vs E3). The failure predates the OS update; the update did not
introduce it.

## Supported hypotheses

These are consistent with the evidence but are **not** proven; see Unknowns.

1. **Mechanism — transport/worker incompatibility.** The communication between
   `mod_fastcgi` and the `php-cgi` worker fails: the worker does not return headers
   (`incomplete headers (0 bytes)`) within the FastCGI idle window, the module
   aborts the exchange (`idle timeout (30 sec)`), and Apache surfaces the handler
   failure (`invalid result code 53`). The most likely explanation is an
   incompatibility or instability between the **2009-era `mod_fastcgi` build** and
   the `php-cgi` worker model on the current Apple Silicon stack, rather than a
   fault in PHP, the application, Apache's core, the database, or name resolution —
   each independently eliminated (see [Timeline](03-timeline.md) and
   [Symptoms](04-symptoms.md)).
2. **Exposure — the OS/tooling update.** The update did not introduce the failure
   (Confirmed finding 8), but it plausibly **exposed or amplified** the existing
   problem during active development — for example by changing how often the
   failing path was exercised, or by shifting timing/load such that the existing
   instability became consistently blocking.

## Evidence supporting the conclusion

- **Differential execution (strongest signal):** identical `php-cgi` binary
  succeeds when invoked directly (E5/E6) and fails through `mod_fastcgi` (E3). The
  variable that distinguishes success from failure is the FastCGI transport.
- **Failure mode:** `incomplete headers (0 bytes)` + `idle timeout` is a transport
  / worker-communication failure, not an application error (which would return a
  PHP error or a non-empty body).
- **Module age:** the loaded `mod_fastcgi` is a 2009 snapshot (E4/E7), which is
  consistent with fragility on a current OS and toolchain.
- **Breadth:** the same signature appears across unrelated projects (E3),
  indicating a stack-level cause rather than an application-level one.

## Unknowns

1. **The precise mechanism** by which `php-cgi` returns zero bytes to
   `mod_fastcgi`. Candidates, none yet proven: worker crash on first request, a
   dynamic-library/runtime load failure under the worker, a hardened-runtime or
   code-signing constraint on the spawned worker, or a socket/permission issue on
   the `FastCgiIpcDir` socket.
2. **The exact trigger.** The failure predates the OS update (Confirmed finding
   8), so the update is not the origin. What first introduced the instability, and
   whether the update changed anything causally (exposure/amplification), is not
   established. Closing this requires correlating the first failure timestamp with
   the project's and the stack's change history.
3. **Whether the failure is operating-system-version-specific at all.** The
   evidence does not establish that any single OS release is required to reproduce
   it. The exact OS version and build observed are recorded as evidence in
   [Environment](02-environment.md), but are not claimed to be a precondition.
4. **Whether replacing the transport resolves it** — `mod_proxy_fcgi` + PHP-FPM,
   or a non-MAMP stack (see [Workarounds](08-workarounds.md)).

## Confidence level

| Statement | Confidence | Basis |
| --- | --- | --- |
| Failure is isolated to the `mod_fastcgi`/`php-cgi` path | **High** | E3 vs E5/E6 (differential) |
| PHP, the app, Apache core, DB, DNS are not the cause | **High** | Each verified independently |
| The failure predates the OS update | **High** | Log signature dated before the update (E3) |
| Cause is the 2009 `mod_fastcgi` + `php-cgi` interaction on this stack | **Medium** | Consistent with E3/E4/E7; mechanism unproven |
| The OS update exposed or amplified an existing failure | **Medium** | Hypothesis; consistent with the timing but not proven |
| A specific mechanism (crash, lib load, signing, socket) | **Low** | Not yet tested (Unknown 1) |
| The OS update introduced the failure | **Very low** | Contradicted: the signature predates the update (E3) |

Nothing under "Supported hypotheses" or "Unknowns" should be cited as established
fact until the corresponding item is closed.

---

Previous: [← Evidence](06-evidence.md) · Next: [Workarounds →](08-workarounds.md)
