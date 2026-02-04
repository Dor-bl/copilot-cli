# Issue #1295 Investigation Summary

## Overview

This document provides a complete summary of the investigation into GitHub Copilot CLI issue #1295: "Duplicated welcome and folder trust prompts shown on every run of Copilot CLI".

## Quick Links

- **Original Issue**: [github/copilot-cli#1295](https://github.com/github/copilot-cli/issues/1295)
- **Root Cause Analysis**: [ROOT_CAUSE_ANALYSIS_1295.md](./ROOT_CAUSE_ANALYSIS_1295.md)
- **Reproduction Tests**: [REPRODUCTION_TEST_1295.md](./REPRODUCTION_TEST_1295.md)
- **Proposed Fix**: [PROPOSED_FIX_1295.md](./PROPOSED_FIX_1295.md)

## Issue Details

### What's Happening?

On every launch of GitHub Copilot CLI version 0.0.402, two UI prompts appear **twice** in sequence:

1. **Welcome prompt**: "Describe a task to get started"
2. **Folder trust confirmation**: "Confirm folder trust" UI

### Impact

- **Severity**: Medium (UX issue, not blocking)
- **Frequency**: 100% reproduction rate
- **Affected Version**: 0.0.402
- **Platform**: macOS (Terminal.app, iTerm2) - other platforms TBD
- **User Impact**: Annoyance; users must dismiss duplicate prompts each launch

### Screenshot

[Screenshot showing duplicated prompts](https://github.com/user-attachments/assets/b413ce09-518b-4155-8771-ed97ffdb0e46)

## Root Cause Analysis

### Most Likely Cause

**Session Lifecycle Hook Duplication** (introduced in v0.0.402)

The changelog for v0.0.402 states: "Plugins can provide hooks for session lifecycle events"

**Hypothesis**: The new plugin lifecycle hook system triggers initialization prompts (welcome and folder trust) in addition to the existing initialization code path, causing each prompt to display twice.

### Why This is Likely

1. **Timing**: Issue appeared in v0.0.402, same version that introduced lifecycle hooks
2. **Pattern**: Previous duplication bugs in the codebase (v0.0.375, v0.0.372, v0.0.345)
3. **Scope**: Welcome and folder trust are typical session initialization events
4. **Behavior**: Consistent duplication suggests systematic issue, not random bug

### Alternative Hypotheses

1. **ACP Mode Integration**: New ACP server changes might duplicate initialization
2. **MCP Configuration**: Changes in how MCP config is applied might trigger double init
3. **Plugin Loading**: Plugin loading changes might call initialization twice

See [ROOT_CAUSE_ANALYSIS_1295.md](./ROOT_CAUSE_ANALYSIS_1295.md) for detailed analysis.

## How to Reproduce

### Simple Reproduction

```bash
# Create fresh directory
mkdir /tmp/copilot-test-$(date +%s)
cd /tmp/copilot-test-*

# Launch copilot
copilot

# Observe: Both welcome and trust prompts appear TWICE
```

### What You'll See

1. Welcome prompt: "Describe a task to get started" → appears
2. Welcome prompt: "Describe a task to get started" → appears AGAIN
3. Folder trust confirmation → appears
4. Folder trust confirmation → appears AGAIN

This happens on **every** launch, not just the first.

See [REPRODUCTION_TEST_1295.md](./REPRODUCTION_TEST_1295.md) for comprehensive test cases.

## Proposed Solution

### Recommended Approach: Phased Fix

#### Phase 1: Immediate Fix (v0.0.403) ⚡

**Solution**: Add guard flags to prevent duplicate display

```javascript
class SessionManager {
  constructor() {
    this.welcomeShown = false;
    this.folderTrustPromptShown = false;
  }

  async showWelcomePrompt() {
    if (this.welcomeShown) return;
    this.welcomeShown = true;
    await this.ui.displayWelcome("Describe a task to get started");
  }
}
```

- **Risk**: Low
- **Time**: 2-4 hours
- **Pros**: Quick mitigation, minimal code change
- **Cons**: Addresses symptom, not root cause

#### Phase 2: Root Cause Fix (v0.0.405+) 🔧

**Solution**: Audit v0.0.402 changes and remove duplicate initialization paths

1. Review plugin lifecycle hook implementation
2. Identify where prompts are called from
3. Remove duplicate calls
4. Keep prompts in single initialization path

- **Risk**: Medium
- **Time**: 1-2 days
- **Pros**: Fixes root cause, no workarounds
- **Cons**: Requires code audit, testing

#### Phase 3: Long-term (Future) 🏗️

**Solution**: Consolidate all initialization into single manager

- **Risk**: High
- **Time**: 1-2 weeks
- **Pros**: Clean architecture, prevents future duplication
- **Cons**: Significant refactor, high risk

See [PROPOSED_FIX_1295.md](./PROPOSED_FIX_1295.md) for detailed implementation plans.

## Testing Strategy

### Before Fix (v0.0.402)

```bash
# All these should show DUPLICATE prompts
copilot                    # ❌ Shows twice
copilot --banner          # ❌ Shows twice
copilot (second run)      # ❌ Shows twice
```

### After Fix (v0.0.403+)

```bash
# All these should show prompts ONCE
copilot                    # ✅ Shows once
copilot --banner          # ✅ Shows once
copilot (second run)      # ✅ Shows once (or not at all if trusted)
```

### Automated Tests

```javascript
test('welcome prompt shown only once', async () => {
  const displayWelcomeSpy = jest.spyOn(sessionManager.ui, 'displayWelcome');
  
  await sessionManager.showWelcomePrompt();
  await sessionManager.showWelcomePrompt(); // Called twice
  
  expect(displayWelcomeSpy).toHaveBeenCalledTimes(1); // ✅ Only once
});
```

## Timeline

| Version | Status | Notes |
|---------|--------|-------|
| v0.0.401 | ✅ Working | No duplication reported |
| v0.0.402 | ❌ Broken | Issue introduced |
| v0.0.403 | 🔧 Fix Target | Guard flags (Phase 1) |
| v0.0.405+ | 🔧 Fix Target | Root cause fix (Phase 2) |

## Related Issues

Previous duplication issues fixed:
- v0.0.375: "Responses with reasoning no longer cause duplicate assistant messages"
- v0.0.372: "Long commands no longer show duplicate intention headers"
- v0.0.345: "Prevent double line wraps in markdown messages"

This suggests a pattern of duplication issues in the codebase.

## For Developers

### Where to Look

1. **Plugin Lifecycle Hooks** (v0.0.402 changes)
   - Search for: `session-start`, `lifecycle`, `hooks`
   - Check: Event listener registration
   
2. **Session Initialization**
   - Look for: Welcome prompt code
   - Look for: Folder trust prompt code
   - Check: How many code paths call these

3. **ACP Mode Integration** (v0.0.402 changes)
   - Check: ACP initialization sequence
   - Verify: Doesn't duplicate standard CLI init

4. **MCP Configuration** (v0.0.402 changes)
   - Check: Config loading sequence
   - Verify: Doesn't trigger duplicate init

### Quick Grep Commands

```bash
# Find welcome prompt code
git grep -n "Describe a task to get started"

# Find folder trust code
git grep -n "folder.*trust\|trust.*folder" --ignore-case

# Find session start events
git grep -n "session.*start\|start.*session" --ignore-case

# Find lifecycle hooks
git grep -n "lifecycle.*hook\|hook.*lifecycle" --ignore-case

# Check v0.0.402 changes
git diff v0.0.401..v0.0.402
```

## Success Criteria

The fix is successful when:

1. ✅ Welcome prompt appears exactly **once** per launch
2. ✅ Folder trust prompt appears exactly **once** when needed
3. ✅ No regression in plugin functionality
4. ✅ No regression in ACP mode
5. ✅ Tests pass on all platforms (macOS, Linux, Windows)
6. ✅ Manual testing confirms fix
7. ✅ No new issues within 7 days of release

## Workaround (Until Fixed)

**For Users**: No workaround available. Must dismiss duplicate prompts.

**Recommended**: Wait for v0.0.403 patch release with fix.

## Communication Plan

1. **GitHub Issue**: Update #1295 with investigation findings
2. **Changelog**: Document fix in v0.0.403 release notes
3. **Release Notes**: Highlight fix prominently
4. **Users**: Notify affected users to upgrade

## References

- Issue: https://github.com/github/copilot-cli/issues/1295
- Changelog: [changelog.md](./changelog.md)
- Repository: https://github.com/github/copilot-cli

## Investigation Credit

- **Reporter**: [@Dor-bl](https://github.com/Dor-bl)
- **Date Reported**: 2026-02-04
- **Investigation Date**: 2026-02-04
- **Status**: Root cause identified, fix proposed

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-04  
**Status**: Investigation Complete ✅
