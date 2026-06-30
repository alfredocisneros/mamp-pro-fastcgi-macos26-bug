# 6. Evidence

Part of the [MAMP PRO FastCGI Runtime Failure investigation](../README.md).

Previous: [← Investigation Log](05-investigation.md) · Next: [Root Cause Analysis →](07-root-cause-analysis.md)

---

Evidence captured on the workstation described in [Environment](02-environment.md)
(Apple M1, macOS 26.5 / 25F71, MAMP PRO 7.4.3). Each item states what it shows,
the command that produced it, and the literal output. Output that was not
captured is marked **Unknown** with the reason.

The log excerpts below are real lines from this system. Their timestamps span
**2026-04-04 to 2026-06-13** across several local projects. This date range is
itself evidence and is discussed in [Root Cause Analysis](07-root-cause-analysis.md).

---

## E1 — Apache process list

**Shows:** Apache is running despite the MAMP PRO failure report.

```bash
ps aux | grep -i "[h]ttpd"
```

```text
root   5052  /Applications/MAMP/Library/bin/httpd -f /Library/Application Support/appsolute/MAMP PRO/conf/httpd.conf -k start
hispahub 29110 .../httpd -f .../httpd.conf -k start
hispahub 29113 .../httpd -f .../httpd.conf -k start
hispahub 29114 .../httpd -f .../httpd.conf -k start
hispahub 29115 .../httpd -f .../httpd.conf -k start
hispahub 29133 .../httpd -f .../httpd.conf -k start
```

## E2 — Port listeners (8888, 8890)

**Shows:** Apache is listening on both ports.

```bash
lsof -nP -i :8888
lsof -nP -i :8890
```

```text
COMMAND  PID     USER  FD  TYPE DEVICE NODE NAME
httpd  29110 hispahub  4u IPv6  ...   TCP *:8888 (LISTEN)
httpd  29113 hispahub  4u IPv6  ...   TCP *:8888 (LISTEN)
...
httpd  29110 hispahub  6u IPv6  ...   TCP *:8890 (LISTEN)
httpd  29113 hispahub  6u IPv6  ...   TCP *:8890 (LISTEN)
```

## E3 — FastCGI failure signatures (Apache SSL error log)

**Shows:** the FastCGI path fails with three recurring signatures.

```bash
grep -E "fastcgi:error|invalid result code 53" "/Applications/MAMP/logs/apache_ssl_error.log"
```

Representative literal lines:

```text
[Sun Jun 07 11:54:59 2026] [fastcgi:error] [client ::1] FastCGI: comm with server
  "/Applications/MAMP/fcgi-bin/php8.3.30.fcgi" aborted: idle timeout (30 sec)
[Sun Jun 07 11:54:59 2026] [fastcgi:error] [client ::1] FastCGI: incomplete headers
  (0 bytes) received from server "/Applications/MAMP/fcgi-bin/php8.3.30.fcgi"
[Sat Jun 13 21:23:36 2026] [core:error] [client ::1] AH00524: Handler for
  fastcgi-script returned invalid result code 53
```

Occurrence counts in the current log:

| Signature | Count |
| --- | --- |
| `idle timeout (30 sec)` | 55 |
| `incomplete headers (0 bytes)` | 55 |
| `invalid result code 53` | 76 |

Date range of these errors: **2026-04-04** (earliest) to **2026-06-13** (latest).
They appear across multiple local hosts (for example `wordpress-dev.elementor`,
`hosting-intersitios`, `hispahub-wordpress`), all routed through the same
`php8.3.30.fcgi` FastCGI server. The failure is not specific to one project.

## E4 — FastCGI behavior at graceful restart (Apache error log)

**Shows:** the FastCGI process manager tears down and re-initializes; the runtime
banner records the module versions.

```bash
tail -n 30 /Applications/MAMP/logs/apache_error.log
```

```text
[Mon Jun 29 17:55:15 2026] [:alert] (4)Interrupted system call: FastCGI: read() from pipe failed (0)
[Mon Jun 29 17:55:15 2026] [:alert] (4)Interrupted system call: FastCGI: the PM is shutting down,
  Apache seems to have disappeared - bye
[Mon Jun 29 17:55:15 2026] [:notice] FastCGI: process manager initialized (pid 29110)
[Mon Jun 29 17:55:15 2026] [mpm_event:notice] AH00489: Apache/2.4.66 (Unix) mod_wsgi/5.0.2
  Python/3.10 mod_fastcgi/mod_fastcgi-SNAP-0910052141 OpenSSL/1.1.1w configured -- resuming normal operations
```

## E5 — Direct CGI execution (the working path)

**Shows:** `php-cgi` runs a script correctly when invoked directly, outside
Apache/FastCGI.

```bash
printf '<?php header("X-Probe: cgi"); echo "cgi-direct-ok ".PHP_VERSION."\n";' > /tmp/research_probe.php
REDIRECT_STATUS=1 SCRIPT_FILENAME=/tmp/research_probe.php \
  /Applications/MAMP/bin/php/php8.3.30/bin/php-cgi -q /tmp/research_probe.php
```

```text
cgi-direct-ok 8.3.30
```

## E6 — CLI execution (the working path)

**Shows:** the PHP runtime is healthy.

```bash
/Applications/MAMP/bin/php/php8.3.30/bin/php -r 'echo "cli-ok ".PHP_VERSION."\n";'
/Applications/MAMP/bin/php/php8.3.30/bin/php -v
```

```text
cli-ok 8.3.30
PHP 8.3.30 (cli) (built: Jan 16 2026 14:08:44) (NTS)
Zend Engine v4.3.30
```

## E7 — FastCGI module and configuration

**Shows:** the exact FastCGI implementation and how it is wired.

```bash
grep -niE 'LoadModule.*fastcgi|FastCgiServer|Action php-fastcgi|FastCgiIpcDir' \
  "/Library/Application Support/appsolute/MAMP PRO/conf/httpd.conf"
cat /Applications/MAMP/fcgi-bin/php8.3.30.fcgi
```

```text
190:LoadModule fastcgi_module modules/mod_fastcgi.so
789:    FastCgiIpcDir /Applications/MAMP/Library/logs/fastcgi
790:    FastCgiServer /Applications/MAMP/fcgi-bin/php8.3.30.fcgi -socket fgci8.3.30.sock
791:    FastCgiServer /Applications/MAMP/fcgi-bin/php8.4.17.fcgi -socket fgci8.4.17.sock
825:    Action php-fastcgi "/fcgi-bin/php8.3.30.fcgi"

# php8.3.30.fcgi
#!/bin/sh
export PHP_FCGI_CHILDREN=4
export PHP_FCGI_MAX_REQUESTS=200
exec /Applications/MAMP/bin/php/php8.3.30/bin/php-cgi -c "/Library/Application Support/appsolute/MAMP PRO/conf/php8.3.30.ini"
```

## E8 — FastCGI child processes

**Shows:** the `php-cgi` workers spawned by the FastCGI process manager.

```bash
ps aux | grep -iE "[p]hp-cgi"
```

```text
hispahub 29112 .../php8.3.30/bin/php-cgi -c .../conf/php8.3.30.ini   (process manager)
hispahub 29123..29126 .../php8.3.30/bin/php-cgi -c .../conf/php8.3.30.ini   (children)
hispahub 29118..29121 .../php8.4.17/bin/php-cgi -c .../conf/php8.4.17.ini
```

## E9 — FastCGI sockets

**Shows:** the IPC sockets used between Apache and the FastCGI servers.

```bash
ls -la /Applications/MAMP/Library/logs/fastcgi/
```

```text
srwxrwxrwx fgci8.3.30.sock
srwxrwxrwx fgci8.4.17.sock
```

## E10 — Database availability

**Shows:** MySQL is installed and reports its version.

```bash
/Applications/MAMP/Library/bin/mysql80/bin/mysql --version
```

```text
mysql  Ver 8.0.44 for macos12.7 on arm64 (Source distribution)
```

> A live `SELECT VERSION();` query was not captured in this pass because it
> requires the database credentials; it is not needed for the isolation result.
> **Status:** version Confirmed; live query **Unknown** (optional).

## E11 — OpenSSL

**Shows:** the TLS library version linked by the stack and seen by PHP.

```bash
/Applications/MAMP/bin/php/php8.3.30/bin/php -r 'echo OPENSSL_VERSION_TEXT, "\n";'
```

```text
OpenSSL 1.1.1w  11 Sep 2023
```

## E12 — Apache configuration validity

**Shows:** the Apache configuration is syntactically valid, ruling out a malformed
config as the cause.

```bash
/Applications/MAMP/Library/bin/httpd -t -f "/Library/Application Support/appsolute/MAMP PRO/conf/httpd.conf"
```

```text
Syntax OK
```

## E13 — HTTP response capture (FastCGI path), live

**Status: Unknown (not captured in this pass).** A fresh `curl -v` against a
running vhost was intentionally not run so the active stack would not be altered
during evidence collection. The recurring server-side failures are already
documented in E3. To capture the client-side view on demand:

```bash
curl -v "http://localhost:8888/<php-endpoint>" -o /dev/null
```

---

## Evidence summary

| ID | Item | Status |
| --- | --- | --- |
| E1 | Apache process list | Confirmed (captured) |
| E2 | Port listeners 8888/8890 | Confirmed (captured) |
| E3 | FastCGI failure signatures + counts + date range | Confirmed (captured) |
| E4 | FastCGI restart + Apache banner | Confirmed (captured) |
| E5 | Direct CGI execution succeeds | Confirmed (captured) |
| E6 | CLI execution succeeds | Confirmed (captured) |
| E7 | FastCGI module + config + wrapper | Confirmed (captured) |
| E8 | php-cgi child processes | Confirmed (captured) |
| E9 | FastCGI sockets | Confirmed (captured) |
| E10 | MySQL version | Confirmed; live query Unknown (optional) |
| E11 | OpenSSL version | Confirmed (captured) |
| E12 | Apache configuration validity (`Syntax OK`) | Confirmed (captured) |
| E13 | Live HTTP 500 capture | Unknown (not run to avoid altering the stack) |

---

Previous: [← Investigation Log](05-investigation.md) · Next: [Root Cause Analysis →](07-root-cause-analysis.md)
