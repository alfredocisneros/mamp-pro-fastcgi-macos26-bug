# 5. Investigation Log

Part of the [MAMP PRO FastCGI Runtime Failure investigation](../README.md).

Previous: [← Symptoms](04-symptoms.md) · Next: [Evidence →](06-evidence.md)

---

Each test is recorded with four fields:

- **Goal** — what the test was meant to establish.
- **Command** — the exact, reproducible command used.
- **Observed result** — what was observed, as verified during the investigation.
- **Interpretation** — what the result allows us to conclude or rule out.

Failed hypotheses are kept. Eliminating a candidate cause is a result.

The commands assume the MAMP PRO bundled binaries under `/Applications/MAMP`.
Adjust the host names to match the affected host. Each result is marked either
**Confirmed (captured)**, with the supporting `E#` reference in
[Evidence](06-evidence.md), or **Observed (not captured)** when the step was
performed during the investigation but no artifact was stored in this repository.
Two client-side `curl -v` captures (Tests 5 and 11, Path B) were not run during
this pass to avoid altering the active stack and are marked as such.

---

## Test 1 — Is Apache actually running?

**Goal:** verify the Apache process independently of the MAMP PRO status
indicator.

**Command:**

```bash
ps aux | grep -i "[h]ttpd"
```

**Observed result:** Apache (`httpd`) processes were present. **Confirmed
(captured — E1).**

**Interpretation:** the MAMP PRO "Apache failed to start" report is a false
negative. Apache is running. Rejected hypothesis: *Apache is not running.*

> Literal output captured in [Evidence](06-evidence.md) (E1).

---

## Test 2 — Is Apache listening on the expected ports?

**Goal:** confirm Apache is bound and accepting connections on 8888 and 8890.

**Command:**

```bash
lsof -nP -i :8888
lsof -nP -i :8890
```

**Observed result:** listeners were bound on **8888** and **8890**. **Confirmed
(captured — E2).**

**Interpretation:** Apache is reachable at the socket level. The failure is not a
bind/listen failure. Rejected hypothesis: *Apache never bound its ports.*

---

## Test 3 — Is name resolution correct?

**Goal:** determine whether the request failures were caused by name resolution.

**Command:**

```bash
cat /etc/hosts
dscacheutil -q host -a name <local-hostname>
ping -c1 <local-hostname>
```

**Observed result:** the `/etc/hosts` file was **incomplete**; expected local
entries were missing. **Observed (not captured)** — no `hosts`-file artifact was
stored in this repository.

**Interpretation:** a real, separate defect in name resolution. It must be
repaired to remove it as a variable, but it is not yet established as the cause of
the HTTP 500s.

---

## Test 4 — Repair host resolution and re-validate

**Goal:** restore the missing host entries and confirm resolution works.

**Command (representative repair procedure):**

> The following is a representative, idempotent repair that ensures the required
> local entries exist. Replace it with the exact script used on the affected host
> if that artifact is being preserved. It is included as reproduction tooling,
> not as captured evidence.

```bash
#!/usr/bin/env bash
# repair-hosts.sh — ensure required local host entries exist in /etc/hosts.
set -euo pipefail

HOSTS_FILE="/etc/hosts"
BACKUP="/etc/hosts.bak.$(date +%Y%m%d%H%M%S)"

# Required entries (edit to match the project's local hostnames).
REQUIRED_ENTRIES=(
  "127.0.0.1 localhost"
  "::1 localhost"
  # "127.0.0.1 <project-local-hostname>"
)

sudo cp "$HOSTS_FILE" "$BACKUP"
echo "Backup written to $BACKUP"

for entry in "${REQUIRED_ENTRIES[@]}"; do
  host="$(awk '{print $2}' <<<"$entry")"
  if ! grep -qiE "[[:space:]]${host}([[:space:]]|$)" "$HOSTS_FILE"; then
    echo "$entry" | sudo tee -a "$HOSTS_FILE" >/dev/null
    echo "Added: $entry"
  else
    echo "Present: $entry"
  fi
done

sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder 2>/dev/null || true
echo "Host resolution refreshed."
```

**Observed result:** host resolution was **restored** and verified. **Observed
(not captured).**

**Interpretation:** name resolution is now correct. If the HTTP 500s were caused
by resolution, they should now disappear. They did not (Test 5). Rejected
hypothesis: *name resolution is the cause of the 500s.*

---

## Test 5 — Do the HTTP 500s persist after the hosts repair?

**Goal:** re-test the failing request now that resolution is restored.

**Command:**

```bash
curl -v -k "https://<local-hostname>:8890/" -o /dev/null
curl -v "http://localhost:8888/<php-endpoint>" -o /dev/null
```

**Observed result:** requests served through Apache continued to fail. **Observed
(not captured)** for the client-side `curl` (not re-run in this pass); the
persistent server-side failure is independently **captured in E3**.

**Interpretation:** the fault is downstream of name resolution and downstream of
Apache accepting the connection. Focus moves to request handling.

> The client-side `curl -v` capture was not re-run in this evidence pass to avoid
> altering the active stack; the recurring server-side failure is documented in
> [Evidence](06-evidence.md) (E3, marked Unknown for the live capture as E13).

---

## Test 6 — Is the Apache configuration valid?

**Goal:** rule out an invalid Apache configuration.

**Command:**

```bash
/Applications/MAMP/Library/bin/httpd -t
/Applications/MAMP/Library/bin/httpd -S
```

**Observed result:** the configuration test returned `Syntax OK`. **Confirmed
(captured — E12).**

**Interpretation:** the configuration is not the fault. Rejected hypothesis:
*invalid Apache configuration.*

---

## Test 7 — Is Apache itself healthy?

**Goal:** confirm Apache is serving at all (independent of PHP/FastCGI).

**Command:**

```bash
# Request a static asset served directly by Apache (no PHP/FastCGI involved).
curl -v "http://localhost:8888/favicon.ico" -o /dev/null
```

**Observed result:** Apache served the static request. **Observed (not captured)**
— this client-side request was not stored as an artifact; the configuration
validity (E12) and the running/listening state (E1, E2) are captured.

**Interpretation:** the web server core is functional. The fault is specific to
the dynamic (PHP) handling path. Rejected hypothesis: *Apache core is broken.*

---

## Test 8 — Is MySQL functioning?

**Goal:** rule out a database failure surfacing as HTTP 500.

**Command:**

```bash
/Applications/MAMP/Library/bin/mysql -h 127.0.0.1 -P 8889 -u root -p -e "SELECT VERSION();"
```

**Observed result:** MySQL is present and reports `8.0.44` (Evidence E10). During
the original investigation it answered queries normally. **Confirmed** (version);
the live `SELECT VERSION();` query was not re-run in this pass (credentials), so
E10 marks it optional.

**Interpretation:** the database layer is healthy. Rejected hypothesis: *database
outage causes the 500s.*

---

## Test 9 — Does PHP execute correctly outside Apache (CGI/CLI)?

**Goal:** determine whether the PHP runtime itself is broken.

**Command:**

```bash
# CLI
/Applications/MAMP/bin/php/php8.3.30/bin/php -v
/Applications/MAMP/bin/php/php8.3.30/bin/php -r 'echo "cli-ok ".PHP_VERSION."\n";'

# Direct CGI execution of a script (no Apache, no FastCGI)
printf '<?php echo "cgi-direct-ok ".PHP_VERSION."\n";' > /tmp/probe.php
REDIRECT_STATUS=1 SCRIPT_FILENAME=/tmp/probe.php \
  /Applications/MAMP/bin/php/php8.3.30/bin/php-cgi -q /tmp/probe.php
```

**Observed result:** PHP **executed correctly** via CLI (`cli-ok 8.3.30`) and via
direct CGI (`cgi-direct-ok 8.3.30`). **Confirmed** (Evidence E5, E6).

**Interpretation:** the PHP binary and its configuration are healthy. The fault is
not in PHP itself. Rejected hypothesis: *the PHP binary is broken.*

---

## Test 10 — Does a full application render outside FastCGI?

**Goal:** determine whether the application (WordPress) is broken.

**Command:**

```bash
# Serve the application with the PHP built-in server (no Apache, no FastCGI).
cd /path/to/wordpress
/Applications/MAMP/bin/php/php8.3.30/bin/php -S 127.0.0.1:9000
# then request it
curl -v "http://127.0.0.1:9000/" -o /dev/null
```

**Observed result:** WordPress **rendered correctly** outside the FastCGI path.
**Observed (not captured)** — the render was not stored as an artifact; the
runtime health it depends on is captured (E5, E6).

**Interpretation:** the application is healthy. The fault is not in the
application. Rejected hypothesis: *the application is broken.*

---

## Test 11 — Differential test: CGI vs FastCGI (the decisive test)

**Goal:** compare the two execution paths for the same PHP to localize the fault.

**Command:**

```bash
# Path A — direct CGI (works, per Test 9 / Evidence E5)
REDIRECT_STATUS=1 SCRIPT_FILENAME=/tmp/probe.php \
  /Applications/MAMP/bin/php/php8.3.30/bin/php-cgi -q /tmp/probe.php

# Path B — through Apache + mod_fastcgi (the failing path)
curl -v "http://localhost:8888/info.php" -o /dev/null
```

**Observed result:** Path A (CGI) **succeeded** (`cgi-direct-ok 8.3.30`, E5).
Path B (Apache + `mod_fastcgi`) fails: the Apache logs record `idle timeout
(30 sec)` and `incomplete headers (0 bytes)` for the `php8.3.30.fcgi` server
(E3). **Confirmed (captured — E3, E5).** The live Path B `curl -v` capture was
not re-run in this pass; the server-side failure is documented in E3.

**Interpretation:** with PHP, the application, the Apache configuration, the
Apache core, the database, and name resolution all eliminated, the only remaining
difference between success and failure is the **FastCGI execution path**. This is
the decisive isolation result.

---

## Test 12 — Inspect the FastCGI layer

**Goal:** gather signals about how the FastCGI path fails.

**Command:**

```bash
# Apache error and SSL error logs
tail -n 200 /Applications/MAMP/logs/apache_error.log
grep -E "fastcgi:error|invalid result code 53" /Applications/MAMP/logs/apache_ssl_error.log

# FastCGI module + version (from the runtime banner; httpd -M needs the MAMP conf)
grep "resuming normal operations" /Applications/MAMP/logs/apache_error.log | tail -1

# PHP / FastCGI child processes
ps aux | grep -iE "[p]hp-cgi"

# Sockets used by the FastCGI workers
ls -la /Applications/MAMP/Library/logs/fastcgi/
```

**Observed result:** the FastCGI implementation is `mod_fastcgi`, build
`mod_fastcgi-SNAP-0910052141` (E4). `php-cgi` children are running (E8) and the
IPC sockets `fgci8.3.30.sock` / `fgci8.4.17.sock` exist (E9). The error logs
record `idle timeout (30 sec)`, `incomplete headers (0 bytes)`, and `invalid
result code 53` for the `php8.3.30.fcgi` server (E3). The signals are consistent
with FastCGI workers failing to return a response, not with the application
returning an error.

**Interpretation:** the failure mode is a non-responsive FastCGI worker path, not
an application error. The precise cause (worker spawn, library load,
signing/hardened-runtime, or socket handling) is **not yet proven**; see
[Root Cause Analysis](07-root-cause-analysis.md).

> Literal log excerpts, banner, process list, and socket list are reproduced in
> [Evidence](06-evidence.md) (E3, E4, E7–E9).

---

Previous: [← Symptoms](04-symptoms.md) · Next: [Evidence →](06-evidence.md)
