# MAMP PRO — Apache FastCGI Runtime Failure (Apple Silicon)

A reproducible engineering investigation of an Apache + `mod_fastcgi` runtime
failure in MAMP PRO on Apple Silicon. The investigation is intentionally framed
as version-independent: it documents the failure, the isolation method, and the
evidence, rather than tying the problem to a single operating-system release.

The local development stack stopped serving PHP applications through Apache, while
the same PHP binaries and applications executed correctly outside the FastCGI
execution path. This repository documents the symptoms, the investigation, the
evidence, the root-cause analysis, and the workarounds that were evaluated.

An operating-system and developer-tooling update is what made the failure
blocking during active development. However, historical log analysis shows the
same FastCGI failure signature occurring **before** that update. The update is
therefore treated as the moment the problem was *exposed or amplified*, not as
its cause. The exact trigger remains unknown.

Author: Alfredo Cisneros
Date: 2026-06-29
Status: **Open** — failure isolated to the Apache → `mod_fastcgi` → `php-cgi`
execution path; the underlying mechanism is a working hypothesis pending vendor
confirmation.

---

## Summary

| Field | Value |
| --- | --- |
| Component | MAMP PRO 7.4.3 local stack (Apache 2.4.66 + `mod_fastcgi` + PHP 8.3.30) |
| Platform | Apple M1 (arm64) — exact OS build recorded in [Environment](docs/02-environment.md) |
| FastCGI module | `mod_fastcgi-SNAP-0910052141` (a 2009 snapshot) |
| Primary symptom | FastCGI requests fail: `idle timeout (30 sec)` + `incomplete headers (0 bytes)` |
| Key differentiator | PHP works through the CLI/CGI path; the FastCGI path fails |
| Observed range | FastCGI failures logged 2026-04-04 to 2026-06-13, across multiple projects |
| Relation to OS update | Update exposed/amplified the failure during development; evidence shows it predated the update |
| Impact | Local development through MAMP PRO is blocked for PHP applications |
| Confidence in isolation | High (differential testing + log evidence) |
| Confidence in mechanism | Low–medium (hypothesis; requires vendor confirmation) |

---

## Documents

1. [Executive Summary](docs/01-executive-summary.md)
2. [Environment](docs/02-environment.md)
3. [Timeline](docs/03-timeline.md)
4. [Symptoms](docs/04-symptoms.md)
5. [Investigation Log](docs/05-investigation.md)
6. [Evidence](docs/06-evidence.md)
7. [Root Cause Analysis](docs/07-root-cause-analysis.md)
8. [Workarounds](docs/08-workarounds.md)
9. [Vendor Report](docs/09-vendor-report.md)
10. [Lessons Learned](docs/10-lessons-learned.md)

---

## How to read this report

The report separates information by epistemic status so that other engineers can
distinguish what was proven from what is inferred. The following tags are used
consistently throughout the documents:

- **Confirmed** — directly observed and verified on the workstation.
- **Hypothesis / Likely** — consistent with the evidence but not proven.
- **Unknown** — not established from this system; the command or capture needed to
  establish it is provided where applicable.

## Evidence-capture note

The environment versions, process and socket state, FastCGI module banner,
configuration, wrapper, and the failure log signatures (with counts and date
range) in [Evidence](docs/06-evidence.md) were captured directly from the
workstation and are reproduced literally. Each value lists the command that
produces it, so the report is auditable. The few items that were not captured in
this pass — a live `curl -v` of the failing request and a live `SELECT VERSION();`
query — are marked **Unknown** with the reason, rather than presenting an
unverified value as fact. The report contains no fabricated output.

The exact operating-system version and build are recorded as captured evidence in
[Environment](docs/02-environment.md). They are intentionally kept out of the
report's title and framing so the investigation is not read as specific to one OS
release; see the open questions in
[Root Cause Analysis](docs/07-root-cause-analysis.md).

## Reproduction at a glance

The detailed, step-by-step procedure is in the
[Investigation Log](docs/05-investigation.md). At a high level:

1. Confirm Apache is actually running and listening, independently of the MAMP
   PRO status indicator.
2. Request a PHP endpoint through Apache/`mod_fastcgi` and observe the FastCGI
   failure (`idle timeout (30 sec)` / `incomplete headers (0 bytes)` in the logs).
3. Execute the same PHP through the CLI/CGI path and observe success.
4. Compare the two execution paths to isolate the failure to FastCGI.

## Repository structure

```
mamp-pro-fastcgi-macos26-bug/
├── README.md            # This index
├── LICENSE
└── docs/
    ├── 01-executive-summary.md
    ├── 02-environment.md
    ├── 03-timeline.md
    ├── 04-symptoms.md
    ├── 05-investigation.md
    ├── 06-evidence.md
    ├── 07-root-cause-analysis.md
    ├── 08-workarounds.md
    ├── 09-vendor-report.md
    └── 10-lessons-learned.md
```

## License

See [LICENSE](LICENSE).
