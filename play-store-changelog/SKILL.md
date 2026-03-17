# Play Store Changelog

## Purpose

Generate user-facing Google Play Store release notes from git history.

Automatically collects commits and changed files since the last git tag (or a specified ref) and transforms them into clear, benefit-focused release notes suitable for the Play Store "What's new" section. Filters out internal-only changes and groups user-visible improvements by theme.

## Use When

- You need to write Play Store "What's new" text before a release
- You want to generate release notes from git history without reading every commit manually
- You need to produce notes in multiple languages (with a base English version to translate from)

---

## Step 1 — Collect Changes Since Last Release

Run these commands to gather raw material:

```bash
# Find the last release tag
git describe --tags --abbrev=0

# List all commits since that tag
git log <last-tag>..HEAD --oneline

# List all files changed since that tag
git diff --name-only <last-tag>..HEAD

# Get full commit messages for context
git log <last-tag>..HEAD --pretty=format:"%h %s%n%b"
```

If no tag exists yet, compare against the previous release branch or a known commit SHA:

```bash
git log <commit-sha>..HEAD --oneline
```

---

## Step 2 — Classify Each Change

Sort every commit into one of these buckets:

| Bucket | Include in notes? | Examples |
|--------|-------------------|---------|
| **User-visible feature** | ✅ Yes | New screen, new setting, new content type |
| **User-visible fix** | ✅ Yes | Crash fix, UI glitch, wrong data displayed |
| **Performance improvement** | ✅ Yes (if noticeable) | Faster load, smoother scroll |
| **Internal / refactor** | ❌ No | Dependency updates, code cleanup, test changes |
| **CI / build / tooling** | ❌ No | Gradle changes, lint fixes, version bumps |
| **Backend / API only** | ❌ No (unless user-facing) | Endpoint changes with no UI impact |

### Signals that a change is user-visible

- Modified files under `ui/`, `feature/`, `res/`, `navigation/`
- Commit message contains: `feat`, `fix`, `improve`, `add`, `update`, `crash`, `perf`
- Changes to `strings.xml`, drawable resources, or navigation graphs

### Signals that a change is internal

- Modified files under `data/`, `domain/`, `di/`, `test/`, `buildSrc/`
- Commit message contains: `refactor`, `chore`, `ci`, `build`, `test`, `cleanup`, `bump`

---

## Step 3 — Write the Release Notes

### Format Rules

- **Max 500 characters** (Play Store hard limit per language)
- Write in **plain language** — no technical terms, no class names, no Jira tickets
- Lead with the most impactful change
- Use short bullet points or a brief paragraph — both are acceptable
- Focus on **user benefit**, not implementation detail
- Past tense for fixes ("Fixed a crash…"), present tense for features ("You can now…")

### Templates

**Feature-heavy release:**
```
What's new in v[X.Y]:

• You can now [feature 1]
• [Feature 2] is here — [one-line benefit]
• Improved [area] for a smoother experience

Bug fixes and performance improvements.
```

**Fix-heavy release:**
```
v[X.Y] brings stability and polish:

• Fixed a crash when [action]
• [Screen] now loads faster
• Resolved an issue where [thing] would [problem]
```

**Minor release:**
```
Bug fixes and performance improvements.
```

### Example Transformation

Raw commits:
```
a1b2c3 feat: add bookmarks screen
d4e5f6 fix: crash on empty feed state  
g7h8i9 refactor: extract PostRepository interface
j0k1l2 chore: bump compileSdk to 35
m3n4o5 fix: profile avatar not loading on slow connections
p6q7r8 feat: swipe to dismiss notifications
```

Resulting notes:
```
What's new:

• Bookmarks — save posts to read later
• Swipe to dismiss notifications
• Fixed a crash on the home feed
• Profile images now load reliably on slow connections
```

---

## Step 4 — Validate

Before finalising, confirm:

1. Every bullet point maps back to a real commit — no invented features
2. Total character count is under 500
3. No internal ticket numbers, branch names, or technical jargon
4. The most important change to users appears first
5. Tone matches your app's voice (friendly, professional, etc.)

```bash
# Quick character count check (macOS/Linux)
echo -n "Your release notes here" | wc -c
```

---

## Multi-language Notes

Play Store supports release notes in every language your app is localised for. Workflow:

1. Write the English (`en-GB` or `en-US`) version first following this skill
2. Use a translation tool or service for other locales
3. Store notes under `fastlane/metadata/android/<locale>/changelogs/<version-code>.txt` if using Fastlane
4. Keep the same bullet structure across languages — don't add or remove points per locale

### Fastlane directory structure
```
fastlane/
  metadata/
    android/
      en-GB/
        changelogs/
          42.txt      ← version code matches your versionCode
      fr-FR/
        changelogs/
          42.txt
```

---

## Checklist

- [ ] All commits since last tag collected
- [ ] Internal / tooling commits filtered out
- [ ] Notes are 500 characters or fewer
- [ ] Every bullet maps to a real commit
- [ ] No technical jargon or internal references
- [ ] Most impactful change listed first
- [ ] Notes stored in version control (e.g. Fastlane metadata)
