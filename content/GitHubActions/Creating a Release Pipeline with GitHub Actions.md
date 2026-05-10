---
tags:
  - "#githubactions"
---
# Creating a Versioned Release Pipeline with GitHub Actions

## Overview

In this lab I extended an existing CI/CD pipeline to automatically publish a versioned GitHub Release — including a zipped artifact containing all application dependencies — every time new code is pushed. The goal was to give downstream teams a reliable, consistently named artifact they can pull without needing access to the build environment.

---

## Why This Matters

When multiple teams share a codebase, it's not enough to just build the code — you need a **predictable, versioned artifact** that anyone can download at any point. GitHub Releases solves this by attaching downloadable assets to a specific commit, tagged with a version number. Automating that process with GitHub Actions means every successful build produces a release without any manual steps.

---

## Environment Setup

The lab repository came with a `deploy-pipeline.yaml` workflow on the `lab` branch that already had two jobs: `lint` and `build`. I forked the repo into my own GitHub account so I could freely push changes, then enabled Actions in the forked repo.

```
Repository: content-github-actions-deep-dive-lesson (fork)
Branch: lab
Workflow file: .github/workflows/deploy-pipeline.yaml
```

---

## Step 1 — Adding the `publish` Job

The first goal was to add a `publish` job that creates a GitHub Release on every push. I appended the following to `deploy-pipeline.yaml` directly in the GitHub browser editor:

```yaml
publish:
  runs-on: ubuntu-latest
  steps:
    - name: Create release
      id: create_release
      uses: actions/create-release@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: ${{ github.run_number }}
        release_name: Release ${{ github.run_number }}
        body: New release for ${{ github.sha }}. Release notes can be found in the docs.
        draft: false
        prerelease: false
```

**Why `github.run_number`?** Each workflow run has an auto-incrementing integer. Using it as the tag name gives us a simple, sortable version number (e.g., `Release 12`) without having to manage version files manually.

**Why `github.sha`?** Embedding the commit SHA in the release body creates a direct traceability link — anyone reading the release notes can pinpoint the exact source code that was built.

---

## Step 2 — Troubleshooting Errors

The first run immediately surfaced two issues, both worth understanding because they reflect **real-world CI/CD gotchas**.

### Error 1 — YAML Indentation (`line 64`)

**Symptom:**

```
Invalid workflow file: .github/workflows/deploy-pipeline.yaml#L64
You have an error in your yaml syntax on line 64
```

**Cause:** The `Download artifact` step was pasted at the wrong indentation level, making it a sibling of the job block instead of a child of `steps`.

**Fix:** Corrected the indentation so the step was properly nested under `steps`. YAML is whitespace-sensitive, so even one extra space breaks parsing.

---

### Error 2 — Python Syntax in `lint` Job

**Symptom:**

```
./lambda_function.py:13:33: E999 SyntaxError: invalid syntax
Error: Process completed with exit code 1.
```

**Cause:** The source file `lambda_function.py` had an invalid `print` statement — a Python 2 style `print` without parentheses, which `flake8` correctly flagged under Python 3.

**Fix:** Updated the print statement to valid Python 3 syntax:

```python
# Before (Python 2 style — invalid in Python 3)
print "hello"

# After
print("hello")
```

---

### Error 3 — Deprecated Action Versions

**Symptom:**

```
This request has been automatically failed because it uses a deprecated version
of `actions/upload-artifact: v2`.
```

**Cause:** GitHub deprecated v1/v2 of several first-party actions. Referencing them causes an immediate failure.

**Fix:** Upgraded all action versions to their current major releases:

|Action|Old Version|Updated Version|
|---|---|---|
|`actions/download-artifact`|`@v2`|`@v4`|
|`actions/upload-release-asset`|`@v1`|Current|

---

### Error 4 — `GITHUB_TOKEN` Permission Denied

**Symptom:**

```
Error: Resource not accessible by integration
```

**Cause:** GitHub changed its default `GITHUB_TOKEN` permissions to **read-only** in newer repositories. The `publish` job was trying to create a release (a write operation) without the required permission scope.

**Fix:** Added an explicit `permissions` block to the `publish` job:

```yaml
publish:
  runs-on: ubuntu-latest
  permissions:
    contents: write   # Required to create releases and upload assets
```

This is a security-conscious default by GitHub — workflows should only request the minimum permissions they need, and `contents: write` explicitly grants the ability to create releases and attach assets.

---

## Step 3 — Attaching the Build Artifact to the Release

Once the release itself was being created successfully, the next step was to attach the compiled zip bundle as a downloadable asset. This required two additions to the `publish` job:

**1. Declare a dependency on the `build` job** so the artifact is always available before publishing:

```yaml
publish:
  runs-on: ubuntu-latest
  needs: build
  permissions:
    contents: write
```

**2. Download the artifact produced by `build`, then upload it to the release:**

```yaml
    - name: Download artifact
      uses: actions/download-artifact@v4
      with:
        name: zipped-bundle

    - name: Upload release asset
      uses: actions/upload-release-asset@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        upload_url: ${{ steps.create_release.outputs.upload_url }}
        asset_path: ./${{ github.sha }}.zip
        asset_name: source_code_with_libraries.zip
        asset_content_type: application/zip
```

`steps.create_release.outputs.upload_url` is the key here — the `create_release` step exposes a dynamic upload URL in its output, which the upload step consumes. This is a good example of **step output chaining** in GitHub Actions.

---

## Final Pipeline — All Jobs Green ✅

After all fixes were committed, the workflow ran cleanly end-to-end:

```
✅ lint
✅ build
✅ publish
```

The **Releases** section of the repository showed:

```
Releases  1
  ◇ Release 12  [Latest]   — 3 minutes ago
```

Clicking into the release revealed the attached `source_code_with_libraries.zip`, ready for any downstream consumer to download.

---

## Complete `publish` Job (Final State)

```yaml
publish:
  runs-on: ubuntu-latest
  needs: build
  permissions:
    contents: write
  steps:
    - name: Create release
      id: create_release
      uses: actions/create-release@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: ${{ github.run_number }}
        release_name: Release ${{ github.run_number }}
        body: New release for ${{ github.sha }}. Release notes can be found in the docs.
        draft: false
        prerelease: false

    - name: Download artifact
      uses: actions/download-artifact@v4
      with:
        name: zipped-bundle

    - name: Upload release asset
      uses: actions/upload-release-asset@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        upload_url: ${{ steps.create_release.outputs.upload_url }}
        asset_path: ./${{ github.sha }}.zip
        asset_name: source_code_with_libraries.zip
        asset_content_type: application/zip
```

---

## Key Takeaways

- **GitHub's default token permissions are read-only** — any job that writes to the repo (releases, packages, PRs) needs an explicit `permissions` block.
- **Action versions matter** — pinning deprecated versions silently fails. Keeping actions up to date is part of pipeline maintenance.
- **YAML indentation errors are silent killers** — a misaligned step doesn't warn you at write time; it only fails at runtime. Using a linter or the GitHub editor's inline validation helps catch these early.
- **Step output chaining** (`steps.<id>.outputs.<key>`) is a powerful pattern for passing data between steps without writing to disk or using environment files.
- **`needs` is required for artifact sharing** — GitHub Actions jobs run in parallel by default. Without `needs: build`, the `publish` job could start before the artifact exists.

---

_Lab source: [content-github-actions-deep-dive-lesson](https://github.com/hectorproko/content-github-actions-deep-dive-lesson)_