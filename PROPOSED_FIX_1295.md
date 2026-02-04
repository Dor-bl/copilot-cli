# Proposed Fix for Issue #1295

## Issue Summary
Duplicated welcome and folder trust prompts shown on every run of Copilot CLI version 0.0.402.

## Root Cause (Hypothesis)
The new "session lifecycle events" hook system introduced in v0.0.402 is triggering welcome and folder trust prompts in addition to the existing initialization code path, causing each prompt to appear twice.

## Proposed Solutions

### Solution 1: Add Guard Flags (Recommended - Minimal Change)

**Approach**: Prevent duplicate prompt display by tracking whether prompts have been shown in the current session.

**Implementation**:

```javascript
// In session initialization module

class SessionManager {
  constructor() {
    this.welcomeShown = false;
    this.folderTrustPromptShown = false;
  }

  async showWelcomePrompt() {
    // Guard against duplicate calls
    if (this.welcomeShown) {
      console.debug('Welcome prompt already shown in this session, skipping');
      return;
    }
    
    this.welcomeShown = true;
    await this.ui.displayWelcome("Describe a task to get started");
  }

  async showFolderTrustPrompt(folder) {
    // Guard against duplicate calls
    if (this.folderTrustPromptShown) {
      console.debug('Folder trust prompt already shown in this session, skipping');
      return;
    }
    
    this.folderTrustPromptShown = true;
    await this.ui.displayFolderTrustPrompt(folder);
  }

  reset() {
    // Reset for new session
    this.welcomeShown = false;
    this.folderTrustPromptShown = false;
  }
}
```

**Pros**:
- Minimal code change
- Low risk
- Quick to implement
- Addresses symptom immediately

**Cons**:
- Doesn't fix the underlying duplicate call issue
- Requires guard flags to be maintained

**Risk Level**: Low

---

### Solution 2: Deduplicate Event Listeners

**Approach**: Ensure event listeners for session initialization are only registered once.

**Implementation**:

```javascript
// In plugin/lifecycle hooks module

class LifecycleHooks {
  constructor() {
    this.registeredHooks = new Set();
  }

  registerSessionStartHook(hookName, callback) {
    const hookId = `session-start:${hookName}`;
    
    // Prevent duplicate registration
    if (this.registeredHooks.has(hookId)) {
      console.debug(`Hook ${hookId} already registered, skipping`);
      return;
    }
    
    this.registeredHooks.add(hookId);
    
    // Remove any existing listener before adding
    this.eventEmitter.off('session-start', callback);
    this.eventEmitter.on('session-start', callback);
  }

  unregisterSessionStartHook(hookName) {
    const hookId = `session-start:${hookName}`;
    this.registeredHooks.delete(hookId);
  }
}
```

**Alternative using once()**:
```javascript
// If the hook should only run once per session
this.eventEmitter.once('session-start', callback);
```

**Pros**:
- Addresses root cause if issue is event listener duplication
- Prevents future similar issues
- Clean event handling

**Cons**:
- Requires understanding event flow
- May affect plugin behavior if they rely on multiple registrations

**Risk Level**: Medium

---

### Solution 3: Consolidate Initialization Paths

**Approach**: Ensure there's only one code path that triggers initialization prompts.

**Implementation**:

```javascript
// In main initialization module

class AppInitializer {
  constructor() {
    this.initialized = false;
  }

  async initialize() {
    if (this.initialized) {
      console.debug('Application already initialized');
      return;
    }

    console.debug('Starting application initialization');
    
    try {
      // Single initialization sequence
      await this.initializeSession();
      await this.loadPlugins();
      await this.setupACPMode();
      
      // Prompts called only once, in controlled sequence
      await this.sessionManager.showWelcomePrompt();
      await this.sessionManager.showFolderTrustPrompt();
      
      this.initialized = true;
      console.debug('Application initialization complete');
    } catch (error) {
      console.error('Initialization failed:', error);
      throw error;
    }
  }
}
```

**Key Changes**:
- Remove prompt calls from:
  - Plugin lifecycle hooks
  - ACP mode initialization
  - MCP configuration loading
- Keep only in main initialization sequence

**Pros**:
- Cleanest solution
- Single source of truth for initialization
- Prevents any duplication

**Cons**:
- Requires significant refactoring
- May break plugin expectations
- Higher risk of regression

**Risk Level**: High

---

### Solution 4: Audit and Fix v0.0.402 Changes

**Approach**: Review specific changes in v0.0.402 and remove duplicate prompt triggers.

**Investigation Steps**:

1. **Check plugin lifecycle hook implementation**:
   ```bash
   # Search for session-start or initialization events
   git diff v0.0.401..v0.0.402 -- "*.ts" "*.js" | grep -A5 -B5 "session.*start\|lifecycle\|welcome\|trust"
   ```

2. **Check ACP mode changes**:
   ```bash
   git diff v0.0.401..v0.0.402 -- "*acp*" | grep -A5 -B5 "initialize\|welcome"
   ```

3. **Check MCP configuration changes**:
   ```bash
   git diff v0.0.401..v0.0.402 -- "*mcp*" | grep -A5 -B5 "initialize\|welcome"
   ```

**Fix**:
- Identify the new code path calling prompts
- Remove duplicate calls
- Ensure prompts only called from original path

**Pros**:
- Targeted fix
- Removes root cause
- No workarounds needed

**Cons**:
- Requires git history access
- Need to understand code changes

**Risk Level**: Medium

---

## Recommended Implementation Plan

### Phase 1: Immediate Fix (Ship with v0.0.403)

Use **Solution 1 (Guard Flags)** for quick mitigation:

1. Add `welcomeShown` and `folderTrustPromptShown` flags
2. Check flags before showing prompts
3. Reset flags on session end or `/clear` command
4. Ship in next patch release

**Estimated Time**: 2-4 hours
**Risk**: Low

### Phase 2: Root Cause Fix (Ship with v0.0.405+)

Implement **Solution 4 (Audit v0.0.402 Changes)**:

1. Review plugin lifecycle hook implementation
2. Review ACP mode initialization
3. Review MCP configuration loading
4. Identify duplicate initialization paths
5. Remove duplicate prompt calls
6. Add integration tests
7. Thoroughly test plugin compatibility

**Estimated Time**: 1-2 days
**Risk**: Medium

### Phase 3: Long-term Improvement (Future)

Consider **Solution 3 (Consolidate Initialization)** as part of larger refactor:

1. Design centralized initialization system
2. Migrate all initialization code to single manager
3. Update plugin API if needed
4. Comprehensive testing

**Estimated Time**: 1-2 weeks
**Risk**: High

---

## Testing Strategy

### Unit Tests

```javascript
describe('SessionManager', () => {
  let sessionManager;

  beforeEach(() => {
    sessionManager = new SessionManager();
  });

  test('welcome prompt shown only once', async () => {
    const displayWelcomeSpy = jest.spyOn(sessionManager.ui, 'displayWelcome');
    
    await sessionManager.showWelcomePrompt();
    await sessionManager.showWelcomePrompt(); // Called twice
    
    expect(displayWelcomeSpy).toHaveBeenCalledTimes(1); // Should only show once
  });

  test('folder trust prompt shown only once', async () => {
    const displayTrustSpy = jest.spyOn(sessionManager.ui, 'displayFolderTrustPrompt');
    
    await sessionManager.showFolderTrustPrompt('/test/path');
    await sessionManager.showFolderTrustPrompt('/test/path'); // Called twice
    
    expect(displayTrustSpy).toHaveBeenCalledTimes(1); // Should only show once
  });

  test('prompts shown after reset', async () => {
    const displayWelcomeSpy = jest.spyOn(sessionManager.ui, 'displayWelcome');
    
    await sessionManager.showWelcomePrompt();
    sessionManager.reset();
    await sessionManager.showWelcomePrompt();
    
    expect(displayWelcomeSpy).toHaveBeenCalledTimes(2); // Should show twice after reset
  });
});
```

### Integration Tests

```javascript
describe('Application Initialization', () => {
  test('prompts appear once on first launch', async () => {
    const { stdout } = await runCLI(['copilot'], { cwd: testDir });
    
    const welcomeCount = countOccurrences(stdout, 'Describe a task to get started');
    const trustCount = countOccurrences(stdout, 'Confirm folder trust');
    
    expect(welcomeCount).toBe(1);
    expect(trustCount).toBe(1);
  });

  test('prompts appear once on subsequent launches', async () => {
    // First launch
    await runCLI(['copilot'], { cwd: testDir });
    
    // Second launch
    const { stdout } = await runCLI(['copilot'], { cwd: testDir });
    
    const welcomeCount = countOccurrences(stdout, 'Describe a task to get started');
    expect(welcomeCount).toBe(1);
  });
});
```

### Manual Testing Checklist

- [ ] Fresh directory - prompts appear once
- [ ] Subsequent run - prompts appear once (or not at all if already trusted)
- [ ] Different terminal (Terminal.app, iTerm2) - consistent behavior
- [ ] Clean config - prompts appear once
- [ ] With `--banner` flag - prompts appear once
- [ ] With plugins loaded - prompts appear once
- [ ] In ACP mode - appropriate behavior
- [ ] After `/clear` command - prompts can appear again once

---

## Rollback Plan

If the fix causes issues:

1. **Immediate**: Revert commit and release v0.0.403-hotfix
2. **Communication**: Notify users via changelog and GitHub issue
3. **Investigation**: Gather logs and error reports
4. **Iteration**: Fix and release v0.0.404 with improved solution

---

## Documentation Updates

Update the following documentation:

1. **Changelog**: Document the fix in v0.0.403 release notes
2. **Issue #1295**: Close with reference to fix version
3. **Plugin API Docs**: If initialization behavior changes
4. **Migration Guide**: If plugin authors need to update code

---

## Success Criteria

Fix is considered successful when:

1. ✅ Welcome prompt appears exactly once per launch
2. ✅ Folder trust prompt appears exactly once when needed
3. ✅ No regression in plugin functionality
4. ✅ No regression in ACP mode
5. ✅ Tests pass on macOS, Linux, Windows
6. ✅ Manual testing confirms fix
7. ✅ No new issues reported within 7 days of release

---

## Related Issues to Monitor

After fix deployment, monitor for:
- Plugin initialization issues
- ACP mode startup problems
- Session lifecycle hook problems
- Other duplicate UI elements

If these appear, may indicate fix was too aggressive or incomplete.
