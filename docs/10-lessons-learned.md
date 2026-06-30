# 10. Lessons Learned

Part of the [MAMP PRO FastCGI Runtime Failure investigation](../README.md).

Previous: [← Vendor Report](09-vendor-report.md) · Back to [index](../README.md)

---

These conclusions are derived from the investigation in this repository. Each one
is tied to a specific point where it changed the outcome.

## 1. Verify infrastructure before assuming an application failure

The MAMP PRO indicator reported an Apache startup failure, but Apache was running
and listening. Trusting the indicator would have sent the investigation in the
wrong direction. Independent verification (`ps`, `lsof`) corrected the premise
early. Treat tool status indicators as signals to verify, not as ground truth.

## 2. Isolate runtime layers independently

The fault was found by testing each layer on its own: name resolution, Apache
configuration, Apache core, MySQL, the PHP runtime, and the application. Verifying
each layer in isolation turns a vague "HTTP 500" into a precise statement about
where the failure is and is not.

## 3. Compare equivalent execution paths

The decisive result came from running the same PHP through two paths: direct CGI
(works) and Apache FastCGI (fails). When two paths should be equivalent and only
one fails, the difference between them is the fault. Differential testing is the
highest-value technique in this investigation.

## 4. Eliminate variables systematically

Name resolution was a real defect, but repairing it did not fix the 500. Fixing
the first problem found is not the same as fixing the reported problem. Continue
eliminating variables until the remaining difference fully explains the symptom.

## 5. Distinguish a timeout from an error

A FastCGI timeout (worker not responding) points to process/IPC behavior, whereas
an application error points to code. Reading the *failure mode*, not just the
HTTP status, narrowed the candidate causes.

## 6. Correlate with recent system changes — carefully

The failure was reported after an OS and developer-tooling update, which is a
useful lead. But the captured logs show the FastCGI failures spanning roughly two
months (2026-04-04 to 2026-06-13), which does not match a single recent event.
Temporal correlation is suggestive, not proof; checking the artifacts kept the
"caused by the latest update" claim honest and demoted it to a hypothesis to
test, not the cause.

## 7. Document evidence before drawing conclusions

Conclusions in this report are separated from the evidence that supports them, and
unproven statements are labeled as hypotheses. Recording the commands and their
results first, and reasoning second, keeps the conclusions auditable and lets
another engineer reach the same conclusion from the same evidence.

## 8. Keep a working path available

Identifying a functioning serving path (the PHP built-in server) preserved the
ability to keep working while the root cause remained open. Restoring productivity
and finishing the root-cause analysis are separate goals; both were addressed.

---

Previous: [← Vendor Report](09-vendor-report.md) · Back to [index](../README.md)
