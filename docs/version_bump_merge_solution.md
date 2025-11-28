# Complete Version Bump & Merge Solution

## Overview

This solution ensures:
1. **Sequential versioning** on main branch (1.0.1 → 1.0.2 → 1.0.3)
2. **Unique builds** for each PR (even if versions collide)
3. **Merge-time correction** when PRs merge out of order
4. **Side-by-side installation** of multiple PR builds

---

## Architecture

### 1. PR Version Bumping (On PR Open/Update)

**Workflow:** `.github/workflows/cd.yml` & `.github/workflows/ci.yml`

**What it does:**
- Checks all open PRs and main branch
- Finds the highest version currently assigned
- Bumps patch version for the current PR
- Ensures sequential versioning across all open PRs

**Example:**
- Main: `1.0.1`
- PR #2 opens → bumps to `1.0.2`
- PR #3 opens → bumps to `1.0.3`
- PR #4 opens → bumps to `1.0.4`

### 2. Unique Build Identification (PR Number in Version Code)

**File:** `android/fastlane/scripts/firebase_distribution_service.rb`

**What it does:**
- Includes PR number in Android version code calculation
- Formula: `(major * 1000000) + (minor * 10000) + (patch * 100) + pr_number`
- Guarantees unique version codes even if versions collide

**Example:**
- PR #2 with version `1.0.1` → version_code `1000102`
- PR #3 with version `1.0.1` → version_code `1000103`
- PR #4 with version `1.0.2` → version_code `1000204`

**Benefits:**
- ✅ No race conditions
- ✅ Side-by-side installation works
- ✅ Each PR build is uniquely identifiable
- ✅ No coordination needed between PRs

### 3. Merge-Time Version Correction (On Merge to Main)

**Workflow:** `.github/workflows/production.yml`

**What it does:**
- Runs when PR merges to main
- Compares merged version with previous main version
- If version is not sequential, automatically corrects it
- Commits the correction back to main

**Example Scenario:**
- Main: `1.0.1`
- PR #4 (1.0.4) merges before PR #2 (1.0.2)
- Merge-time correction: Detects `1.0.4` is not sequential
- Corrects to `1.0.2` (next sequential after `1.0.1`)
- Commits: `chore: correct version to 1.0.2 to maintain sequential ordering [auto-version-fix]`

**Benefits:**
- ✅ Main branch always has sequential versions
- ✅ Handles out-of-order merges gracefully
- ✅ No manual intervention needed
- ✅ Clean version history

---

## Flow Diagrams

### PR Open Flow
```
PR #2 Opens
    ↓
Check main version (1.0.1)
    ↓
Check all open PRs versions
    ↓
Find max version (1.0.1)
    ↓
Bump to 1.0.2
    ↓
Commit & Push to PR branch
```

### Merge Flow
```
PR #4 Merges to Main
    ↓
Main has version 1.0.4
Previous main was 1.0.1
    ↓
Version Correction Job Runs
    ↓
Detects: 1.0.4 is not sequential
    ↓
Corrects to 1.0.2 (next after 1.0.1)
    ↓
Commits correction to main
    ↓
Production deployment continues
```

### Build Flow
```
PR #2 Build Requested
    ↓
Read package.json version (1.0.2)
    ↓
Calculate version code:
  (1 * 1000000) + (0 * 10000) + (2 * 100) + 2
  = 1000202
    ↓
Build APK with unique version code
    ↓
Deploy to Firebase
```

---

## Key Features

### 1. Guaranteed Uniqueness
- **PR number in version code** ensures every build is unique
- No collisions even if multiple PRs have same version
- Works independently without coordination

### 2. Sequential Main Branch
- **Merge-time correction** ensures main always has sequential versions
- Handles out-of-order merges automatically
- Clean version history for production releases

### 3. Cross-PR Coordination
- **Version bump workflow** checks all open PRs
- Ensures new PRs get next sequential version
- Prevents version conflicts before they happen

### 4. Side-by-Side Installation
- **Unique version codes** allow Android to install multiple PR builds
- Each PR build is treated as different app version
- QA can test multiple PRs simultaneously

---

## Edge Cases Handled

### Case 1: PRs Open Simultaneously
**Problem:** Two PRs open at same time, both see main at 1.0.1

**Solution:**
- Both PRs check open PRs
- First to complete gets 1.0.2
- Second sees 1.0.2, gets 1.0.3
- PR number in version code ensures uniqueness even if both get 1.0.2

### Case 2: PR Merges Out of Order
**Problem:** PR #4 (1.0.4) merges before PR #2 (1.0.2)

**Solution:**
- Merge-time correction detects non-sequential version
- Automatically corrects to 1.0.2
- When PR #2 merges later, it will be corrected to 1.0.3

### Case 3: Version Collision (Edge Case)
**Problem:** Multiple PRs end up with same version (1.0.3)

**When can this happen?**
Even with all safeguards, version collision can occur in these rare edge cases:

1. **Race Condition Window:**
   ```
   Time T0: PR #2 opens, workflow starts checking versions
   Time T1: PR #3 opens (before PR #2's version is pushed)
   Time T2: Both workflows see main at 1.0.1, no other PRs yet
   Time T3: Both calculate next version as 1.0.2
   Time T4: Both push 1.0.2 to their branches
   Result: Both PRs have version 1.0.2
   ```

2. **Git Fetch Timing:**
   ```
   PR #2 workflow: Fetches branches, sees PR #3 branch doesn't exist yet
   PR #3 opens: Creates branch
   PR #2 workflow: Already calculated version, doesn't re-check
   Result: Both get same version
   ```

3. **API/Git Failures:**
   ```
   PR #2 workflow: GitHub API fails to fetch open PRs
   Falls back to main version only
   PR #3 workflow: Also fails, falls back to main version
   Result: Both calculate from same baseline
   ```

4. **Workflow Skipped:**
   ```
   PR #2: Version bump workflow skipped/failed
   PR #3: Opens, sees PR #2 has no version bump yet
   Both end up with same version
   ```

**Solution (Why it's not a problem):**
- ✅ **PR number in version code** ensures unique builds regardless
  - PR #2 with 1.0.2 → version_code `1000202`
  - PR #3 with 1.0.2 → version_code `1000203`
- ✅ **Side-by-side installation works** - Different version codes
- ✅ **Main branch gets corrected** - Merge-time correction ensures sequential
- ✅ **No build conflicts** - Each PR build is uniquely identifiable

**Note:** This scenario is rare and demonstrates the system's resilience. Even if versions collide, the PR number in version code ensures everything still works correctly.

### Case 4: PR Closed Without Merge
**Problem:** PR with version 1.0.3 is closed, never merged

**Solution:**
- Version is "lost" but doesn't matter
- Next PR will use next available version
- Main branch versioning remains sequential

---

## Configuration

### Version Code Formula
```ruby
version_code = (major * 1000000) + (minor * 10000) + (patch * 100) + pr_number
```

**Limits:**
- Major: 0-99
- Minor: 0-99
- Patch: 0-99
- PR Number: 0-99

**Example Limits:**
- Version `99.99.99` with PR #99 → version_code `99999999` (within int32 limit)

### Workflow Triggers

**PR Workflows (cd.yml, ci.yml):**
- `opened` - New PR created
- `synchronize` - PR updated with new commits
- `reopened` - PR reopened after being closed
- `ready_for_review` - PR marked ready for review

**Production Workflow (production.yml):**
- `push` to `main` - PR merged to main

---

## Testing

### Test Scenario 1: Sequential Merges
1. Open PR #2 → should get 1.0.2
2. Open PR #3 → should get 1.0.3
3. Merge PR #2 → main should have 1.0.2
4. Merge PR #3 → main should have 1.0.3

### Test Scenario 2: Out-of-Order Merges
1. Open PR #2 → should get 1.0.2
2. Open PR #3 → should get 1.0.3
3. Open PR #4 → should get 1.0.4
4. Merge PR #4 first → main should be corrected to 1.0.2
5. Merge PR #2 → main should be corrected to 1.0.3
6. Merge PR #3 → main should be corrected to 1.0.4

### Test Scenario 3: Simultaneous Opens (Normal Case)
1. Open PR #2 and PR #3 simultaneously
2. Both workflows check open PRs
3. First to complete gets 1.0.2
4. Second sees 1.0.2, gets 1.0.3
5. Both should have unique version codes

**Expected Result:** Sequential versions (1.0.2, 1.0.3)

### Test Scenario 3b: Version Collision (Edge Case)
**To test collision scenario:**
1. Open PR #2
2. Immediately open PR #3 (within 1-2 seconds)
3. Both workflows may see same state
4. Both might get 1.0.2

**Expected Result:** 
- Both have version 1.0.2 (collision)
- But version codes are unique:
  - PR #2: version_code `1000202`
  - PR #3: version_code `1000203`
- Both can be installed side-by-side ✅
- On merge, main gets corrected to sequential ✅

---

## Monitoring

### Check Version History
```bash
git log --oneline --all --grep="auto-bump\|auto-version-fix"
```

### Check Current Versions
```bash
# Main branch version
git show main:package.json | jq .version

# PR branch version
git show origin/pr-branch:package.json | jq .version
```

### Check Version Codes
- Check Firebase App Distribution releases
- Each release shows version code in metadata
- Verify PR numbers match version codes

---

## Troubleshooting

### Issue: Version not sequential on main
**Cause:** Merge-time correction didn't run or failed

**Fix:** Manually correct version:
```bash
# Get previous version
PREV=$(git show HEAD~1:package.json | jq -r .version)

# Calculate next version
# Increment patch version

# Update package.json
# Commit and push
```

### Issue: PR has wrong version
**Cause:** Version bump workflow didn't run or failed

**Fix:** Re-run workflow or manually bump:
```bash
# Check current version
jq .version package.json

# Bump patch version
# Commit and push
```

### Issue: Version codes collide
**Cause:** PR number > 99 or version > 99.99.99

**Fix:** Adjust version code formula or use different versioning scheme

---

## Future Improvements

1. **Version Bump on Merge Only**
   - Only bump version when PR merges
   - Simpler workflow
   - True sequential ordering

2. **Version Display in App**
   - Show PR number in app version string
   - Easier testing and identification

3. **Build Variants**
   - Use Android build variants for different PRs
   - Cleaner separation
   - More complex setup

4. **Version History Tracking**
   - Track version assignments in a file
   - Better visibility
   - Easier debugging

