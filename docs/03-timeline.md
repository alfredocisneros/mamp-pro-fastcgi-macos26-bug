# 3. Timeline

Part of the [MAMP PRO FastCGI Runtime Failure investigation](../README.md).

Previous: [← Environment](02-environment.md) · Next: [Symptoms →](04-symptoms.md)

---

The timeline reconstructs the investigation chronologically and records the
reasoning at each major decision point, including hypotheses that were later
rejected. Rejected hypotheses are retained because they document which causes
were eliminated and why.

## T0 — Failure became blocking during development

The PHP development stack stopped serving applications through Apache during
active development, around the time of an operating-system and developer-tooling
update. The MAMP PRO interface reported that Apache had failed to start.

- **Initial assumption:** Apache failed to start (as reported by MAMP PRO).
- **Decision:** verify Apache independently before trusting the status
  indicator.
- **Confirmed finding (added on review):** the captured Apache logs show the same
  FastCGI failure signature dated **2026-04-04 to 2026-06-13** across multiple
  projects (see [Evidence](06-evidence.md) E3). The failure therefore **predates**
  the update that made it blocking; it is not a new failure introduced at T0.
- **Supported hypothesis:** the update *exposed or amplified* an existing runtime
  problem during development.
- **Unknown:** the exact trigger and whether the update changed anything
  causally. This is tracked in [Root Cause Analysis](07-root-cause-analysis.md).

## T1 — Apache verified as actually running

Independent inspection showed Apache processes present and two listeners bound on
ports 8888 and 8890, contradicting the MAMP PRO status indicator.

- **Result:** the "Apache failed to start" report was a **false negative** in the
  MAMP PRO health check. Apache was healthy at the process and socket level.
- **Hypothesis rejected:** "Apache is not running." Eliminated by direct process
  and port inspection.
- **Decision:** because Apache was up but requests still failed, shift focus to
  what happens *after* the request reaches Apache.

## T2 — DNS and hosts investigation

Because some requests targeted named hosts, name resolution was examined as a
possible cause of the failures.

- **Finding:** the `hosts` file was **incomplete**; expected local host entries
  were missing.
- **Decision:** repair host resolution before drawing further conclusions, to
  remove name resolution as a variable.

## T3 — Hosts repair and validation

A repair procedure was developed to restore the missing host entries, and
resolution was then re-validated.

- **Result:** host resolution was **restored** and verified.
- **Hypothesis rejected:** "name resolution is the cause of the 500s." The
  incomplete `hosts` file was a real and separate defect, but repairing it did
  **not** resolve the HTTP 500 responses (see T4). It was therefore necessary but
  not sufficient.

## T4 — HTTP 500 persists with resolution restored

With name resolution restored, requests served through Apache continued to return
HTTP 500.

- **Decision:** systematically verify each layer (Apache config, Apache runtime,
  database, PHP runtime) in isolation to localize the fault.

## T5 — Apache configuration and runtime verified

The Apache configuration was validated and the server runtime was confirmed
healthy.

- **Result:** the configuration was **valid** and Apache itself was **healthy**.
- **Hypothesis rejected:** "invalid Apache configuration." Eliminated by
  configuration validation plus the confirmed running state.

## T6 — Database layer verified

MySQL was checked to rule out a database-side failure surfacing as HTTP 500.

- **Result:** MySQL was **functioning correctly**.
- **Hypothesis rejected:** "database outage causing 500s." Eliminated.

## T7 — PHP verified outside Apache (CLI/CGI)

The PHP runtime was exercised directly, outside Apache, including direct CGI
execution.

- **Result:** PHP **executed correctly** outside Apache. A full PHP application
  (WordPress) **rendered correctly** when served outside the FastCGI path.
- **Hypothesis rejected:** "the PHP binary or the application is broken."
  Eliminated; the runtime and the application are healthy on their own.

## T8 — Differential test: CGI vs FastCGI

The same PHP execution was compared across two paths: direct CGI execution versus
the Apache FastCGI path.

- **Result:** the CGI path **succeeded**; the FastCGI path **timed out** and
  produced HTTP 500.
- **Interpretation:** the fault is specific to the FastCGI execution path, not to
  PHP, the application, Apache configuration, the database, or name resolution.

## T9 — FastCGI inspection

The FastCGI path was inspected more closely: Apache and SSL error logs, process
inspection, PHP child processes, and socket verification.

- **Result:** the FastCGI module was identified as `mod_fastcgi`, build
  `mod_fastcgi-SNAP-0910052141` (a 2009 snapshot). The error logs record `idle
  timeout (30 sec)`, `incomplete headers (0 bytes)`, and `invalid result code 53`
  for the `php8.3.30.fcgi` server — behavior consistent with FastCGI workers
  failing to service requests rather than returning application errors.
- **Status:** the precise mechanism is not yet proven. See
  [Root Cause Analysis](07-root-cause-analysis.md).

## T10 — Current state

The failure is reproducibly isolated to the FastCGI path. The underlying cause is
a working hypothesis pending confirmation, and a vendor report has been prepared.
Workarounds were evaluated; see [Workarounds](08-workarounds.md).

---

Previous: [← Environment](02-environment.md) · Next: [Symptoms →](04-symptoms.md)
