# 1. Executive Summary

Part of the [MAMP PRO FastCGI Runtime Failure investigation](../README.md).

Next: [Environment →](02-environment.md)

---

## Issue

During active development on an Apple Silicon machine, the MAMP PRO stack stopped
serving PHP applications through Apache: requests routed through the FastCGI path
failed. The MAMP PRO interface additionally reported that Apache had failed to
start, even though Apache was in fact running and accepting connections.

The failure became blocking around the time of an operating-system and developer-
tooling update. However, historical log analysis shows the same FastCGI failure
signature occurring **before** that update (see [Evidence](06-evidence.md) E3).
The update is treated as the point at which the problem was *exposed or amplified*
during development — not as the cause. The exact trigger remains unknown.

The investigation isolated the failure to the **FastCGI execution path**. The
same PHP runtime and the same applications executed correctly when invoked
outside FastCGI (CLI/CGI and the PHP built-in server). This is the central,
verified finding of the report.

## Environment (verified)

- **Platform:** Apple M1 (arm64). The exact operating-system version and build are
  recorded as captured evidence in [Environment](02-environment.md); they are
  deliberately not used to scope the report's identity.
- **Stack:** MAMP PRO 7.4.3 (build 57687); Apache 2.4.66;
  `mod_fastcgi-SNAP-0910052141` (a 2009 snapshot of `mod_fastcgi`); PHP 8.3.30
  active for FastCGI; OpenSSL 1.1.1w; MySQL 8.0.44.
- **Software currency:** this investigation was performed using the latest
  available release of MAMP PRO (7.4.3, build 57687). The issue therefore cannot
  be attributed to running an outdated version of the software.
- **Relation to the OS update:** the captured FastCGI failures span 2026-04-04 to
  2026-06-13, so they predate the update that made the issue blocking; a single
  recent update is not established as the cause (see
  [Root Cause Analysis](07-root-cause-analysis.md)).

All component versions above were verified on the workstation; full commands and
outputs are in [Environment](02-environment.md).

## Impact

- Local development through MAMP PRO is blocked for PHP applications served via
  Apache/FastCGI.
- The impact is limited to the FastCGI serving path. PHP itself, the PHP
  applications, the Apache configuration, and the database layer were each
  verified as healthy in isolation, so the blast radius is narrow and
  well-defined.

## Current status

**Open.**

- **Confirmed findings:** the failure is isolated to the Apache → `mod_fastcgi` →
  `php-cgi` path; equivalent execution paths (CLI/CGI, PHP built-in server)
  succeed. The FastCGI logs show `idle timeout (30 sec)` and `incomplete headers
  (0 bytes)` for the `php8.3.30.fcgi` server. The same signature is present in
  logs dated before the OS update, so the failure predates the update.
- **Supported hypotheses:** the 2009-era `mod_fastcgi` build and the `php-cgi`
  worker model fail to communicate on the current Apple Silicon stack, while the
  PHP binary itself remains functional; the OS/tooling update exposed or amplified
  the existing problem during development. Both are consistent with the evidence
  but not proven.
- **Unknowns:** the precise underlying mechanism (for example, worker spawn
  failure, code-signing/hardened-runtime constraints, a dynamic-library load
  failure, or socket handling); whether replacing the FastCGI transport (for
  example PHP-FPM via `mod_proxy_fcgi`) resolves it; and the exact trigger.

A formal report prepared for the MAMP PRO team is included in
[Vendor Report](09-vendor-report.md).

---

Next: [Environment →](02-environment.md)
