# 8. Workarounds

Part of the [MAMP PRO FastCGI Runtime Failure investigation](../README.md).

Previous: [← Root Cause Analysis](07-root-cause-analysis.md) · Next: [Vendor Report →](09-vendor-report.md)

---

Each workaround is recorded with its intent, its outcome, and the reason it was
accepted or rejected. Outcomes that were directly verified during the
investigation are tagged **Confirmed**. Options that were evaluated but whose
final outcome on the affected host was not verified are tagged **Evaluated —
outcome requires confirmation**, so they are not presented as proven results.

The goal of a workaround here is to restore a working local PHP serving path
while the FastCGI defect is unresolved. None of these change the root cause; they
route around it.

## W1 — Hosts file repair

- **Intent:** restore the missing local host entries and name resolution.
- **Outcome (Confirmed):** resolution was restored; the FastCGI failure persisted.
- **Decision:** **Accepted as a necessary fix** for a separate defect, but
  **rejected as a fix for the FastCGI failure**, because it did not change the
  FastCGI behavior. Necessary, not sufficient.

## W2 — Clean MAMP restart

- **Intent:** clear any transient state by fully stopping and restarting the
  stack.
- **Outcome (Confirmed):** the FastCGI failure persisted after a clean restart.
- **Decision:** **Rejected.** A restart does not address a runtime/compatibility
  defect in the FastCGI path.

## W3 — Direct CGI execution

- **Intent:** run PHP through the CGI path to confirm the runtime and to serve
  scripts without FastCGI.
- **Outcome (Confirmed):** CGI execution works.
- **Decision:** **Accepted as a diagnostic** (it is the basis of the decisive
  differential test) and usable for one-off script execution. **Not a general
  serving solution**, because it does not provide a persistent local web server
  workflow.

## W4 — PHP built-in server (`php -S`)

- **Intent:** serve the application without Apache and without FastCGI.
- **Outcome (Confirmed):** WordPress rendered correctly through the built-in
  server.
- **Decision:** **Accepted as the primary temporary workaround** for local
  development. Limitations to be aware of: single-process/limited concurrency,
  not intended for production, and it does not reproduce Apache-specific behavior
  (rewrite rules, `.htaccess`, modules). Adequate for continuing development work.

## W5 — PHP-FPM instead of the FastCGI/`php-cgi` path

- **Intent:** replace the `php-cgi`-based FastCGI path with PHP-FPM, a different
  process model, behind Apache (`mod_proxy_fcgi`) or another front end.
- **Outcome (Evaluated — outcome requires confirmation):** PHP-FPM uses a
  persistent pool rather than Apache-spawned `php-cgi` workers, so it may bypass
  the specific failing path. Whether the affected OS/tooling change also affects
  PHP-FPM was not verified on the affected host.
- **Decision:** **Candidate**, pending a verification run. If adopted, document
  the exact result (success/failure and configuration) in [Evidence](06-evidence.md).

## W6 — Homebrew-based Apache + PHP stack

- **Intent:** replace MAMP PRO's bundled binaries with a Homebrew-managed Apache
  and PHP, rebuilt for the current OS/toolchain.
- **Outcome (Evaluated — outcome requires confirmation):** a stack compiled
  against the updated system may avoid a bundled-binary compatibility problem.
- **Decision:** **Candidate with trade-offs.** It moves development off MAMP PRO
  and requires reconfiguring virtual hosts, PHP versions, and the database.
  Reasonable as a medium-term option; not adopted as the immediate fix.

## W7 — Laravel Valet

- **Intent:** use Valet (Nginx + PHP-FPM) as a lightweight local environment that
  does not use Apache FastCGI at all.
- **Outcome (Evaluated — outcome requires confirmation):** Valet avoids the
  Apache/`php-cgi` FastCGI path entirely by using Nginx and PHP-FPM.
- **Decision:** **Candidate with trade-offs.** It changes the web server (Nginx,
  not Apache) and the workflow; suitable if Apache-specific behavior is not
  required. Not adopted as the immediate fix.

## Summary

| Workaround | Outcome | Decision |
| --- | --- | --- |
| W1 Hosts repair | Resolution restored; FastCGI failure persisted | Accepted (separate fix); rejected for FastCGI |
| W2 Clean MAMP restart | No change | Rejected |
| W3 Direct CGI execution | Works | Accepted as diagnostic; not a serving solution |
| W4 PHP built-in server | App renders | **Accepted (primary temporary workaround)** |
| W5 PHP-FPM | Plausible bypass | Candidate — requires confirmation |
| W6 Homebrew stack | Plausible bypass | Candidate — trade-offs |
| W7 Laravel Valet | Plausible bypass | Candidate — trade-offs |

**Recommended immediate workaround:** the PHP built-in server (W4) for local
development, with PHP-FPM (W5) as the next option to validate if an Apache-based
workflow is required.

---

Previous: [← Root Cause Analysis](07-root-cause-analysis.md) · Next: [Vendor Report →](09-vendor-report.md)
