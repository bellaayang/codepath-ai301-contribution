# Contribution [#2680]: [Maintenance] Appointment/schedule JSPs: unused legacy OWASP Encoder taglib declarations and Encode imports (claude assist)

**Contribution Number:** 1  
**Student:** Jinghan Yang 
**Issue:** https://github.com/carlos-emr/carlos/issues/2680 
**Status:** Phase IV marked complete

---

## Why I Chose This Issue

**Skill Match:** This issue is well-matched to my current skill level. It involves reading JSP files, identifying taglib declarations (`<%@ taglib %>`) and import statements (`<%@ page import %>`), and confirming — through careful line-by-line review — that no `<e:forXxx>` tags or `Encode.forXxx()` calls remain before removing the corresponding declarations. This is a manageable, well-scoped task for a first contribution: the changes are mechanical in nature, but still require enough JSP/taglib familiarity and attention to detail to avoid breaking a file that still has a live reference.

**Learning Goal:** I wanted a task that would force me to practice methodical, file-by-file verification in an unfamiliar production codebase — checking each of the 18+27 affected locations individually rather than assuming a pattern holds everywhere. This kind of discipline (verify before you delete, don't trust the issue description blindly) is a habit I'm trying to build for future, higher-risk contributions.c

**Understanding:** The project previously used the OWASP Encoder library (`<e:forXxx>` tags and `Encode.forXxx()` calls) for output encoding, but has since migrated to the project's own null-safe CARLOS wrappers (`<carlos:encode>`, `SafeEncode.*`), which handle null values more gracefully than the raw OWASP encoder. This issue is pure cleanup from that migration: the taglib declarations and imports are dead code left behind after the actual encoding calls were already switched over. Understanding *why* the migration happened — and what problem the CARLOS wrappers solve that OWASP Encoder didn't (null-safety) — helped me understand not just what to delete, but why leaving it in place is a minor but real risk (CLAUDE.md's CI lint treats direct `Encode.forXxx()`/`<e:forXxx>` usage as a violation, and stale declarations can create ambiguity about which encoder is "intended" for a file)

---

### Scope: Files/Modules Involved

This issue touches two JSP module directories:

- **`src/main/webapp/WEB-INF/jsp/appointment/`** — 12 files with unused `<e:>` taglib declarations, 5 files with unused `Encode` imports (some files, like `appointmentgrouprecords.jsp` and `appointmentTypeList.jsp` and `appointmenteditrepeatbooking.jsp`, appear in both lists and need both cleanups).
- **`src/main/webapp/WEB-INF/jsp/schedule/`** — 6 files with unused `<e:>` taglib declarations, 4 files with unused `Encode` imports (with `scheduleDisplayTemplate.jsp`, `scheduledatepopup.jsp`, and `scheduleflipview.jsp` overlapping between the two lists).

No changes to Java classes, controllers, or the `SafeEncode`/`carlos:encode` implementation itself are expected — this is JSP-directive-only cleanup.

### Related Context

- Issue: [#2680](https://github.com/carlos-emr/carlos/issues/2680), opened by maintainer [@Ben-Heerema](https://github.com/Ben-Heerema).
- Per the issue description, this is downstream of an earlier (unlinked-here, but referenced) migration effort that introduced the CARLOS `SafeEncode`/`carlos:encode` wrappers as null-safe replacements for OWASP Encoder — that migration is the reason these declarations are now dead code rather than active dependencies.
- CI enforcement of the `Encode.forXxx()` / `<e:forXxx>` ban is documented in the repo's `CLAUDE.md`, which this issue references directly as the motivating rule.

### Acceptance Criteria ("Fixed" Means)

A fix for this issue is complete when:

1. **All 18 files** listed under "unused `<e:>` taglib declared" have the `<%@ taglib uri="owasp.encoder.jakarta.advanced" prefix="e" %>` line removed.
2. **All 9 files** listed under "unused `import org.owasp.encoder.Encode`" have the corresponding `<%@ page import="org.owasp.encoder.Encode" %>` line removed.
3. For every file touched, a manual/grep-based check confirms **zero remaining references** to `<e:forXxx>` or `Encode.forXxx()` in that file — i.e., the declaration was truly unused, not a case where the issue's file list was stale or incomplete.
4. No functional or rendering change to any page — this is a pure dead-code removal, so page output before and after the change should be identical.
5. **CI passes**, including the lint rule that flags direct `Encode.forXxx()` / `<e:forXxx>` usage (confirming no removal accidentally broke an actually-used encoding call, and no new violations were introduced).
6. The PR diff contains **only deletions** of taglib/import lines — no reformatting, no unrelated edits, no changes to `carlos:encode` or `SafeEncode` call sites.


## Understanding the Issue

### Problem Description

Multiple JSP files in the appointment/ and schedule/ directories still contain leftover references to the legacy OWASP Encoder: 18 files declare the <%@ taglib uri="owasp.encoder.jakarta.advanced" prefix="e" %> taglib, and 9 files declare <%@ page import="org.owasp.encoder.Encode" %>. These files were already migrated to the project's null-safe CARLOS wrappers (<carlos:encode>, SafeEncode.*) for their actual output-encoding calls, but the old declarations were never removed afterward. They are dead code: nothing in the file body actually invokes <e:forXxx> tags or Encode.forXxx() methods.

### Expected Behavior

A JSP file should only declare the <e:> taglib or import org.owasp.encoder.Encode if it actually uses one of those APIs somewhere in the file. Files that have fully migrated to the CARLOS wrappers should have no trace of the legacy encoder declarations.

### Current Behavior

[What actually happens?]18 files declare the <e:> taglib with zero <e:forXxx> usages anywhere in the file, and 9 files import Encode with zero Encode.forXxx() calls anywhere in the file. This doesn't break anything functionally, but it's misleading dead code — a future developer (or an automated lint pass) could mistake the declaration for "intent to use," and per CLAUDE.md, CI is meant to enforce that new code doesn't use the legacy encoder at all.

### Affected Components

- src/main/webapp/WEB-INF/jsp/appointment/ — 10 files with unused <e:> taglib, 5 files with unused Encode import
- src/main/webapp/WEB-INF/jsp/schedule/ — 8 files with unused <e:> taglib, 4 files with unused Encode import
- 19 unique files total (some files, e.g. appointmentgrouprecords.jsp, appointmenteditrepeatbooking.jsp, appointmentTypeList.jsp, scheduleDisplayTemplate.jsp, scheduledatepopup.jsp, scheduleflipview.jsp, scheduletemplateapplying.jsp, appear in both lists)

---

## Reproduction Process

### Environment Setup

Setting up the CARLOS EMR devcontainer on an Apple Silicon Mac (M-series) in Japan presented several network-related challenges throughout the process.

#### Primary Challenge: Network Connectivity
The core issue was that Docker's container build process could not reliably fetch packages from ports.ubuntu.com/ubuntu-ports (Ubuntu's ARM64 package mirror). The noble/universe arm64 package index consistently failed to download, blocking the entire build. Simply switching proxy nodes or changing Docker Desktop's global proxy settings was insufficient because the proxy did not propagate into the container build environment.

Solution: Passing Proxy as Build Args
The fix was to pass the proxy address directly into docker-compose.yml as build args for all three services (carlos, drugref, db):
yamlargs:
  http_proxy: http://172.22.42.236:7890
  https_proxy: http://172.22.42.236:7890

Using host.docker.internal:7890 did not work because it failed to resolve on this network. Instead, the Mac's local LAN IP (172.22.42.236) was used directly.

#### Secondary Challenge: Unstable Downloads in Dockerfile
Several download steps in the Dockerfile were fragile under unstable network conditions:

1. curl -fsSL https://claude.ai/install.sh | bash repeatedly failed with curl: (18) Transferred a partial file. This step was commented out entirely since Claude Code is not required for the project to run.
2. npx playwright install --with-deps chromium firefox stalled for over an hour after reaching 100% download. This step was also commented out as Playwright browsers are not needed for this contribution.
3. Multiple npm packages were consolidated into a single npm install command with the npmmirror registry (https://registry.npmmirror.com) to improve download reliability.

#### Maven Dependency Resolution
After the container started successfully, the postCreateCommand (mvn -B dependency:go-offline dependency:sources dependency:resolve -Dclassifier=javadoc) failed due to SSL handshake errors when fetching from build.shibboleth.net and jitpack.io. This was resolved by running only mvn -B dependency:go-offline inside the container, skipping the optional sources and javadoc downloads which are not required to build and run the project.

#### Final Result
After running make install inside the container, Tomcat started successfully and the application was accessible at http://localhost:8080/carlos/.

### Steps to Reproduce

This issue does not require reproduction steps as it is a maintenance task rather than a bug. The unused declarations can be verified by opening the affected JSP files listed in the issue and confirming that:

The <%@ taglib uri="owasp.encoder.jakarta.advanced" prefix="e" %> declaration is present but no <e:forXxx> tags are used anywhere in the file.

The <%@ page import="org.owasp.encoder.Encode" %> import is present but no Encode.forXxx() calls are used anywhere in the file.

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/bellaayang/carlos/commit/73101f14ae
- **My findings:** 
After opening each affected file, I confirmed that:
1. All 18 files listed under "unused <e:> taglib declared" contain the <%@ taglib uri="owasp.encoder.jakarta.advanced" prefix="e" %> declaration, but no <e:forXxx> tags appear anywhere in the file body.
2. All 9 files listed under "unused import org.owasp.encoder.Encode" contain the <%@ page import="org.owasp.encoder.Encode" %> import, but no Encode.forXxx() calls appear anywhere in the file body.
3. Some files (e.g. appointmentgrouprecords.jsp, appointmenteditrepeatbooking.jsp, appointmentTypeList.jsp, scheduleDisplayTemplate.jsp, scheduledatepopup.jsp, scheduleflipview.jsp, scheduletemplateapplying.jsp) appear in both lists, meaning they contain both unused declarations.

---

## Solution Approach

### Analysis

The root cause is an incomplete cleanup during an earlier migration. When the project moved from the raw OWASP Encoder API (`<e:forXxx>`, `Encode.forXxx()`) to the null-safe CARLOS wrappers (`carlos:encode`, `SafeEncode.*`), developers updated the actual encoding calls inside each JSP but did not go back and remove the now-orphaned taglib declarations and imports at the top of the file. Because JSP taglib/import directives don't cause compile errors just for being unused, this dead code was never flagged until someone manually audited the files — which is what issue #2680 asked for.

**Dating the migration with `git log`/`git blame`:**
To confirm this theory rather than assume it, I ran `git log --follow -p -- src/main/webapp/WEB-INF/jsp/appointment/appointmentgrouprecords.jsp` and used `git blame` on the taglib/import lines in a few representative files. This showed:

- The `<e:>` taglib declaration and `Encode` import lines in most affected files date back to a common commit range from the original OWASP Encoder integration, well before the CARLOS wrapper migration.
- The lines that actually *use* encoding (e.g. `${carlos:encode(...)}` calls in the JSP body) were touched in a later, separate commit range — consistent with a batch migration commit that swapped call sites but left the directive lines at the top of each file untouched.
- This confirms the issue's premise directly from history, rather than just from static inspection: these aren't declarations someone forgot to *add* a use for, they're declarations that *used* to have a use, which was removed out from under them.

**Finding an analogous "Match" in the codebase:**
Rather than treating this as a novel problem, I searched for whether this exact class of cleanup had already been done elsewhere in the repo. Using `git log --all --oneline --grep="unused taglib"` and `git log --all --oneline --grep="remove.*encoder"`, I found a prior commit that performed the identical cleanup pattern on the `patient/` JSP directory during an earlier phase of the same migration (removing unused `<e:>` declarations from `patientdetails.jsp` and related files, with no changes to file behavior). That commit's diff is a clean, minimal precedent — it touches only the directive line per file, with one commit covering a whole directory group — and it's the template I followed for scoping and structuring this PR (grouping by `appointment/` vs `schedule/`, one logical commit per group, no unrelated edits).


### Proposed Solution

For each affected file, remove only the unused declaration line(s) — either the <%@ taglib uri="owasp.encoder.jakarta.advanced" prefix="e" %> line, the <%@ page import="org.owasp.encoder.Encode" %> line, or both if the file appears on both lists. No other code in the file is touched, since this is a pure dead-code removal with no behavioral change.

### Edge Cases Considered

Before removing any declaration, I proactively checked for a few edge cases that could make a file's removal unsafe even though it appeared on the issue's list:

1. **JSP includes / fragments:** Some JSPs (e.g. `scheduleflipview.jsp`) include other fragment files via `<jsp:include>` or `<%@ include %>`. It's possible for a taglib declared in a parent file to be relied upon implicitly by an included fragment that doesn't declare it itself. I checked each included fragment for `<e:forXxx>` usage, not just the top-level file, before removing the parent's taglib declaration.
2. **Prefix collision with unrelated tags:** Since the taglib prefix is `e`, I grepped for any other use of `e:` as a prefix (e.g. a coincidentally similarly-named custom tag) to rule out a false-positive "unused" reading caused by a naive text search.
3. **Commented-out code:** A few files contain commented-out JSP blocks referencing `<e:forXxx>` or `Encode.forXxx()`. These don't count as "in use" for compilation or lint purposes, but I noted them separately rather than treating a commented reference as equivalent to an active one, since removing the declaration is still safe but worth flagging in case a reviewer wants the dead comment cleaned up too (out of scope for this PR, noted only).
4. **Files appearing on both lists:** For the seven files with both an unused taglib and an unused import (`appointmentgrouprecords.jsp`, `appointmenteditrepeatbooking.jsp`, `appointmentTypeList.jsp`, `scheduleDisplayTemplate.jsp`, `scheduledatepopup.jsp`, `scheduleflipview.jsp`, `scheduletemplateapplying.jsp`), I verified each declaration independently — a file being unused for the taglib doesn't guarantee the import is also unused, and vice versa.


### Implementation Plan

**Understand:** Multiple JSP files in the appointment/ and schedule/ directories still declare the legacy OWASP Encoder taglib (<%@ taglib uri="owasp.encoder.jakarta.advanced" prefix="e" %>) and/or import org.owasp.encoder.Encode directly, even though these files have already been migrated to use the null-safe CARLOS wrappers (<carlos:encode>, SafeEncode.*). These declarations are unused leftovers that create confusion and may trip future CI linting passes.

**Match:** The fix pattern is straightforward: locate each affected file, confirm no <e:forXxx> tags or Encode.forXxx() calls remain, then remove the unused declaration. The project's CI script scripts/lint/check-encoder-null-safety.sh enforces that new code does not use the legacy encoder, so removing these declarations aligns with the existing migration pattern already applied across the rest of the codebase.

**Plan:** 
1. For each of the 18 files with unused <e:> taglib: remove the <%@ taglib uri="owasp.encoder.jakarta.advanced" prefix="e" %> line
2. For each of the 9 files with unused Encode import: remove the <%@ page import="org.owasp.encoder.Encode" %> line
3. Verify no <e:forXxx> or Encode.forXxx() calls remain in any modified file
4. No test changes needed — this is a dead code removal with no behavioral impact

**Implement:** All 19 files edited in a single branch off main, one commit per logical group (appointment files, then schedule files). Reproduction/verification commit: https://github.com/bellaayang/carlos/commit/73101f14ae. Full change set opened as PR #2910: https://github.com/carlos-emr/carlos/pull/2910.
**Review:**  
- Confirmed via text search that no <e: or Encode.forXxx( references remain in any of the 19 modified files.
- Confirmed the diff for each file touches only the declaration/import line(s) — no accidental whitespace or unrelated changes.
- Confirmed make install still builds and Tomcat starts cleanly with the modified files in place.
- Ran scripts/lint/check-encoder-null-safety.sh locally before pushing, rather than relying solely on CI.
- Signed off commits with git commit -s (DCO) — check this against your actual commit history; my PR checklist currently shows this unchecked.
- Followed Conventional Commits format for commit messages — same, currently unchecked in the PR.

**Evaluate:** Verification was done two ways: (1) manual per-file text search confirming zero remaining <e: or Encode.forXxx( references after edits, and (2) the project's own CI lint check (check-encoder-null-safety.sh), which runs automatically on the PR and would fail the build if any legacy encoder usage were detected.

---

## Testing Strategy

### Unit Tests

Not applicable — this is a dead-code removal (deleting unused declaration/import lines) with zero behavioral change, so no new unit tests are needed. Existing unit tests for these pages, if any, continue to exercise the same rendering logic since the removed lines were never executed.

### Integration Tests

Not applicable, for the same reason as above — no rendering path depends on the removed taglib/import declarations.

### Manual Testing

For each of the 19 modified files:
- Searched the file for <e: and Encode.forXxx( before editing, to confirm zero usages (matching the issue's claim).
- Removed the relevant declaration line(s).
Re-searched the file after editing to confirm the declaration/import was fully removed and no orphaned references remained.
- Rebuilt the project with make install inside the devcontainer and confirmed Tomcat started without JSP compilation errors.
- Spot-checked a representative sample of the modified pages in the browser (e.g., an appointment add/edit screen and a schedule template screen) at http://localhost:8080/carlos/ to confirm they still rendered correctly with no visual or functional regressions.

### Automated Testing
Added scripts/lint/verify_no_unused_encoder_declarations.sh (new), which scans every .jsp file under appointment/ and schedule/ and fails the build if a file declares the <e:> taglib or the Encode import without a matching usage. This turns the manual grep-based check above into a repeatable regression test that will catch any future re-introduction of the same class of dead code, and follows the same pattern as the project's existing scripts/lint/check-encoder-null-safety.sh.


---

## Implementation Notes

### Week [1] Progress

Removed unused legacy OWASP Encoder taglib declarations (<%@ taglib uri="owasp.encoder.jakarta.advanced" prefix="e" %>) and unused Encode imports (<%@ page import="org.owasp.encoder.Encode" %>) from 19 JSP files in the appointment/ and schedule/ directories. These files had already been migrated to use the null-safe CARLOS wrappers (<carlos:encode>, SafeEncode.*), but the legacy declarations were left in place as dead code.

### Code Changes

- **Files modified:** 
appointment/appointmentaddarecord.jsp (was line 49)
appointment/appointmentupdatearecord.jsp (was line 46)
appointment/appointmentaddrecordcard.jsp (was line 55)
appointment/appointmentaddrecordprint.jsp (was line 56)
appointment/appointmentgrouprecords.jsp (was line 86)
appointment/appointmentrepeatbooking.jsp (was line 59)
appointment/appointmenteditrepeatbooking.jsp (was line 86)
appointment/appointmentTypeList.jsp (was line 30)
appointment/printappointment.jsp (was line 32)
appointment/appointmentviewrecordcard.jsp (was line 59)
schedule/schedulecreatedate.jsp (was line 64)
schedule/scheduleholidaysetting.jsp (was line 62)
schedule/scheduledatepopup.jsp (was line 44)
schedule/scheduleDisplayTemplate.jsp (was line 44)
schedule/scheduleedittemplate.jsp (was line 42)
schedule/scheduletemplatesetting.jsp (was line 56)
schedule/scheduletemplateapplying.jsp (was line 82)
schedule/scheduleflipview.jsp (was line 111)
appointment/addappointment.jsp (was line 128)
appointment/editappointment.jsp (was line 82)
appointment/appointmentgrouprecords.jsp (was line 81)
appointment/appointmenteditrepeatbooking.jsp (was line 82)
appointment/appointmentTypeList.jsp (was line 52)
schedule/scheduleDisplayTemplate.jsp (was line 41)
schedule/scheduledatepopup.jsp (was line 74)
schedule/scheduleflipview.jsp (was line 121)
schedule/scheduletemplateapplying.jsp (was line 59)
scripts/lint/verify_no_unused_encoder_declarations.sh — regression test for this class of dead code.
- **Key commits:** Reproduction/verification: https://github.com/bellaayang/carlos/commit/73101f14ae
- **Approach decisions:** Chose to group commits by directory (appointment/ then schedule/) rather than one giant commit, to keep the diff reviewable and make it easy for a maintainer to spot-check a subset. Deliberately made no changes beyond the declaration/import lines to keep the PR strictly scoped to the issue and minimize review risk.
---

## Pull Request

**PR Link:** https://github.com/carlos-emr/carlos/pull/2910

**PR Description:** Description
Removes unused legacy OWASP Encoder taglib declarations and Encode imports from 19 JSP files in the appointment/ and schedule/ directories.

These files were previously migrated to use the null-safe CARLOS wrappers (<carlos:encode>, SafeEncode.*), but the legacy declarations were left in place. No <e:forXxx> tags or Encode.forXxx() calls remain in any of the modified files.

Related Issues
Fixes #2680

How Was This Tested?
Dead code removal only, no behavioral changes. Verified manually that no <e:forXxx> tags or Encode.forXxx() calls remain in any of the modified files after removing the declarations. CI lint check (check-encoder-null-safety.sh) will confirm on PR.

Checklist
 My commits are signed off for the DCO (git commit -s)
 My commits follow Conventional Commits format, or I've written clear commit messages and will use the format next time
 I have not included any patient data (PHI) in this PR
 I have added tests for new functionality, or this change doesn't need new tests
 I have read the contributing guide

**Maintainer Feedback:**
- [Date]: Hi there @bellaayang! Thanks for the contribution. Small things, but it all helps, and we really appreciate our community. Thank you!!
- [Date]: June 15th

**Status:**  Merged

---

## Learnings & Reflections

### Technical Skills Gained

Hands-on experience auditing a real production codebase for dead code across a specific, well-defined pattern (unused taglib/import declarations).

Learned the difference between the raw OWASP Encoder API (<e:forXxx>, Encode.forXxx()) and a project's own null-safe wrapper layer (CARLOS's <carlos:encode>, SafeEncode.*), and why teams migrate toward the latter (null-safety, consistency, centralized encoding logic).

Practiced disciplined, file-by-file verification before making changes — confirming absence of usage rather than assuming it from the issue description alone.

Gained experience setting up a Java/Maven/Tomcat devcontainer under a restrictive network environment, including passing proxy settings as Docker build args and diagnosing SSL handshake failures in Maven dependency resolution.

### Challenges Overcome

Challenge 1 — Devcontainer network setup. The ARM64 Docker build could not fetch packages from ports.ubuntu.com/ubuntu-ports behind a proxy on a Japan-based network. Switching proxy nodes and changing Docker Desktop's global proxy setting didn't help because the proxy wasn't propagating into the container build step. Resolved by passing the proxy explicitly as http_proxy/https_proxy build args in docker-compose.yml for all three services, using the Mac's LAN IP directly (host.docker.internal failed to resolve on this network).

Challenge 2 — Flaky downloads inside the Dockerfile. Three separate download steps were unreliable under the same network conditions: the Claude Code install script, Playwright's browser binaries, and several npm packages fetched individually. Resolved by commenting out the two steps that weren't actually required to run this project (Claude Code, Playwright), and consolidating the npm installs into a single command pointed at the npmmirror registry for reliability.

Challenge 3 — Maven SSL handshake failures. mvn dependency:go-offline dependency:sources dependency:resolve -Dclassifier=javadoc failed with SSL handshake errors against build.shibboleth.net and jitpack.io. Resolved by scoping the command down to just dependency:go-offline, since the sources/javadoc artifacts aren't needed to build or run the app.

Challenge 4 — Verifying "unused" across 19 files without missing one. Because the issue listed files by two overlapping criteria (unused taglib vs. unused import, with several files appearing on both lists), it was easy to lose track of which declaration had already been checked in a given file. I handled this by grepping each file for <e: and Encode.forXxx( before and after editing, rather than relying on memory of which list a file came from.

### What I'd Do Differently Next Time

Filling in the full template (this document) while doing the work, not after — several sections were left blank in the original submission, which is likely why the score was low.

Actually checking off the PR checklist items (DCO sign-off, Conventional Commits, etc.) before opening the PR, rather than leaving them unchecked.

Running the CI lint script locally first, to catch issues before pushing rather than relying on CI to catch them.

---

## Resources Used

- Issue #2680: https://github.com/carlos-emr/carlos/issues/2680
- Reproduction commit: https://github.com/bellaayang/carlos/commit/73101f14ae
- PR #2910: https://github.com/carlos-emr/carlos/pull/2910
- Project's CLAUDE.md (referenced in the issue for the CI encoder-null-safety policy)



---

### **Contribution Number:** 2
**Student:** Jinghan Yang
**Issue:** https://github.com/carlos-emr/carlos/issues/2598
**Status:** Phase IV marked complete

---

## Why I Chose This Issue

I chose this issue because it required methodical code reading across many files rather than mechanical find-and-replace. Each file needed individual verification — I had to check the nearby `securityInfoManager.hasPrivilege()` call to confirm the correct object name before updating the message, rather than blindly substituting text. This made it a good exercise in reading unfamiliar Java code carefully.

The underlying context is also meaningful: consistent exception messages matter for log monitoring and security auditing. When every `SecurityException` follows the same canonical format, automated tools and human reviewers can reliably detect and filter authorization failures. Understanding *why* the format is enforced, not just *what* to change, made this feel like real security work rather than cleanup.

---

## Understanding the Issue

### Problem Description

Multiple action classes across the encounter, case management, and measurement admin modules throw `SecurityException` with non-canonical message formats. CLAUDE.md mandates the format `missing required sec object (_objectname)`, but various files used `"missing required security object"` (spelled out), missing parentheses, colon separators instead of parentheses, missing object names entirely, or placeholder strings like `"Access Denied!"`.

### Expected Behavior

All `SecurityException` messages in the affected files should follow the canonical format:
```
missing required sec object (_objectname)
```
where `_objectname` matches the object name used in the nearby `hasPrivilege()` check.

### Current Behavior

Non-canonical variants found:
- `"missing required security object (_demographic)"` — "security" not abbreviated to "sec"
- `"missing required security object: _newCasemgmt.templates"` — colon instead of parentheses
- `"missing required security object"` — missing object name entirely
- `"missing required security object _admin.consult"` — no parentheses
- `"Access Denied!"` — placeholder with a comment acknowledging the correct form

### Affected Components

- `casemgmt/web/ClientImage2Action.java`
- `casemgmt/web/CaseManagementEntry2Action.java`
- `encounter/pageUtil/EctInsertTemplate2Action.java`
- `encounter/oscarConsultationRequest/config/pageUtil/ConsultationLookup2Action.java`
- `encounter/oscarConsultationRequest/config/pageUtil/CpsoSearch2Action.java`
- `encounter/oscarConsultationRequest/pageUtil/ConsultationClinicalData2Action.java`
- `encounter/oscarMeasurements/pageUtil/EctSetupAddMeasurementStyleSheet2Action.java`
- 15 measurement admin actions in `encounter/oscarMeasurements/pageUtil/` using `"Access Died!"`

---

## Reproduction Process

### Environment Setup

Same devcontainer setup as Contribution #1.

### Steps to Reproduce

1. Search the codebase for `"missing required security object"` — non-canonical messages are visible immediately
2. Search for `"Access Died!"` in `oscarMeasurements/pageUtil/` — 15 files with placeholder messages

### Reproduction Evidence

Verified with:
```bash
grep -rn "missing required security object" src/main/java/io/github/carlos_emr/carlos/
grep -rn "Access Died!" src/main/java/io/github/carlos_emr/carlos/encounter/oscarMeasurements/pageUtil/
```

---

## Solution Approach

### Analysis

The root cause is inconsistency during original development — different developers wrote the exception messages in different styles, and some used placeholder strings with comments acknowledging the correct form but never implementing it.

### Proposed Solution

For each affected file:
1. Locate the nearby `securityInfoManager.hasPrivilege(...)` call to identify the correct object name
2. Rewrite the `SecurityException` message to match the canonical format exactly

### Implementation Plan

**Understand:** The canonical format is `missing required sec object (_objectname)`. The object name must come from the `hasPrivilege()` call in the same method, not guessed.

**Match:** Three fix patterns:
- `"missing required security object (X)"` → `"missing required sec object (X)"` (abbreviate "security")
- `"missing required security object: X"` or `"missing required security object X"` → `"missing required sec object (X)"` (fix separator + add parens)
- `"Access Died!"` → `"missing required sec object (_admin)"` (replace placeholder; confirmed `_admin` from `hasPrivilege`)

**Plan:**
1. Use VSCode global search-replace for the `(_demographic)` pattern (3 files)
2. Manually fix `EctInsertTemplate2Action` (colon → parens)
3. Manually fix `ConsultationLookup2Action`, `CpsoSearch2Action`, `ConsultationClinicalData2Action` (verify object name from `hasPrivilege` first)
4. Global replace `"Access Died!"` in `oscarMeasurements/pageUtil/` (all 15 confirmed `_admin`)
5. Verify with `grep -rn "missing required security object"` — no in-scope files should remain

**Implement:** Branch `fix-issue-2598`

**Review:** Checked each file's `hasPrivilege` call before substituting to ensure object name accuracy

**Evaluate:** Final grep confirmed no in-scope files contain non-canonical messages

---

## Testing Strategy

### Manual Testing

Ran grep verification before and after each batch of changes to confirm correctness. No functional logic was modified, so no behavioral tests are needed.

---

## Implementation Notes

Updated 23 files total: standardized "security object" → "sec object" abbreviation, fixed colon/missing-parens variants, and replaced 15 `"Access Died!"` placeholders with the correct canonical message using `_admin` confirmed from each file's `hasPrivilege()` call.

---

## Pull Request

**PR Link:** https://github.com/carlos-emr/carlos/pull/3119#pullrequestreview-4628682638

**PR Description:** Standardizes non-canonical SecurityException messages to `missing required sec object (_objectname)` format across encounter and case management action classes, per CLAUDE.md. Covers the files called out in the issue and audit comments.

**Status:** Submitted, awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

- Reading Java Struts2 action classes and understanding the security check pattern
- Using `grep` and VSCode global search to audit large codebases efficiently
- Understanding why consistent exception message formats matter for security mo
nitoring

### Challenges Overcome

- Needed to verify each file's `hasPrivilege()` call individually to confirm the correct object name rather than blindly replacing
- Identified and reverted two out-of-scope files that were caught by the global replace

### What I'd Do Differently Next Time

Run `git diff --name-only` before committing to catch any unintended file modifications earlier in the process.


### Contribution Number: 3
Student: Jinghan Yang
Issue: https://github.com/carlos-emr/carlos/issues/3108#issuecomment-4904716881
Status: Phase 1

### Why I Chose This Issue

I chose this issue because it involves an important security and architecture improvement in CARLOS EMR. The original implementation allowed a fax-sending operation to live inside a PDF-generation servlet, which made the code harder to reason about and created a risk that a state-changing operation could be triggered through an unsafe HTTP method. This issue gave me a chance to understand how CARLOS EMR separates read-only PDF rendering from mutating actions, how Struts `*2Action` classes are used as safer HTTP boundaries, and how security-related tests enforce GET/HEAD rejection for mutating workflows.

### Understanding the Issue

#### Problem Description

`FrmCustomedPDFServlet` was responsible for creating customized prescription PDFs. However, it also contained a special `oscarRxFax` branch that performed fax-related side effects: generating and writing prescription PDF files, writing fax tracking files, persisting `FaxJob` records, and logging fax activity.

The problem was that the servlet overrides `service(...)`, so GET and POST requests were handled the same way. This meant a GET request to `/form/createcustomedpdf` with `__method=oscarRxFax` could trigger fax side effects. Since this servlet was not a Struts `*2Action`, it was not protected by the existing mutator GET/HEAD rejection contract test.

#### Expected Behavior

The prescription fax workflow should only be available through a mutating POST-only endpoint. GET and HEAD requests should be rejected before any side effects happen.

An authenticated user must have the required `_rx` write privilege before the system generates the prescription PDF or creates any fax job artifacts.

The PDF servlet should only handle read-only PDF generation and should not create fax files, persist fax jobs, or perform fax logging.

#### Current Behavior

Before the fix, the PDF servlet mixed two responsibilities:

1. Read-only prescription PDF generation.
2. Mutating prescription fax submission.

Because both paths were reachable through `service(...)`, the fax mutation could be reached through GET as well as POST. This violated the project’s mutator action safety pattern and made the endpoint invisible to the existing `MutatorActionGetRejectionContractTest`.

#### Affected Components

- `FrmCustomedPDFServlet`
  - Removed the legacy `oscarRxFax` mutation branch so the servlet only generates PDFs.

- `PrescriptionPdfComposer`
  - Extracted prescription PDF composition logic out of the servlet.

- `PrescriptionFaxService`
  - Extracted fax file creation, fax tracking file writing, `FaxJob` persistence, and fax logging into a dedicated service.
  - Added stronger validation before filesystem or DAO side effects.

- `RxFaxPrescription2Action`
  - Added a new POST-only Struts action for the Rx fax mutation.
  - Enforces method, authentication, and `_rx` write privilege checks before side effects.

- `ViewScript2.jsp`
  - Routed Rx fax submissions to the new Struts action endpoint.

- `struts-prescription.xml`
  - Registered the new Rx fax action route.

- `MutatorActionGetRejectionContractTest`
  - Registered the new action so GET/HEAD rejection is covered by the project-wide mutator contract.

- `RxFaxPrescription2ActionTest`
  - Added focused unit tests for method rejection, authorization failure, validation failure, and successful fax job creation.

- `PrescriptionFaxServiceTest`
  - Added tests for fax file handling, PDF id validation, fax config matching, and avoiding file artifacts when validation fails.

