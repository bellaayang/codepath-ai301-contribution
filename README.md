# Contribution [#]: [Issue Title]

**Contribution Number:** 1  
**Student:** Jinghan Yang 
**Issue:** https://github.com/carlos-emr/carlos/issues/2680 
**Status:** Phase IV marked complete

---

## Why I Chose This Issue

I chose this issue because it offers a clear, well-scoped entry point into a real production codebase. The task — removing unused legacy OWASP Encoder taglib declarations and imports across JSP files — is straightforward enough for a first contribution, but still requires carefully reading each file to confirm no <e:forXxx> or Encode.forXxx() calls remain before removing the declarations. That kind of methodical, file-by-file verification is good practice for working in an unfamiliar codebase without breaking things.

I also find the underlying context interesting. The project has already migrated to null-safe CARLOS wrappers, and this issue is about cleaning up the leftover artifacts from that migration. Understanding why the old encoder was replaced — and how the new SafeEncode wrappers differ — connects directly to what I'm learning about secure coding practices. It's a low-risk issue with a clear success condition (CI passes), which makes it a good fit for building confidence contributing to open source.

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

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

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Multiple JSP files in the appointment/ and schedule/ directories still declare the legacy OWASP Encoder taglib (<%@ taglib uri="owasp.encoder.jakarta.advanced" prefix="e" %>) and/or import org.owasp.encoder.Encode directly, even though these files have already been migrated to use the null-safe CARLOS wrappers (<carlos:encode>, SafeEncode.*). These declarations are unused leftovers that create confusion and may trip future CI linting passes.

**Match:** The fix pattern is straightforward: locate each affected file, confirm no <e:forXxx> tags or Encode.forXxx() calls remain, then remove the unused declaration. The project's CI script scripts/lint/check-encoder-null-safety.sh enforces that new code does not use the legacy encoder, so removing these declarations aligns with the existing migration pattern already applied across the rest of the codebase.

**Plan:** 
1. For each of the 18 files with unused <e:> taglib: remove the <%@ taglib uri="owasp.encoder.jakarta.advanced" prefix="e" %> line
2. For each of the 9 files with unused Encode import: remove the <%@ page import="org.owasp.encoder.Encode" %> line
3. Verify no <e:forXxx> or Encode.forXxx() calls remain in any modified file
4. No test changes needed — this is a dead code removal with no behavioral impact

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

Removed unused legacy OWASP Encoder taglib declarations (<%@ taglib uri="owasp.encoder.jakarta.advanced" prefix="e" %>) and unused Encode imports (<%@ page import="org.owasp.encoder.Encode" %>) from 19 JSP files in the appointment/ and schedule/ directories. These files had already been migrated to use the null-safe CARLOS wrappers (<carlos:encode>, SafeEncode.*), but the legacy declarations were left in place as dead code.


### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** https://github.com/carlos-emr/carlos/pull/2910

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:**  Merged

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]


---

### **Contribution Number:** 2
**Student:** Jinghan Yang
**Issue:** https://github.com/carlos-emr/carlos/issues/2598
**Status:** PR submitted, awaiting review

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

