# 9. Vendor Report

Part of the [MAMP PRO FastCGI Runtime Failure investigation](../README.md).

Previous: [← Workarounds](08-workarounds.md) · Next: [Lessons Learned →](10-lessons-learned.md)

---

The following is the engineering report prepared for the MAMP PRO team. It is
written to be self-contained so the vendor can act on it without reading the rest
of the repository, while linking back to the supporting detail.

---

**To:** MAMP PRO engineering team
**From:** Alfredo Cisneros
**Date:** 2026-06-29
**Subject:** Apache `mod_fastcgi` runtime failure on Apple Silicon (MAMP PRO
7.4.3) — PHP works via CLI/CGI, fails via FastCGI

### 1. Summary

On an Apple Silicon machine, the MAMP PRO stack fails to serve PHP applications
through Apache: requests routed through the FastCGI path fail. The MAMP PRO
interface also reports that Apache failed to start, while Apache is in fact
running and listening on ports 8888 and 8890.

The failure became blocking around the time of an operating-system and developer-
tooling update, but the Apache logs show the same FastCGI failure signature
**before** that update (see section 3). The update is treated as having exposed or
amplified the problem during development, not as its cause; the exact trigger is
unknown.

Through differential testing, the failure has been isolated to the Apache →
`mod_fastcgi` → `php-cgi` execution path. The same PHP runtime and applications
work correctly via CLI, via direct CGI, and via the PHP built-in server.

### 2. Affected environment (verified)

- macOS 26.5, build 25F71; Apple M1 (`arm64`).
- Apple Command Line Tools 26.6.0.0.
- MAMP PRO 7.4.3 (build 57687).
- Apache 2.4.66 (Unix); runtime banner:
  `Apache/2.4.66 (Unix) mod_wsgi/5.0.2 Python/3.10 mod_fastcgi/mod_fastcgi-SNAP-0910052141 OpenSSL/1.1.1w`.
- FastCGI module: `mod_fastcgi`, build `mod_fastcgi-SNAP-0910052141` (2009 snapshot).
- PHP 8.3.30 (NTS) active for FastCGI, via the `php8.3.30.fcgi` wrapper running
  `php-cgi` with `PHP_FCGI_CHILDREN=4`, `PHP_FCGI_MAX_REQUESTS=200`.
- OpenSSL 1.1.1w; MySQL 8.0.44.

This investigation was performed using the latest available release of MAMP PRO
(7.4.3, build 57687). The issue therefore cannot be attributed to running an
outdated version of the software.

Full commands and outputs are in [Environment](02-environment.md).

### 3. Observed behavior

1. MAMP PRO reports "Apache failed to start."
2. Apache is actually running and listening on 8888 and 8890 (false negative in
   the status indicator).
3. Requests served through FastCGI fail. The Apache logs record, for the
   `php8.3.30.fcgi` server:
   - `FastCGI: comm with server ".../php8.3.30.fcgi" aborted: idle timeout (30 sec)`
   - `FastCGI: incomplete headers (0 bytes) received from server ".../php8.3.30.fcgi"`
   - `AH00524: Handler for fastcgi-script returned invalid result code 53`
   These span 2026-04-04 to 2026-06-13 and affect multiple local projects.
4. The incomplete `hosts` file was repaired; the FastCGI failures persisted (so
   name resolution is not the cause).

### 4. Isolation (what was ruled out)

Each layer was verified independently and eliminated as the cause:

- Name resolution — repaired and verified.
- Apache configuration — valid (`httpd -t`).
- Apache core — serves non-PHP requests.
- MySQL — answers queries.
- PHP runtime — works via CLI and direct CGI.
- Application — WordPress renders via the PHP built-in server.

The decisive test: for the same script, **direct CGI succeeds** while the
**Apache FastCGI path times out**. Full detail is in
[Investigation Log](05-investigation.md) (Tests 1–12) and
[Root Cause Analysis](07-root-cause-analysis.md).

### 5. Reproduction

```bash
# 1. Confirm Apache is up and listening (independently of the MAMP PRO indicator)
ps aux | grep -i "[h]ttpd"
lsof -nP -i :8888 ; lsof -nP -i :8890

# 2. Fail through FastCGI (server-side: idle timeout / incomplete headers in logs)
curl -v "http://localhost:8888/info.php" -o /dev/null

# 3. Succeed through direct CGI (same PHP)
printf '<?php echo "cgi-direct-ok ".PHP_VERSION."\n";' > /tmp/probe.php
REDIRECT_STATUS=1 SCRIPT_FILENAME=/tmp/probe.php \
  /Applications/MAMP/bin/php/php8.3.30/bin/php-cgi -q /tmp/probe.php       # works

# 4. Succeed through the built-in server (same app)
cd /path/to/app
/Applications/MAMP/bin/php/php8.3.30/bin/php -S 127.0.0.1:9000            # renders
```

### 6. Request to the vendor

To confirm the root cause, the following from the MAMP PRO side would help:

1. Confirmation of which FastCGI component MAMP PRO uses on Apple Silicon for the
   affected PHP version (FastCGI Apache module and `php-cgi` wrapper), and their
   build details.
2. Whether those bundled binaries are known to be affected by recent OS /
   developer-tooling changes (for example, code-signing/hardened-runtime,
   dynamic-library, or process-spawning changes). Note that the failure predates
   the most recent update on this host (section 3), so this is about a
   longer-standing compatibility question, not a single update.
3. Whether a rebuilt FastCGI layer for the current OS/toolchain is available or
   planned.
4. Guidance on the supported configuration to capture the FastCGI worker failure
   (any additional logging the team recommends).

### 7. Supporting evidence

The reproduction commands and the captured artifacts (process list, port
listeners, log signatures with counts and date range, FastCGI module banner,
wrapper, configuration, and direct-CGI/CLI output) are in
[Evidence](06-evidence.md). They can be exported and attached to a support ticket.

### 8. Current assessment

- **Confirmed findings:** the failure is isolated to the `mod_fastcgi` →
  `php-cgi` path; all other layers are healthy. The same failure signature is
  present in logs predating the most recent OS/tooling update, so the failure is
  not new to that update.
- **Supported hypotheses:** the 2009-era `mod_fastcgi` build and the `php-cgi`
  worker model fail to communicate on the current Apple Silicon stack, while PHP
  itself is functional; the OS/tooling update exposed or amplified the existing
  problem during development.
- **Unknowns:** the exact mechanism and the exact trigger. This report does not
  assert that the update introduced the failure, nor does it assert a specific
  mechanism as fact.

---

Previous: [← Workarounds](08-workarounds.md) · Next: [Lessons Learned →](10-lessons-learned.md)
