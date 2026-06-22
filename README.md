# Contribution [#]: [Issue Title]

**Contribution Number:** [1 / 2 / 3]  
**Student:** [Your Name]  
**Issue:** https://github.com/carlos-emr/carlos/issues/2680 
**Status:** Phase 2 Complete

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

**Status:** [Awaiting review / Iterating / Approved / Merged]

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
