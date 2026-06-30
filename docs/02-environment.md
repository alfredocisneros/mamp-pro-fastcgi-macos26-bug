# 2. Environment

Part of the [MAMP PRO FastCGI Runtime Failure investigation](../README.md).

Previous: [← Executive Summary](01-executive-summary.md) · Next: [Timeline →](03-timeline.md)

---

The values below were verified directly on the workstation used for this
investigation. Each row lists the value and the command that produces it, so the
environment is auditable and reproducible. Items that could not be verified from
this system are marked **Unknown** with the reason.

## Scope note (read first)

All verified values come from a single Apple Silicon workstation and are recorded
here as captured evidence. The operating-system version and build below are
reported exactly as the system returns them; they are documented as facts, **not**
as a precondition for the failure. The investigation is framed as
version-independent (see [README](../README.md)), and whether the failure is
specific to any OS release is an open item in the
[Root Cause Analysis](07-root-cause-analysis.md). If a host on a different build
is also affected, capture its values with the same commands and record them
alongside these.

## Platform

| Item | Verified value | Verification command |
| --- | --- | --- |
| Operating system | macOS 26.5 | `sw_vers -productVersion` |
| Build | 25F71 | `sw_vers -buildVersion` |
| Architecture | `arm64` (Apple Silicon) | `uname -m` |
| Hardware | Apple M1 | `sysctl -n machdep.cpu.brand_string` |
| Apple Command Line Tools | 26.6.0.0.1781586589 | `pkgutil --pkg-info=com.apple.pkg.CLTools_Executables` |
| Command Line Tools path | `/Library/Developer/CommandLineTools` | `xcode-select -p` |

## Local development stack (MAMP PRO)

This investigation was performed using the latest available release of MAMP PRO
(7.4.3, build 57687). The issue therefore cannot be attributed to running an
outdated version of the software.

| Component | Verified value | Verification command |
| --- | --- | --- |
| MAMP PRO | 7.4.3 (build 57687) — latest available release at the time of the investigation | `/usr/libexec/PlistBuddy -c "Print :CFBundleShortVersionString" "/Applications/MAMP PRO.app/Contents/Info.plist"` |
| Apache (`httpd`) | Apache/2.4.66 (Unix), built Dec 19 2025 | `/Applications/MAMP/Library/bin/httpd -v` |
| FastCGI module | `mod_fastcgi` — `mod_fastcgi-SNAP-0910052141` | Apache runtime banner (see below) |
| PHP (active for FastCGI) | 8.3.30 (NTS, built Jan 16 2026) | `/Applications/MAMP/bin/php/php8.3.30/bin/php -v` |
| PHP (also configured) | 8.4.17 | `/Applications/MAMP/bin/php/php8.4.17/bin/php -v` |
| PHP (installed set) | 7.3.33, 7.4.33, 8.3.30, 8.4.17, 8.5.2 | `ls -d /Applications/MAMP/bin/php/php*/` |
| OpenSSL (linked by Apache) | OpenSSL 1.1.1w (11 Sep 2023) | Apache runtime banner; `/Applications/MAMP/bin/php/php8.3.30/bin/php -r 'echo OPENSSL_VERSION_TEXT;'` |
| MySQL | 8.0.44 (arm64) | `/Applications/MAMP/Library/bin/mysql80/bin/mysql --version` |

### Apache runtime banner (authoritative module versions)

Captured from the Apache error log at startup:

```
Apache/2.4.66 (Unix) mod_wsgi/5.0.2 Python/3.10 mod_fastcgi/mod_fastcgi-SNAP-0910052141 OpenSSL/1.1.1w
```

This banner is the source of truth for three facts used in the analysis:

- **FastCGI implementation:** `mod_fastcgi` (the original FastCGI Apache module),
  **not** `mod_fcgid` and **not** `mod_proxy_fcgi`.
- **FastCGI module build:** `mod_fastcgi-SNAP-0910052141`. The suffix is a
  development snapshot dated 2009-10-05. The module loaded by the current stack is
  therefore a 2009-era build of `mod_fastcgi`.
- **TLS library:** OpenSSL 1.1.1w, which reached end of life in September 2023.

> Reproduce the banner: `grep "resuming normal operations" /Applications/MAMP/logs/apache_error.log | tail -1`

## FastCGI configuration (verified)

From `/Library/Application Support/appsolute/MAMP PRO/conf/httpd.conf`:

```apache
LoadModule fastcgi_module modules/mod_fastcgi.so

<IfModule mod_fastcgi.c>
    SetHandler fastcgi-script
    AddHandler fastcgi-script .fcgi .fpl
    FastCgiIpcDir /Applications/MAMP/Library/logs/fastcgi
    FastCgiServer /Applications/MAMP/fcgi-bin/php8.3.30.fcgi -socket fgci8.3.30.sock
    FastCgiServer /Applications/MAMP/fcgi-bin/php8.4.17.fcgi -socket fgci8.4.17.sock
    AddHandler php-fastcgi .php
    Action php-fastcgi "/fcgi-bin/php8.3.30.fcgi"
</IfModule>
```

FastCGI wrapper `/Applications/MAMP/fcgi-bin/php8.3.30.fcgi` (verified):

```sh
#!/bin/sh
export PHP_FCGI_CHILDREN=4
export PHP_FCGI_MAX_REQUESTS=200
exec /Applications/MAMP/bin/php/php8.3.30/bin/php-cgi -c "/Library/Application Support/appsolute/MAMP PRO/conf/php8.3.30.ini"
```

Active FastCGI sockets present at runtime (note the `fgci` spelling is MAMP's own):

```
/Applications/MAMP/Library/logs/fastcgi/fgci8.3.30.sock
/Applications/MAMP/Library/logs/fastcgi/fgci8.4.17.sock
```

## Ports (verified)

Apache was confirmed listening on both ports via `lsof`:

| Service | Port | Verification command |
| --- | --- | --- |
| Apache (HTTP) | 8888 | `lsof -nP -i :8888` |
| Apache (HTTPS) | 8890 | `lsof -nP -i :8890` |

The literal `lsof` output is reproduced in [Evidence](06-evidence.md) (E2).

## One-pass capture

```bash
echo "== OS =="; sw_vers; uname -m; sysctl -n machdep.cpu.brand_string
echo "== Apple tooling =="; xcode-select -p; pkgutil --pkg-info=com.apple.pkg.CLTools_Executables
echo "== MAMP PRO =="; /usr/libexec/PlistBuddy -c "Print :CFBundleShortVersionString" "/Applications/MAMP PRO.app/Contents/Info.plist"
echo "== Apache =="; /Applications/MAMP/Library/bin/httpd -v
echo "== Apache banner =="; grep "resuming normal operations" /Applications/MAMP/logs/apache_error.log | tail -1
echo "== PHP =="; /Applications/MAMP/bin/php/php8.3.30/bin/php -v
echo "== MySQL =="; /Applications/MAMP/Library/bin/mysql80/bin/mysql --version
echo "== Ports =="; lsof -nP -i :8888; lsof -nP -i :8890
```

---

Previous: [← Executive Summary](01-executive-summary.md) · Next: [Timeline →](03-timeline.md)
