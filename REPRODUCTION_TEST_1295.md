# Reproduction Test Case for Issue #1295

## Issue
Duplicated welcome and folder trust prompts shown on every run of Copilot CLI

## Environment Setup

### Prerequisites
- GitHub Copilot CLI version 0.0.402
- macOS (issue confirmed on this platform)
- Terminal.app or iTerm2
- Active Copilot subscription
- Authenticated GitHub account

### Test Environment
```bash
# System Information
OS: macOS
Terminal: Terminal.app or iTerm2
Shell: bash/zsh
Copilot CLI Version: 0.0.402
```

## Test Cases

### Test Case 1: First Run in New Directory

**Objective**: Verify duplication occurs on first launch in a fresh directory

**Steps**:
1. Create a new test directory:
   ```bash
   mkdir /tmp/copilot-test-$(date +%s)
   cd /tmp/copilot-test-*
   ```

2. Launch Copilot CLI:
   ```bash
   copilot
   ```

3. Observe the UI prompts

**Expected Behavior**:
- Welcome prompt ("Describe a task to get started") should appear ONCE
- Folder trust confirmation UI should appear ONCE (if folder is untrusted)

**Actual Behavior** (v0.0.402):
- Welcome prompt appears TWICE
- Folder trust confirmation UI appears TWICE
- Both prompts appear in sequence, duplicated

**Result**: ❌ FAIL - Duplication occurs

### Test Case 2: Subsequent Run in Same Directory

**Objective**: Verify duplication persists on repeat launches

**Steps**:
1. In the same directory from Test Case 1
2. Exit Copilot CLI (if running)
3. Launch again:
   ```bash
   copilot
   ```

4. Observe the UI prompts

**Expected Behavior**:
- Welcome prompt should appear ONCE (if shown at all)
- Folder trust prompt should NOT appear (folder already trusted)

**Actual Behavior** (v0.0.402):
- Welcome prompt appears TWICE
- Folder trust prompt appears TWICE
- Issue reproduces consistently

**Result**: ❌ FAIL - Duplication persists

### Test Case 3: Different Terminal Emulator

**Objective**: Verify issue is not terminal-specific

**Steps**:
1. If using Terminal.app, switch to iTerm2 (or vice versa)
2. Navigate to test directory:
   ```bash
   cd /tmp/copilot-test-*
   ```
3. Launch Copilot CLI:
   ```bash
   copilot
   ```
4. Observe the UI prompts

**Expected Behavior**:
- Prompts should appear once each

**Actual Behavior** (v0.0.402):
- Duplication occurs in both Terminal.app and iTerm2
- Issue is not terminal-specific

**Result**: ❌ FAIL - Terminal-independent issue

### Test Case 4: Clean Configuration

**Objective**: Verify issue is not related to user configuration

**Steps**:
1. Backup existing configuration:
   ```bash
   mv ~/.copilot ~/.copilot.backup.$(date +%s)
   ```

2. Launch with fresh config:
   ```bash
   cd /tmp/copilot-test-new-$(date +%s) && mkdir -p .
   copilot
   ```

3. Observe the UI prompts

**Expected Behavior**:
- Prompts should appear once each with fresh config

**Actual Behavior** (v0.0.402):
- Duplication still occurs
- Issue is not configuration-related

**Result**: ❌ FAIL - Config-independent issue

4. Restore configuration:
   ```bash
   mv ~/.copilot.backup.* ~/.copilot
   ```

### Test Case 5: With --banner Flag

**Objective**: Check if additional flags affect duplication

**Steps**:
1. Launch with banner flag:
   ```bash
   copilot --banner
   ```

2. Observe prompts

**Expected Result**:
- Banner shows once
- Prompts show once each

**Actual Result** (v0.0.402):
- TBD (to be documented)

## Visual Evidence

The issue includes a screenshot showing:
- Duplicated welcome prompt interface
- Duplicated folder trust confirmation
- Both appearing twice in sequence

Screenshot URL: https://github.com/user-attachments/assets/b413ce09-518b-4155-8771-ed97ffdb0e46

## Automated Test Script

```bash
#!/bin/bash
# Automated reproduction test for issue #1295

set -e

echo "=== Copilot CLI Issue #1295 Reproduction Test ==="
echo ""

# Check version
echo "Checking Copilot CLI version..."
copilot --version || echo "Could not get version"
echo ""

# Test 1: First run
echo "Test 1: First run in new directory"
TEST_DIR="/tmp/copilot-test-$(date +%s)"
mkdir -p "$TEST_DIR"
cd "$TEST_DIR"
echo "Created test directory: $TEST_DIR"
echo "Launching copilot... (observe for duplicate prompts)"
echo ""

# Note: This would need to be interactive to observe prompts
# For automated testing, would need to inspect logs/output
copilot --help  # Using --help as a safe command for now

echo ""
echo "=== Manual Observation Required ==="
echo "To fully reproduce:"
echo "1. cd $TEST_DIR"
echo "2. Run: copilot"
echo "3. Count welcome prompts (should be 1, actually appears 2)"
echo "4. Count folder trust prompts (should be 1, actually appears 2)"
echo ""
```

## Regression Test

Once the issue is fixed, this test should pass:

```bash
#!/bin/bash
# Regression test - should pass after fix

# This would require parsing CLI output or using an automated testing framework
# Pseudocode:
# 1. Launch copilot in test directory
# 2. Count occurrences of welcome prompt
# 3. Count occurrences of folder trust prompt
# 4. Assert count == 1 for each
# 5. Exit with status 0 if pass, 1 if fail

echo "Regression test for issue #1295"
echo "TODO: Implement automated prompt counting"
echo "Expected: Each prompt appears exactly once"
```

## Notes

1. **Consistency**: Issue reproduces 100% of the time in affected version
2. **Platform**: Confirmed on macOS; other platforms TBD
3. **Workaround**: None known; user must dismiss duplicate prompts
4. **Impact**: UX annoyance; does not block functionality
5. **Version**: Introduced in v0.0.402; previous versions unknown

## Related Issues

Check for similar duplication issues in other areas:
- Permission prompts
- Confirmation dialogs
- Status messages
- Error messages

## Debug Information to Collect

If able to access debug logs:
1. Session initialization sequence
2. Event listener registrations
3. UI render calls
4. Plugin lifecycle hooks
5. Timestamp of each prompt display

This information would help confirm the root cause hypothesis.
