# Root Cause Analysis: Issue #1295 - Duplicated Welcome and Folder Trust Prompts

## Issue Summary
**Issue**: [github/copilot-cli#1295](https://github.com/github/copilot-cli/issues/1295)  
**Title**: Duplicated welcome and folder trust prompts shown on every run of Copilot CLI  
**Affected Version**: 0.0.402  
**Severity**: Medium (UX issue, not blocking functionality)

## Problem Description
On every launch of the `copilot` command, two UI prompts appear twice in sequence:
1. Welcome prompt: "Describe a task to get started"
2. Folder trust confirmation UI

This duplication occurs:
- On first run in a new directory
- On subsequent runs in the same directory
- Consistently across different terminals (Terminal.app, iTerm2)
- On macOS platform

## Investigation

### Analysis of Changelog
Review of the changelog.md file reveals a history of similar duplication issues that have been fixed in previous versions:

1. **v0.0.375**: "Responses with reasoning no longer cause duplicate assistant messages"
2. **v0.0.372**: "Long commands no longer show duplicate intention headers when wrapping"
3. **v0.0.345**: "Prevent double line wraps in markdown messages"

This pattern suggests the codebase has had recurring issues with UI element duplication, possibly due to:
- Event listener double-registration
- UI component double-rendering
- State management issues

### Related Changelog Entry
**v0.0.285** mentions: "Hid the 'Welcome to GitHub Copilot CLI' welcome message on session resumption and `/clear` for a cleaner look"

This indicates that the welcome message display logic has been modified before, and the current issue might be a regression introduced in subsequent changes.

### Version 0.0.402 Changes
Version 0.0.402 (released 2026-02-03) included several significant changes:
- ACP server supports agent and plan session modes
- MCP configuration applies to ACP mode
- Agent creation wizard styling improvements
- Custom agents with unknown fields load with warnings instead of errors
- Custom agents receive environment context when run as subagents
- **Plugins can provide hooks for session lifecycle events** ⚠️
- Plugin update command works for direct plugins and handles Windows file locks
- Stop MCP servers when uninstalling plugins

## Hypothesized Root Causes

### Most Likely: Session Lifecycle Hook Duplication
The addition of "Plugins can provide hooks for session lifecycle events" in v0.0.402 suggests that the initialization/welcome prompts might now be triggered through both:
1. The original initialization path
2. A new plugin lifecycle hook

**Why this is likely:**
- Timing aligns with v0.0.402 release
- Session lifecycle hooks are a new feature
- Welcome/trust prompts are typical session initialization events

### Alternative Hypothesis 1: ACP Mode Integration
The ACP (Agent Client Protocol) server changes might have introduced a code path that duplicates the initialization sequence when running in standard CLI mode.

**Why this is possible:**
- Significant architectural changes in v0.0.402
- ACP mode and standard mode might share initialization code
- Could be triggering initialization twice when in non-ACP mode

### Alternative Hypothesis 2: MCP Configuration Apply
"MCP configuration applies to ACP mode" suggests configuration loading changes that might now happen twice in the initialization sequence.

### Alternative Hypothesis 3: Plugin Loading Regression
Changes to plugin loading ("Plugin update command works for direct plugins and handles Windows file locks") might have modified the initialization sequence to load/initialize plugins twice.

## Common Patterns for This Type of Bug

Based on typical CLI application architecture, duplication issues like this often occur due to:

1. **Event Listener Double-Registration**
   ```javascript
   // Problematic pattern
   function initializeApp() {
     // This might be called twice
     eventEmitter.on('session-start', showWelcomePrompt);
     eventEmitter.on('session-start', checkFolderTrust);
   }
   ```

2. **Component Re-rendering**
   ```javascript
   // UI components rendered twice
   function renderApp() {
     renderWelcomeScreen();  // Called in two different code paths
   }
   ```

3. **State Management Issue**
   ```javascript
   // Session state not properly checked
   if (!hasShownWelcome) {  // This flag might not be set/checked correctly
     showWelcome();
   }
   ```

4. **Async Initialization Race**
   ```javascript
   // Multiple async initializations completing
   Promise.all([initPlugins(), initSession()])
     .then(() => {
       // Both might trigger welcome prompts
       showWelcomePrompt();
     });
   ```

## Recommended Investigation Steps

For developers with source code access:

1. **Check Session Lifecycle Hook Implementation**
   - Look for where `session lifecycle events` hooks were added in v0.0.402
   - Verify that welcome/trust prompts aren't triggered by both old and new code paths
   - Search for any event listener registration that might happen twice

2. **Review ACP Mode Integration**
   - Check if ACP mode initialization shares code with standard CLI mode
   - Verify conditional logic for mode-specific initialization
   - Look for any initialization code that runs regardless of mode

3. **Audit Welcome Prompt Code Path**
   - Search codebase for "Describe a task to get started" string
   - Trace all code paths that can trigger this prompt
   - Check for duplicate calls in the initialization sequence

4. **Check Folder Trust Prompt Logic**
   - Look for folder trust verification code
   - Verify it's not called twice in the initialization
   - Check if plugin loading triggers additional trust checks

5. **Add Logging/Debugging**
   - Add debug logging around welcome and trust prompt triggering
   - Track initialization sequence to identify duplicate calls
   - Monitor event listener registration

## Proposed Fix Approach

Without source code access, the recommended approach would be:

1. **Add Guard Flags**
   ```javascript
   let welcomeShown = false;
   let trustPromptShown = false;
   
   function showWelcomePrompt() {
     if (welcomeShown) return;
     welcomeShown = true;
     // Show prompt...
   }
   ```

2. **Deduplicate Event Listeners**
   ```javascript
   // Remove before adding
   eventEmitter.off('session-start', showWelcomePrompt);
   eventEmitter.on('session-start', showWelcomePrompt);
   ```

3. **Consolidate Initialization**
   - Ensure there's a single initialization entry point
   - Remove duplicate initialization in plugin/ACP code paths
   - Use a centralized session manager

## Testing Recommendations

To verify the fix:

1. Fresh directory test:
   ```bash
   mkdir /tmp/copilot-test-$(date +%s)
   cd /tmp/copilot-test-*
   copilot
   # Should see welcome prompt only ONCE
   ```

2. Repeat run test:
   ```bash
   # In same directory
   copilot
   # Should see prompts only ONCE (if applicable)
   ```

3. Different terminal test:
   - Test in Terminal.app
   - Test in iTerm2
   - Test in VS Code integrated terminal

4. Clean config test:
   ```bash
   mv ~/.copilot ~/.copilot.backup
   copilot
   # Should see prompts only ONCE
   ```

## Conclusion

The most likely root cause is the new "session lifecycle events" hook system introduced in v0.0.402, which may be triggering welcome and folder trust prompts in addition to the existing initialization code path. 

The fix likely requires:
1. Identifying where the duplication occurs in the initialization sequence
2. Adding guard flags or deduplication logic
3. Ensuring lifecycle hooks don't duplicate existing functionality

This is consistent with the pattern of previous duplication bugs in the codebase and the timing of the issue's appearance with the v0.0.402 release.
