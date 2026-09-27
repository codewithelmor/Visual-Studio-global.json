# global.json Reference Guide

`global.json` is a configuration file used by the .NET SDK to control which version of the .NET SDK is used when building, running, or restoring a project. It is read by the `dotnet` CLI and by Visual Studio (and any IDE that shells out to the .NET CLI, such as VS Code, Rider, etc.).

---

## Table of Contents

1. [How It Works](#how-it-works)
2. [File Location](#file-location)
3. [Full Schema](#full-schema)
4. [Root-Level Properties](#root-level-properties)
5. [`sdk` Object Properties](#sdk-object-properties)
   - [`version`](#version)
   - [`rollForward`](#rollforward)
   - [`allowPrerelease`](#allowprerelease)
   - [`paths`](#paths)
   - [`errorMessage`](#errormessage)
6. [`rollForward` Values Explained in Detail](#rollforward-values-explained-in-detail)
7. [`msbuild-sdks` Object](#msbuild-sdks-object)
8. [Creating a global.json File](#creating-a-globaljson-file)
9. [Common Scenarios & Examples](#common-scenarios--examples)
10. [How Visual Studio Uses It](#how-visual-studio-uses-it)
11. [Troubleshooting](#troubleshooting)
12. [Best Practices](#best-practices)

---

## How It Works

When you run a `dotnet` command (`build`, `run`, `restore`, `test`, `publish`, etc.) or open a solution in Visual Studio, the .NET SDK resolver searches for a `global.json` file:

1. Starting in the **current working directory**.
2. Walking **upward** through parent directories until one is found, or the root of the drive/filesystem is reached.

If a `global.json` is found, its `sdk` settings determine which installed SDK version is used for that build context. If none is found, the **latest installed SDK** is used by default.

> **Note:** `global.json` pins the **SDK/tooling version** — it does NOT change the **target framework** (`net8.0`, `net6.0`, etc.) specified in your `.csproj`/`.fsproj`/`.vbproj` files. Those are independent settings. You can build a `net6.0` project using SDK 8.x tooling, for example.

---

## File Location

- Typically placed at the **root of a repository or solution**, alongside the `.sln` file.
- Only **one** `global.json` should apply per build context — if you have multiple projects with different SDK needs, place separate `global.json` files in their respective subfolders (the resolver uses the *closest* one found walking up from the project).
- File name is case-sensitive on Linux/macOS: must be exactly `global.json`.

---

## Full Schema

```json
{
  "sdk": {
    "version": "8.0.401",
    "rollForward": "latestFeature",
    "allowPrerelease": false,
    "paths": [ "./sdk", "global" ],
    "errorMessage": "Custom message shown if no matching SDK is found."
  },
  "msbuild-sdks": {
    "My.Custom.Sdk": "1.2.3",
    "Another.Sdk": "4.5.6"
  }
}
```

---

## Root-Level Properties

| Property | Type | Description |
|---|---|---|
| `sdk` | object | Controls which .NET SDK version is selected for the build. |
| `msbuild-sdks` | object | Pins versions of MSBuild project SDKs (used in SDK-style project imports, unrelated to the .NET CLI SDK version). |

---

## `sdk` Object Properties

### `version`

- **Type:** `string`
- **Format:** `MAJOR.MINOR.PATCH` (e.g., `"8.0.401"`), can also be a feature-band version like `"8.0.400"`.
- **Purpose:** Specifies the exact (or baseline) SDK version to use.
- **Behavior:** Combined with `rollForward` to determine what happens if that exact version isn't installed on the machine.

```json
{ "sdk": { "version": "8.0.401" } }
```

If omitted entirely, the SDK resolver just uses the latest installed SDK (rollForward rules mostly become moot without a version to roll forward *from*).

---

### `rollForward`

- **Type:** `string`
- **Default:** `"latestPatch"` (if `version` is specified but `rollForward` is not)
- **Purpose:** Defines the fallback strategy when the exact `version` specified isn't installed.
- **Allowed values:** `patch`, `feature`, `minor`, `major`, `latestPatch`, `latestFeature`, `latestMinor`, `latestMajor`, `disable`

See the [detailed breakdown](#rollforward-values-explained-in-detail) below — each value behaves quite differently.

---

### `allowPrerelease`

- **Type:** `boolean`
- **Default:** `true` if not specified (SDK resolver may pick preview SDKs) — but effectively `false` for many roll-forward policies depending on installed SDKs. To be explicit and avoid ambiguity, set this yourself.
- **Purpose:** Controls whether preview/RC (release candidate) SDKs can be selected when resolving a version.

```json
{ "sdk": { "version": "9.0.100", "allowPrerelease": false } }
```

Set to `false` in production/CI pipelines to guarantee only stable, released SDKs are used.

---

### `paths`

- **Type:** `array of strings`
- **Purpose:** Specifies additional locations to search for the SDK, beyond the default global install locations.
- **Special value:** `"global"` tells the resolver to also check the machine-wide/global SDK install location.

```json
{
  "sdk": {
    "version": "8.0.401",
    "paths": [ "./my-local-sdks", "global" ]
  }
}
```

Useful for xcopy-deployed SDKs or CI agents with SDKs installed in non-standard locations.

---

### `errorMessage`

- **Type:** `string`
- **Purpose:** A custom error message displayed to the developer if no matching SDK can be resolved (instead of the generic .NET error).

```json
{
  "sdk": {
    "version": "8.0.401",
    "rollForward": "disable",
    "errorMessage": "Please install .NET SDK 8.0.401 exactly. Download: https://dotnet.microsoft.com/download"
  }
}
```

---

## `rollForward` Values Explained in Detail

| Value | Behavior |
|---|---|
| `patch` | Uses the specified major.minor.feature band, rolls forward to the **latest patch** only if the exact version is missing. Will NOT roll to a different feature band. This is the **narrowest** option. |
| `feature` | Rolls forward to the **latest patch within the same feature band** if an exact match isn't found (e.g., `8.0.4xx` → any `8.0.4xx` patch). Will not roll to a higher feature band. |
| `minor` | If no matching feature band is found within the specified minor version, rolls forward to the **lowest available feature band within that same minor version**. |
| `major` | If no compatible minor version is found, rolls forward to the **lowest available minor version within a higher major version**. Broadest "upgrade" behavior while still trying to stay low. |
| `latestPatch` | Uses the latest installed **patch** within the specified major.minor.feature band. This is the **default** behavior when `rollForward` is omitted. |
| `latestFeature` | Uses the latest installed feature band and patch within the specified major.minor version. |
| `latestMinor` | Uses the latest installed minor version and feature band within the specified major version. |
| `latestMajor` | Uses the absolute latest SDK installed on the machine, regardless of major version. Most permissive option. |
| `disable` | **No roll-forward at all.** Requires the *exact* `version` specified to be installed, or the build fails. Most restrictive — best for strict reproducibility. |

### Visual Comparison (assume `version: "8.0.401"`)

| rollForward | Will accept 8.0.402? | Will accept 8.0.4xx→8.0.5xx? | Will accept 8.1.x? | Will accept 9.x? |
|---|---|---|---|---|
| `disable` | ❌ (must be exact 8.0.401) | ❌ | ❌ | ❌ |
| `patch` | ✅ | ❌ | ❌ | ❌ |
| `feature` | ✅ | ❌ (won't jump band) | ❌ | ❌ |
| `latestPatch` | ✅ (latest patch in band) | ❌ | ❌ | ❌ |
| `latestFeature` | ✅ | ✅ | ❌ | ❌ |
| `minor` | ✅ | ✅ (if needed) | ✅ (lowest available) | ❌ |
| `latestMinor` | ✅ | ✅ | ✅ (latest) | ❌ |
| `major` | ✅ | ✅ | ✅ | ✅ (lowest available) |
| `latestMajor` | ✅ | ✅ | ✅ | ✅ (latest available) |

---

## `msbuild-sdks` Object

This is unrelated to the .NET CLI SDK version — it pins versions of **MSBuild project SDKs**, which are referenced in `.csproj` files via:

```xml
<Project Sdk="My.Custom.Sdk">
```

Instead of hardcoding the version in every `.csproj`, you can centralize it:

```json
{
  "msbuild-sdks": {
    "My.Custom.Sdk": "1.2.3",
    "Microsoft.Build.Traversal": "3.4.0"
  }
}
```

Then in your project file, omit the version:
```xml
<Project Sdk="My.Custom.Sdk">
```

---

## Creating a global.json File

### Via CLI
```bash
dotnet new globaljson --sdk-version 8.0.401
```

Optional flags:
```bash
dotnet new globaljson --sdk-version 8.0.401 --roll-forward latestFeature
```

### Check what SDKs are installed first
```bash
dotnet --list-sdks
```

Example output:
```
6.0.428 [C:\Program Files\dotnet\sdk]
8.0.401 [C:\Program Files\dotnet\sdk]
9.0.100 [C:\Program Files\dotnet\sdk]
```

### Manually
Simply create a `global.json` file at your solution root with the desired JSON content (see [Full Schema](#full-schema)).

---

## Common Scenarios & Examples

### 1. Lock CI/CD builds to an exact SDK (strict reproducibility)
```json
{
  "sdk": {
    "version": "8.0.401",
    "rollForward": "disable"
  }
}
```

### 2. Allow patch updates but nothing else (recommended default for most teams)
```json
{
  "sdk": {
    "version": "8.0.401",
    "rollForward": "latestPatch"
  }
}
```

### 3. Stay within .NET 8 but accept any installed 8.x SDK
```json
{
  "sdk": {
    "version": "8.0.100",
    "rollForward": "latestMinor"
  }
}
```

### 4. Allow preview SDKs for testing an upcoming .NET release
```json
{
  "sdk": {
    "version": "9.0.100-rc.1",
    "rollForward": "latestFeature",
    "allowPrerelease": true
  }
}
```

### 5. Legacy project pinned to an older SDK, isolated from newer projects on the same machine
```json
{
  "sdk": {
    "version": "6.0.428",
    "rollForward": "latestPatch"
  }
}
```
Place this `global.json` only in the legacy project's folder; other projects elsewhere on the machine continue using whatever SDK they resolve to.

---

## How Visual Studio Uses It

- When you open a `.sln` or folder in Visual Studio, it invokes the same SDK resolution logic as the `dotnet` CLI.
- If the pinned SDK version (per `rollForward` rules) **isn't installed**, Visual Studio will show a load error / the project may fail to load, and prompt you that the required SDK is missing.
- The **Visual Studio installer's bundled SDK** must satisfy the `global.json` constraints, or you need to install the required SDK separately (Visual Studio's own SDK is not always the newest available from dotnet.microsoft.com).
- IntelliSense, NuGet restore, and build all respect the resolved SDK version — mismatches are a common cause of "works on my machine but not in VS" issues.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| "A compatible .NET SDK was not found" | `rollForward` too strict (e.g., `disable`) and exact version not installed | Install the exact SDK version, or relax `rollForward` |
| Different behavior in VS vs. CLI | VS using a different/bundled SDK than the CLI's PATH | Check **Tools > Options > Projects and Solutions > .NET Core** in VS, and compare with `dotnet --version` from a terminal in the repo folder |
| global.json ignored | File not at a directory the resolver walks through (e.g., placed in a subfolder below the project) | Move it to the repo/solution root, or to a common ancestor folder |
| Wrong SDK picked on build server | CI agent has multiple SDKs installed, no global.json committed | Commit a `global.json` with explicit `version` and `rollForward` |

---

## Best Practices

1. **Commit `global.json` to source control** — ensures every developer and CI agent builds with a consistent SDK.
2. **Use `latestPatch` or `latestFeature`** for most day-to-day development — gets security/bug fixes automatically without breaking changes.
3. **Use `disable` only when strict reproducibility is critical** (e.g., regulated environments, exact build reproducibility requirements) — it's brittle and requires manual updates whenever the SDK changes.
4. **Set `allowPrerelease: false` explicitly** in any `global.json` committed to a production/release branch.
5. **Don't over-scope** — only add a `global.json` where you actually need to pin a version; unnecessary files add maintenance overhead across a large repo with many projects.
6. **Keep it in sync with your CI pipeline's installed SDK** to avoid pipeline failures after a `dotnet` SDK version change on the build agent.

---

## References

- [Microsoft Docs: .NET SDK overview](https://learn.microsoft.com/dotnet/core/tools/global-json)
- [Microsoft Docs: `dotnet new globaljson`](https://learn.microsoft.com/dotnet/core/tools/dotnet-new-sdk-templates#globaljson-file)
