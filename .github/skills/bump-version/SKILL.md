---
name: bump-version
description: This skill should be used when the user asks to "bump the version", "release a new version", "prepare a release", "update the version number", "version bump", or needs to determine and apply a semver version increment based on recent commits.
---

# Bump Version

Analyze commits since the last release tag, determine the appropriate semver version bump, and release the `DevProxy.Hosting` NuGet package.

## Workflow

Follow these steps in order. Do not skip the confirmation step.

### Step 1: Identify the Latest Tag

Run `git tag --sort=-v:refname | head -1` to find the most recent version tag. Tags follow the `v*` pattern (e.g., `v1.0.3`).

Read the current package version from the `<Version>` property in `DevProxy.Hosting/DevProxy.Hosting.csproj`. Verify that it matches the latest tag without the `v` prefix. If they differ, stop and ask the user which version is authoritative.

### Step 2: List Commits Since the Last Tag

Run `git log <latest-tag>..HEAD --oneline` to retrieve all unreleased commits. If there are no commits since the last tag, inform the user and stop.

### Step 3: Determine the Version Bump Type

Analyze each commit message to classify the version bump:

- **major** — Any commit indicates a breaking change. Look for:
  - `BREAKING CHANGE` or `BREAKING:` in the message
  - `!` after the type (e.g., `feat!:`, `fix!:`)
- **minor** — Any commit adds new functionality. Look for:
  - `feat:` or `feat(scope):` prefix
  - Commits describing new extension methods, configuration options, resource types, or capabilities
- **patch** — All other changes. Common indicators:
  - `fix:`, `deps:`, `chore:`, `docs:`, `refactor:`, `perf:`, `test:`, `ci:` prefixes
  - Dependency bumps, bug fixes, maintenance tasks

Apply the highest applicable level: if any commit is major, bump major. If any commit is minor (and none are major), bump minor. Otherwise, bump patch.

### Step 4: Confirm with the User

**STOP — Do not proceed without user confirmation.**

Present the analysis to the user:
1. List the commits since the last tag
2. State the recommended bump type and the reasoning
3. Show what the new version number will be (current → new)
4. Ask the user to confirm or choose a different bump type

### Step 5: Update the Project Version

Update the `<Version>` property in `DevProxy.Hosting/DevProxy.Hosting.csproj` to the confirmed version. Do not change dependency versions or other project metadata.

### Step 6: Validate the Package

Build and pack the project in Release configuration:

```bash
dotnet restore DevProxy.Hosting/DevProxy.Hosting.csproj
dotnet build DevProxy.Hosting/DevProxy.Hosting.csproj --configuration Release --no-restore
dotnet pack DevProxy.Hosting/DevProxy.Hosting.csproj --configuration Release --no-build --output DevProxy.Hosting/nupkg
```

Verify that `DevProxy.Hosting/nupkg/DevProxy.Hosting.X.Y.Z.nupkg` exists. If validation fails, do not commit or tag.

### Step 7: Commit and Tag the Release

Check `git status --short`. Do not stage unrelated user changes or generated package files. Stage only the project file, commit it, and tag that commit:

```bash
git add DevProxy.Hosting/DevProxy.Hosting.csproj
git commit -m "Bump version to X.Y.Z in project file"
git tag vX.Y.Z
```

Confirm that `git show vX.Y.Z:DevProxy.Hosting/DevProxy.Hosting.csproj` contains `<Version>X.Y.Z</Version>`.

### Step 8: Remind About Publishing

After bumping, remind the user to push the commit and tag to trigger the publish workflow:

```bash
git push && git push --tags
```

The `.github/workflows/publish.yml` workflow reads the package version from the `v*` tag and automatically publishes `DevProxy.Hosting` to NuGet.
