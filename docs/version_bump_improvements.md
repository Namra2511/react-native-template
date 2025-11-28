# Better Approaches for Version Bumping & Deployment Differentiation

## Current Issues

1. **Version Bumping:**
   - Race conditions when multiple PRs open simultaneously
   - Requires checking all open PRs (slow with many PRs)
   - No guarantee of sequential ordering in edge cases

2. **Deployment Differentiation:**
   - If versions collide, version codes collide
   - Android can't install multiple PR builds side-by-side
   - Only differentiated by Firebase metadata, not Android version code

---

## Recommended Approach: PR Number in Version Code

### Benefits
✅ **Guaranteed uniqueness** - No coordination needed  
✅ **No race conditions** - Each PR calculates independently  
✅ **Side-by-side installation** - Android treats each PR as different app version  
✅ **Simpler logic** - No need to check other PRs  
✅ **Works even if versions collide** - PR number ensures uniqueness

### Implementation

**1. Update Version Code Calculation:**

```ruby
# android/fastlane/scripts/firebase_distribution_service.rb
def build_apk(pr_number:)
  package_json = JSON.parse(File.read(package_json_path))
  version = package_json["version"]
  major, minor, patch = version.split('.').map(&:to_i)
  
  # Include PR number in version code for guaranteed uniqueness
  # Format: (major * 1000000) + (minor * 10000) + (patch * 100) + pr_number
  # This allows up to 99 PRs per version, 99 patches per minor, etc.
  version_code = (major * 1000000) + (minor * 10000) + (patch * 100) + pr_number.to_i
  
  # ... rest of build logic
end
```

**Example:**
- PR #2 with version 1.0.1 → version_code = 1000102
- PR #3 with version 1.0.1 → version_code = 1000103
- PR #4 with version 1.0.2 → version_code = 1000204

**2. Simplify Version Bump Workflow:**

Since PR number guarantees uniqueness, you can:
- Keep sequential version bumping (for main branch releases)
- OR use simpler approach: bump from main only (no cross-PR checking)

---

## Alternative Approach 1: PR-Based Version String

### Concept
Use PR number in version string for PR builds: `1.0.1-pr2`, `1.0.1-pr3`

### Benefits
- Clear identification in app
- No version code collision
- Simple to implement

### Drawbacks
- Non-standard version format
- Requires parsing in version code calculation

### Implementation

```ruby
# For PR builds, use: version-pr{PR_NUMBER}
version_string = "#{version}-pr#{pr_number}"
version_code = calculate_version_code(version, pr_number)
```

---

## Alternative Approach 2: Distributed Lock via GitHub API

### Concept
Use GitHub API to create/check a "lock" file or issue comment to coordinate version assignment.

### Benefits
- True sequential ordering
- No race conditions

### Drawbacks
- More complex
- Requires additional API calls
- Potential for deadlocks if workflow fails

### Implementation

```yaml
- name: Acquire version lock
  run: |
    # Create a GitHub issue comment or use a file in a special branch
    # to coordinate version assignment
    # Only one PR can acquire lock at a time
```

---

## Alternative Approach 3: Timestamp-Based Version Code

### Concept
Use timestamp or build number in version code instead of PR number.

### Benefits
- Always unique
- Chronological ordering

### Drawbacks
- Not human-readable
- Doesn't relate to PR number

### Implementation

```ruby
version_code = (major * 100000) + (minor * 1000) + patch + (Time.now.to_i % 10000)
```

---

## Comparison Matrix

| Approach | Uniqueness | Race Conditions | Simplicity | Side-by-side Install | Sequential Ordering |
|----------|-----------|-----------------|------------|---------------------|---------------------|
| **PR in Version Code** | ✅ Guaranteed | ✅ None | ✅ Simple | ✅ Yes | ⚠️ Not guaranteed |
| PR in Version String | ✅ Guaranteed | ✅ None | ✅ Simple | ✅ Yes | ⚠️ Not guaranteed |
| Distributed Lock | ✅ Guaranteed | ✅ None | ❌ Complex | ✅ Yes | ✅ Guaranteed |
| Timestamp-based | ✅ Guaranteed | ✅ None | ✅ Simple | ✅ Yes | ✅ Guaranteed |
| Current (check PRs) | ⚠️ Can collide | ⚠️ Possible | ⚠️ Medium | ❌ No | ⚠️ Mostly |

---

## Recommended Solution

**Use Approach 1: PR Number in Version Code**

### Why?
1. **Simplest** - No coordination needed
2. **Most reliable** - Guaranteed uniqueness
3. **Best UX** - Side-by-side installation works
4. **Maintainable** - Easy to understand and debug

### Implementation Steps

1. **Update `firebase_distribution_service.rb`:**
   ```ruby
   version_code = (major * 1000000) + (minor * 10000) + (patch * 100) + pr_number.to_i
   ```

2. **Simplify version bump workflow:**
   - Keep sequential bumping for main branch
   - Remove cross-PR checking (not needed anymore)
   - OR: Only bump from main baseline (simpler)

3. **Optional: Add version display:**
   ```ruby
   version_display = "#{version} (PR ##{pr_number})"
   ```

---

## Migration Path

1. **Phase 1:** Add PR number to version code (backward compatible)
2. **Phase 2:** Simplify version bump workflow (remove cross-PR checks)
3. **Phase 3:** Update documentation

---

## Additional Improvements

### 1. Version Display in App
Show PR number in app version for easier testing:
```typescript
// In app code
const version = `${packageVersion} (PR #${PR_NUMBER})`;
```

### 2. Build Variants
Use Android build variants for different PRs (more complex but cleaner separation).

### 3. Version Bump on Merge
Only bump version when PR merges to main (not when opened), ensuring true sequential ordering.

